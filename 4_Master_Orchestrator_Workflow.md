# Master Orchestrator Workflow

## Before Building — Your Confirmed Node Specs

```
Based on the research output:

Schedule Trigger    → typeVersion 1.3
Code Node (Python)  → typeVersion 2
  Data access       → _items[0]["json"]["field"]
  No dot notation   → _item.json.field WRONG
IF Node             → typeVersion 2.2
  New condition     → conditions.options.version required
HTTP Request        → typeVersion 4.2
Execute Command     → typeVersion 1 (must enable first)
No Operation        → typeVersion 1

CRITICAL BEFORE IMPORTING:
Add this to your n8n environment variables:
  NODES_EXCLUDE=[]
  N8N_BLOCK_ENV_ACCESS_IN_NODE=false

Without these two lines, Execute Command node
will show as UNKNOWN and Python cannot read
environment variables.
```

---

## One Critical Question First

```
In your n8n setup, where does scraper.py live
and what path runs Python?

Common scenarios:

SCENARIO A: n8n runs in Docker
  Execute Command runs INSIDE the container
  scraper.py must be INSIDE the container
  Python path: /usr/bin/python3 or python3

SCENARIO B: n8n runs directly on server (npm/node)
  Execute Command runs on the actual server
  scraper.py can be anywhere on server
  Python path: wherever you installed it

Check which you have:
  docker ps | grep n8n
  If you see n8n in the list = SCENARIO A
  If nothing = SCENARIO B

I am building for SCENARIO A (Docker) with
volume mount so scraper.py is accessible.
Adjust paths if you are on SCENARIO B.
```

---

## Master Orchestrator — Complete Workflow JSON

```json
{
  "name": "LI Intel — Master Orchestrator",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "0 9 * * 2,3,4"
            },
            {
              "field": "cronExpression",
              "expression": "0 13 * * 2,3,4"
            },
            {
              "field": "cronExpression",
              "expression": "0 18 * * 2,3,4"
            },
            {
              "field": "cronExpression",
              "expression": "0 9 * * 1,5"
            }
          ]
        }
      },
      "name": "Cron Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.3,
      "position": [0, 300],
      "id": "n1"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nimport random\nimport time\nfrom datetime import datetime\nfrom pathlib import Path\n\n# ── Generate session ID ───────────────────────────────────────────\nts = int(datetime.utcnow().timestamp())\nrand = random.randint(1000, 9999)\nsession_id = f'sess_{ts}_{rand}'\n\n# ── Check freeze state ────────────────────────────────────────────\nfreeze_file = Path('/data/linkedin_intel/state/freeze.json')\nfrozen = False\nfreeze_reason = ''\n\nif freeze_file.exists():\n    try:\n        state = json.loads(freeze_file.read_text())\n        if state.get('frozen', False):\n            resume_str = state.get('resumeAfter', '')\n            if resume_str:\n                resume_dt = datetime.fromisoformat(resume_str)\n                now = datetime.utcnow()\n                if now < resume_dt:\n                    frozen = True\n                    freeze_reason = state.get('reason', 'UNKNOWN')\n                else:\n                    # Freeze period has passed — auto unfreeze\n                    freeze_file.write_text(\n                        json.dumps({'frozen': False, 'unfrozenAt': now.isoformat()})\n                    )\n    except Exception as e:\n        freeze_reason = f'FREEZE_READ_ERROR: {str(e)}'\n\n# ── Blackout day check ────────────────────────────────────────────\n# Every 14th day from start of year is a rest day\nday_of_year = datetime.utcnow().timetuple().tm_yday\nis_blackout = (day_of_year % 14 == 0)\n\n# ── Add jitter delay ─────────────────────────────────────────────\n# Random 0 to 8 minutes so cron does not fire at exact same second\njitter_seconds = random.randint(0, 480)\ntime.sleep(jitter_seconds)\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'frozen': frozen,\n            'freezeReason': freeze_reason,\n            'isBlackout': is_blackout,\n            'startedAt': datetime.utcnow().isoformat(),\n            'jitterSeconds': jitter_seconds\n        }\n    }\n]"
      },
      "name": "Build Session + Freeze Check",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [220, 300],
      "id": "n2"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "c1",
              "leftValue": "={{ $json.frozen }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            },
            {
              "id": "c2",
              "leftValue": "={{ $json.isBlackout }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "or"
        }
      },
      "name": "Frozen OR Blackout?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [440, 300],
      "id": "n3"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\nreason = _items[0]['json'].get('freezeReason', '')\nis_blackout = _items[0]['json'].get('isBlackout', False)\n\nlog_reason = 'BLACKOUT_DAY' if is_blackout else f'FROZEN: {reason}'\n\n# Write to log\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SESSION_SKIPPED',\n        'sessionId': session_id,\n        'reason': log_reason,\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'skipped': True,\n            'reason': log_reason,\n            'sessionId': session_id\n        }\n    }\n]"
      },
      "name": "Log Skip Reason",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [660, 160],
      "id": "n4"
    },

    {
      "parameters": {},
      "name": "Stop — Skipped",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [880, 160],
      "id": "n5"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nimport sqlite3\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\n\n# ── Ensure database and tables exist ─────────────────────────────\ndb_path = '/data/linkedin_intel/leads.db'\nPath(db_path).parent.mkdir(parents=True, exist_ok=True)\n\nconn = sqlite3.connect(db_path)\nconn.execute('PRAGMA journal_mode=WAL')\n\nconn.executescript('''\n    CREATE TABLE IF NOT EXISTS leads (\n        id TEXT PRIMARY KEY,\n        author_name TEXT DEFAULT \"\",\n        author_url TEXT DEFAULT \"\",\n        post_text TEXT DEFAULT \"\",\n        emails TEXT DEFAULT \"[]\",\n        hashtags TEXT DEFAULT \"[]\",\n        tech_stack TEXT DEFAULT \"[]\",\n        role_title TEXT DEFAULT \"\",\n        experience_level TEXT DEFAULT \"\",\n        remote_signal TEXT DEFAULT \"\",\n        location TEXT DEFAULT \"\",\n        salary TEXT DEFAULT \"\",\n        relevance_score INTEGER DEFAULT 0,\n        bait_score INTEGER DEFAULT 0,\n        status TEXT DEFAULT \"NEW\",\n        session_id TEXT DEFAULT \"\",\n        created_at TEXT DEFAULT \"\",\n        notes TEXT DEFAULT \"\"\n    );\n\n    CREATE TABLE IF NOT EXISTS sessions (\n        session_id TEXT PRIMARY KEY,\n        started_at TEXT,\n        ended_at TEXT DEFAULT \"\",\n        posts_scanned INTEGER DEFAULT 0,\n        leads_stored INTEGER DEFAULT 0,\n        detection_event TEXT DEFAULT \"\",\n        status TEXT DEFAULT \"RUNNING\"\n    );\n\n    CREATE TABLE IF NOT EXISTS email_seen (\n        email_hash TEXT PRIMARY KEY,\n        email TEXT,\n        first_seen TEXT\n    );\n\n    CREATE INDEX IF NOT EXISTS idx_leads_status\n        ON leads(status);\n    CREATE INDEX IF NOT EXISTS idx_leads_score\n        ON leads(relevance_score DESC);\n    CREATE INDEX IF NOT EXISTS idx_leads_created\n        ON leads(created_at DESC);\n'''\n)\n\n# ── Check session limit for today ─────────────────────────────────\ntoday = datetime.utcnow().date().isoformat()\nrow = conn.execute(\n    \"SELECT COUNT(*) FROM sessions WHERE started_at LIKE ? AND status != 'RUNNING'\",\n    (f'{today}%',)\n).fetchone()\ntoday_count = row[0]\n\n# ── Register this session as started ─────────────────────────────\nconn.execute(\n    'INSERT OR IGNORE INTO sessions (session_id, started_at, status) VALUES (?, ?, ?)',\n    (session_id, datetime.utcnow().isoformat(), 'RUNNING')\n)\nconn.commit()\nconn.close()\n\nmax_sessions = 3\ncan_proceed = today_count < max_sessions\n\nreturn [\n    {\n        'json': {\n            **_items[0]['json'],\n            'canProceed': can_proceed,\n            'todaySessionCount': today_count,\n            'maxSessionsPerDay': max_sessions\n        }\n    }\n]"
      },
      "name": "Init DB + Session Limit Check",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [660, 440],
      "id": "n6"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "c3",
              "leftValue": "={{ $json.canProceed }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        }
      },
      "name": "Can Proceed?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [880, 440],
      "id": "n7"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\ncount = _items[0]['json'].get('todaySessionCount', 0)\nmax_s = _items[0]['json'].get('maxSessionsPerDay', 3)\n\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SESSION_LIMIT_REACHED',\n        'sessionId': session_id,\n        'todayCount': count,\n        'maxAllowed': max_s,\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'skipped': True,\n            'reason': f'SESSION_LIMIT_{count}_of_{max_s}',\n            'sessionId': session_id\n        }\n    }\n]"
      },
      "name": "Log Limit Reached",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1100, 580],
      "id": "n8"
    },

    {
      "parameters": {},
      "name": "Stop — Limit",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [1320, 580],
      "id": "n9"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nimport time\nimport requests\nfrom stem import Signal\nfrom stem.control import Controller\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\n\n# ── Configuration ─────────────────────────────────────────────────\nTOR_PASSWORD = 'YOUR_TOR_PASSWORD_HERE'\nTOR_CONTROL_PORT = 9051\nTOR_SOCKS_PORT = 9050\nALLOWED_COUNTRIES = {'US', 'GB', 'DE', 'CA', 'AU'}\nBLOCKED_ORGS = ['Amazon', 'Google', 'Microsoft', 'DigitalOcean', 'Cloudflare', 'Linode']\n\nTOR_PROXIES = {\n    'http': 'socks5h://127.0.0.1:9050',\n    'https': 'socks5h://127.0.0.1:9050'\n}\n\n# ── Rotate Tor circuit ────────────────────────────────────────────\ndef rotate_circuit():\n    try:\n        with Controller.from_port(port=TOR_CONTROL_PORT) as ctrl:\n            ctrl.authenticate(password=TOR_PASSWORD)\n            ctrl.signal(Signal.NEWNYM)\n        time.sleep(5)\n        return True\n    except Exception as e:\n        return False\n\n# ── Get current IP info ───────────────────────────────────────────\ndef get_ip_info():\n    try:\n        r = requests.get(\n            'https://ipinfo.io/json',\n            proxies=TOR_PROXIES,\n            timeout=15\n        )\n        return r.json()\n    except Exception:\n        return {}\n\n# ── Validate IP is usable ─────────────────────────────────────────\ndef is_usable(info):\n    country = info.get('country', '')\n    org = info.get('org', '')\n    if country not in ALLOWED_COUNTRIES:\n        return False, f'Country not allowed: {country}'\n    if any(b in org for b in BLOCKED_ORGS):\n        return False, f'Datacenter org: {org}'\n    return True, 'OK'\n\n# ── Try up to 3 times ─────────────────────────────────────────────\nip_result = {\n    'success': False,\n    'ip': None,\n    'country': None,\n    'timezone': 'America/New_York',\n    'attempts': 0\n}\n\nfor attempt in range(1, 4):\n    ip_result['attempts'] = attempt\n    rotated = rotate_circuit()\n    if not rotated:\n        continue\n    info = get_ip_info()\n    if not info:\n        continue\n    usable, reason = is_usable(info)\n    if usable:\n        ip_result['success'] = True\n        ip_result['ip'] = info.get('ip', 'unknown')\n        ip_result['country'] = info.get('country', 'unknown')\n        ip_result['timezone'] = info.get('timezone', 'America/New_York')\n        break\n\n# ── Log result ────────────────────────────────────────────────────\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'IP_ROTATION',\n        'sessionId': session_id,\n        'result': ip_result\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            **_items[0]['json'],\n            'ipRotated': ip_result['success'],\n            'currentIp': ip_result['ip'],\n            'ipCountry': ip_result['country'],\n            'sessionTimezone': ip_result['timezone'],\n            'rotationAttempts': ip_result['attempts']\n        }\n    }\n]"
      },
      "name": "Rotate Tor IP",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1100, 300],
      "id": "n10"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "c4",
              "leftValue": "={{ $json.ipRotated }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        }
      },
      "name": "IP Rotation OK?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [1320, 300],
      "id": "n11"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\nattempts = _items[0]['json'].get('rotationAttempts', 3)\n\n# Update session to failed\nimport sqlite3\nconn = sqlite3.connect('/data/linkedin_intel/leads.db')\nconn.execute(\n    \"UPDATE sessions SET status='FAILED', ended_at=? WHERE session_id=?\",\n    (datetime.utcnow().isoformat(), session_id)\n)\nconn.commit()\nconn.close()\n\n# Log\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'IP_ROTATION_FAILED',\n        'sessionId': session_id,\n        'attempts': attempts,\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'failed': True,\n            'reason': 'IP_ROTATION_FAILED',\n            'sessionId': session_id\n        }\n    }\n]"
      },
      "name": "Log IP Failure + Mark Failed",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1540, 440],
      "id": "n12"
    },

    {
      "parameters": {
        "url": "http://localhost:8080/message",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "YOUR_GOTIFY_TOKEN_HERE"
            }
          ]
        },
        "sendBody": true,
        "contentType": "json",
        "body": "={\"title\": \"⚠️ LI Intel: IP Rotation Failed\", \"message\": \"Session \" + $json.sessionId + \" could not get a usable Tor IP after 3 attempts. Session aborted.\", \"priority\": 6}"
      },
      "name": "Notify IP Failure",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1760, 440],
      "id": "n13"
    },

    {
      "parameters": {},
      "name": "Stop — IP Failed",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [1980, 440],
      "id": "n14"
    },

    {
      "parameters": {
        "command": "=python3 /data/scripts/scraper.py {{ $json.sessionId }}"
      },
      "name": "Run Scraper Script",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [1540, 160],
      "id": "n15"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\n\n# Get stdout from Execute Command node\nstdout = _items[0]['json'].get('stdout', '{}')\nstderr = _items[0]['json'].get('stderr', '')\nexit_code = _items[0]['json'].get('exitCode', 0)\n\n# Retrieve session data passed through\n# Note: Execute Command node replaces all json with stdout/stderr/exitCode\n# We need to read session_id from stdout since input json is overwritten\ntry:\n    scraper_result = json.loads(stdout)\nexcept Exception:\n    scraper_result = {\n        'status': 'PARSE_ERROR',\n        'sessionId': 'unknown',\n        'postsFound': 0,\n        'detectionEvent': None,\n        'rawFile': '',\n        'query': ''\n    }\n\nstatus = scraper_result.get('status', 'UNKNOWN')\nsession_id = scraper_result.get('sessionId', 'unknown')\nposts_found = scraper_result.get('postsFound', 0)\ndetection_event = scraper_result.get('detectionEvent', None)\nraw_file = scraper_result.get('rawFile', '')\nquery_used = scraper_result.get('query', '')\n\n# Determine if scraper completed enough to process\ncan_process = status in ['COMPLETED', 'PARTIAL'] and posts_found > 0\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'scraperStatus': status,\n            'postsFound': posts_found,\n            'detectionEvent': detection_event,\n            'rawFile': raw_file,\n            'queryUsed': query_used,\n            'canProcess': can_process,\n            'exitCode': exit_code,\n            'hasStderr': bool(stderr)\n        }\n    }\n]"
      },
      "name": "Parse Scraper Output",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1760, 160],
      "id": "n16"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "c5",
              "leftValue": "={{ $json.canProcess }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        }
      },
      "name": "Posts Found?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [1980, 160],
      "id": "n17"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nfrom datetime import datetime\nfrom pathlib import Path\nimport sqlite3\n\nsession_id = _items[0]['json']['sessionId']\nstatus = _items[0]['json'].get('scraperStatus', 'FAILED')\ndetection_event = _items[0]['json'].get('detectionEvent', None)\n\n# Update session record\nconn = sqlite3.connect('/data/linkedin_intel/leads.db')\nconn.execute(\n    \"UPDATE sessions SET status=?, ended_at=?, detection_event=? WHERE session_id=?\",\n    ('FAILED', datetime.utcnow().isoformat(), str(detection_event), session_id)\n)\nconn.commit()\nconn.close()\n\n# Write log\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SCRAPER_FAILED_NO_POSTS',\n        'sessionId': session_id,\n        'scraperStatus': status,\n        'detectionEvent': detection_event,\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'failed': True,\n            'sessionId': session_id,\n            'reason': status,\n            'detectionEvent': detection_event\n        }\n    }\n]"
      },
      "name": "Log Scraper Failed",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2200, 300],
      "id": "n18"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nimport re\nimport sqlite3\nimport hashlib\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\nraw_file = _items[0]['json'].get('rawFile', '')\n\n# ── YOUR PROFILE — EDIT THESE ─────────────────────────────────────\nYOUR_STACK = {\n    'python', 'django', 'fastapi', 'flask',\n    'postgresql', 'postgres', 'redis', 'docker',\n    'kubernetes', 'aws', 'gcp', 'react',\n    'typescript', 'golang', 'kafka', 'terraform'\n}\nYOUR_ROLES = {\n    'senior software engineer',\n    'backend engineer',\n    'senior backend engineer',\n    'platform engineer',\n    'staff engineer',\n    'senior developer',\n    'software architect',\n    'devops engineer',\n    'site reliability engineer',\n    'sre'\n}\nYOUR_LEVEL = 'senior'\nPREFER_REMOTE = True\nHIGH_PRIORITY_THRESHOLD = 7\nMIN_SCORE_TO_STORE = 2\n\n# ── Engagement bait patterns ──────────────────────────────────────\nBAIT = [\n    (r'comment\\s+[\"\\']?yes[\"\\']?', -3),\n    (r'type\\s+[\"\\']?yes[\"\\']?\\s+if', -3),\n    (r'drop\\s+a\\s+[\"\\']?yes[\"\\']?', -3),\n    (r'post\\s+your\\s+jobs?\\s+in\\s+the\\s+comments', -3),\n    (r'who\\s+needs?\\s+to\\s+hear\\s+this', -2),\n    (r'not\\s+sure\\s+who\\s+needs?\\s+to\\s+hear', -2),\n    (r'tag\\s+someone\\s+who', -2),\n    (r'repost\\s+to\\s+help', -1),\n    (r'share\\s+this\\s+with\\s+your\\s+network', -1),\n    (r'agree\\??\\s*$', -2)\n]\n\n# ── Real post signals ─────────────────────────────────────────────\nREAL = [\n    (r'\\d+\\s*(?:years?|yrs?)\\s+(?:of\\s+)?experience', 2),\n    (r'(?:remote|hybrid|onsite|on-site)', 1),\n    (r'(?:salary|compensation|comp).*\\$[\\d,]+', 2),\n    (r\"i'?m\\s+personally\\s+reviewing\", 2),\n    (r'send\\s+(?:me|your)\\s+(?:resume|cv)', 2),\n    (r'(?:immediately|asap|urgent)', 1),\n    (r'(?:full.?time|contract|freelance)', 1)\n]\n\n# ── Helper functions ──────────────────────────────────────────────\ndef extract_emails(text):\n    patterns = [\n        r'[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}',\n        r'(?i)email\\s*(?:me\\s*at|:)\\s*([a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,})',\n        r'(?i)reach\\s+(?:me|out)\\s+at\\s+([a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,})',\n        r'(?i)contact\\s+me\\s+at\\s*:?\\s*([a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,})'\n    ]\n    found = set()\n    for p in patterns:\n        for m in re.findall(p, text):\n            e = m.strip().rstrip('.,;:)')\n            if '@' in e and '.' in e.split('@')[-1]:\n                if not any(x in e for x in ['example.com', 'test@', 'youremail']):\n                    found.add(e.lower())\n    return list(found)\n\ndef get_bait_score(text):\n    score = 0\n    for pattern, weight in BAIT:\n        if re.search(pattern, text, re.I):\n            score += weight\n    return score\n\ndef get_relevance_score(post):\n    text = post.get('text', '').lower()\n    score = 0\n\n    # Stack match +1 per tech found\n    found_stack = []\n    for tech in YOUR_STACK:\n        if re.search(r'\\b' + re.escape(tech) + r'\\b', text, re.I):\n            found_stack.append(tech)\n    score += len(found_stack)\n    post['tech_stack'] = found_stack\n\n    # Role match\n    role_found = ''\n    for role in YOUR_ROLES:\n        if role in text:\n            role_found = role.title()\n            score += 3\n            break\n    if not role_found:\n        # Partial match\n        for role in YOUR_ROLES:\n            words = role.split()\n            if any(w in text for w in words if len(w) > 4):\n                role_found = role.title()\n                score += 1\n                break\n    post['role_title'] = role_found\n\n    # Experience level\n    exp = 'unknown'\n    if re.search(r'(?:senior|sr\\.?|5\\+\\s*years?|6\\+\\s*years?|7\\+\\s*years?)', text, re.I):\n        exp = 'senior'\n        if YOUR_LEVEL == 'senior':\n            score += 2\n    elif re.search(r'(?:lead|principal|staff|8\\+\\s*years?|10\\+\\s*years?)', text, re.I):\n        exp = 'lead'\n        score += 1\n    elif re.search(r'(?:junior|entry|0-2\\s*years?|1-2\\s*years?)', text, re.I):\n        exp = 'entry'\n    elif re.search(r'(?:mid.?level|2-4\\s*years?|3-5\\s*years?)', text, re.I):\n        exp = 'mid'\n    post['experience_level'] = exp\n\n    # Remote signal\n    remote = 'unknown'\n    if re.search(r'\\bfully\\s+remote\\b|\\bremote\\s+first\\b|\\b100%\\s+remote\\b', text, re.I):\n        remote = 'remote'\n        if PREFER_REMOTE:\n            score += 2\n    elif re.search(r'\\bremote\\b', text, re.I):\n        remote = 'remote'\n        if PREFER_REMOTE:\n            score += 1\n    elif re.search(r'\\bhybrid\\b', text, re.I):\n        remote = 'hybrid'\n        if PREFER_REMOTE:\n            score += 1\n    elif re.search(r'\\bonsite\\b|\\bon-site\\b|\\bin.office\\b', text, re.I):\n        remote = 'onsite'\n    post['remote_signal'] = remote\n\n    # Location\n    loc_match = re.search(\n        r'\\b(?:New York|San Francisco|London|Austin|Seattle|'\n        r'Chicago|Boston|Los Angeles|Berlin|Toronto|Vancouver|'\n        r'Remote|Worldwide|Anywhere)\\b',\n        text, re.I\n    )\n    post['location'] = loc_match.group(0) if loc_match else ''\n\n    # Salary\n    sal_match = re.search(\n        r'\\$[\\d,]+(?:k|K)?(?:\\s*[-\\u2013]\\s*\\$[\\d,]+(?:k|K)?)?', text\n    )\n    post['salary'] = sal_match.group(0) if sal_match else ''\n\n    # Real post signals\n    for pattern, weight in REAL:\n        if re.search(pattern, text, re.I):\n            score += weight\n\n    return score\n\n# ── Load raw posts ────────────────────────────────────────────────\nif not raw_file or not Path(raw_file).exists():\n    return [\n        {\n            'json': {\n                'sessionId': session_id,\n                'leadsStored': 0,\n                'highPriorityLeads': [],\n                'totalProcessed': 0,\n                'error': 'Raw file not found'\n            }\n        }\n    ]\n\ntry:\n    posts = json.loads(Path(raw_file).read_text())\nexcept Exception as e:\n    return [\n        {\n            'json': {\n                'sessionId': session_id,\n                'leadsStored': 0,\n                'highPriorityLeads': [],\n                'totalProcessed': 0,\n                'error': f'Raw file parse error: {str(e)}'\n            }\n        }\n    ]\n\n# ── Connect to database ───────────────────────────────────────────\ndb_path = '/data/linkedin_intel/leads.db'\nconn = sqlite3.connect(db_path)\nconn.execute('PRAGMA journal_mode=WAL')\n\nnow = datetime.utcnow().isoformat()\nleads_stored = 0\nhigh_priority = []\n\nfor post in posts:\n    text = post.get('text', '')\n    if not text:\n        continue\n\n    # Extract emails\n    emails = extract_emails(text)\n    if not emails:\n        continue\n\n    # Bait filter\n    bait_sc = get_bait_score(text)\n    if bait_sc < -2:\n        continue\n\n    # Score\n    rel_sc = get_relevance_score(post)\n\n    # Skip very low relevance\n    if rel_sc < MIN_SCORE_TO_STORE:\n        continue\n\n    # Generate lead ID\n    author_url = post.get('author_url', '')\n    found_at = post.get('found_at', now)\n    lead_id = hashlib.sha256(\n        f'{author_url}::{found_at}'.encode()\n    ).hexdigest()\n\n    # Check post-level duplicate\n    row = conn.execute(\n        'SELECT id FROM leads WHERE id = ?', (lead_id,)\n    ).fetchone()\n    if row:\n        continue\n\n    # Check email-level duplicate\n    dup_email = False\n    for email in emails:\n        email_hash = hashlib.sha256(email.encode()).hexdigest()\n        row = conn.execute(\n            'SELECT email_hash FROM email_seen WHERE email_hash = ?',\n            (email_hash,)\n        ).fetchone()\n        if row:\n            dup_email = True\n            break\n    if dup_email:\n        continue\n\n    # Save lead\n    try:\n        conn.execute('''\n            INSERT INTO leads (\n                id, author_name, author_url, post_text,\n                emails, hashtags, tech_stack, role_title,\n                experience_level, remote_signal, location,\n                salary, relevance_score, bait_score,\n                status, session_id, created_at\n            ) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)\n        ''', (\n            lead_id,\n            post.get('author_name', ''),\n            author_url,\n            text,\n            json.dumps(emails),\n            json.dumps(post.get('hashtags', [])),\n            json.dumps(post.get('tech_stack', [])),\n            post.get('role_title', ''),\n            post.get('experience_level', ''),\n            post.get('remote_signal', ''),\n            post.get('location', ''),\n            post.get('salary', ''),\n            rel_sc,\n            bait_sc,\n            'NEW',\n            session_id,\n            now\n        ))\n\n        # Register emails in dedup table\n        for email in emails:\n            email_hash = hashlib.sha256(email.encode()).hexdigest()\n            conn.execute(\n                'INSERT OR IGNORE INTO email_seen (email_hash, email, first_seen) VALUES (?,?,?)',\n                (email_hash, email, now)\n            )\n\n        conn.commit()\n        leads_stored += 1\n\n        # Collect high priority\n        if rel_sc >= HIGH_PRIORITY_THRESHOLD:\n            high_priority.append({\n                'role_title': post.get('role_title', 'Unknown Role'),\n                'author_name': post.get('author_name', ''),\n                'emails': emails,\n                'tech_stack': post.get('tech_stack', []),\n                'remote_signal': post.get('remote_signal', ''),\n                'relevance_score': rel_sc,\n                'salary': post.get('salary', '')\n            })\n\n    except Exception:\n        conn.rollback()\n        continue\n\n# Update session record\nconn.execute(\n    'UPDATE sessions SET posts_scanned=?, leads_stored=? WHERE session_id=?',\n    (len(posts), leads_stored, session_id)\n)\nconn.commit()\nconn.close()\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'leadsStored': leads_stored,\n            'highPriorityLeads': high_priority,\n            'highPriorityCount': len(high_priority),\n            'totalProcessed': len(posts)\n        }\n    }\n]"
      },
      "name": "Extract + Score + Save Leads",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2200, 60],
      "id": "n19"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "c6",
              "leftValue": "={{ $json.highPriorityCount }}",
              "rightValue": 0,
              "operator": {
                "type": "number",
                "operation": "gt"
              }
            }
          ],
          "combinator": "and"
        }
      },
      "name": "High Priority Found?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [2420, 60],
      "id": "n20"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\n\nleads = _items[0]['json'].get('highPriorityLeads', [])\nsession_id = _items[0]['json'].get('sessionId', 'unknown')\ntotal = _items[0]['json'].get('leadsStored', 0)\n\n# Build notification message\nlines = []\nfor lead in leads[:5]:  # Max 5 in one notification\n    role = lead.get('role_title', 'Unknown Role')\n    author = lead.get('author_name', '')\n    emails = lead.get('emails', [])\n    score = lead.get('relevance_score', 0)\n    remote = lead.get('remote_signal', '')\n    stack = ', '.join(lead.get('tech_stack', [])[:3])\n    email_str = emails[0] if emails else 'no email'\n    lines.append(f'{role} ({score}/15) | {email_str} | {remote} | {stack}')\n\nmessage = f'Session: {session_id}\\nTotal stored: {total}\\n\\n' + '\\n'.join(lines)\n\nif len(leads) > 5:\n    message += f'\\n\\n...and {len(leads) - 5} more. Check dashboard.'\n\nreturn [\n    {\n        'json': {\n            'notificationTitle': f'⭐ {len(leads)} High Priority Lead(s) Found',\n            'notificationMessage': message,\n            'notificationPriority': 8,\n            'sessionId': session_id,\n            'leadsStored': total,\n            'highPriorityCount': len(leads)\n        }\n    }\n]"
      },
      "name": "Build Notification Message",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2640, -80],
      "id": "n21"
    },

    {
      "parameters": {
        "method": "POST",
        "url": "http://localhost:8080/message",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "YOUR_GOTIFY_TOKEN_HERE"
            }
          ]
        },
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ \"title\": $json.notificationTitle, \"message\": $json.notificationMessage, \"priority\": $json.notificationPriority }) }}"
      },
      "name": "Send Phone Notification",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [2860, -80],
      "id": "n22"
    },

    {
      "parameters": {
        "mode": "runOnceForAllItems",
        "language": "python",
        "jsCode": "import json\nimport sqlite3\nfrom datetime import datetime\nfrom pathlib import Path\n\n# This node receives from multiple paths\n# It reads from whichever path completed last\ninput_data = _items[0]['json'] if _items else {}\nsession_id = input_data.get('sessionId', 'unknown')\ndetection_event = input_data.get('detectionEvent', None)\nleads_stored = input_data.get('leadsStored', 0)\n\n# ── Handle detection event if any ────────────────────────────────\nif detection_event and detection_event not in ['None', None, '']:\n    freeze_hours = 48 if 'HARD' in str(detection_event) else 6\n    from datetime import timedelta\n    resume_time = (datetime.utcnow() + timedelta(hours=freeze_hours)).isoformat()\n\n    freeze_file = Path('/data/linkedin_intel/state/freeze.json')\n    freeze_file.parent.mkdir(parents=True, exist_ok=True)\n    freeze_file.write_text(json.dumps({\n        'frozen': True,\n        'reason': str(detection_event),\n        'frozenAt': datetime.utcnow().isoformat(),\n        'resumeAfter': resume_time,\n        'freezeHours': freeze_hours\n    }))\n\n# ── Archive raw posts file ────────────────────────────────────────\nraw_file = input_data.get('rawFile', '')\nif raw_file:\n    src = Path(raw_file)\n    if src.exists():\n        archive_dir = Path('/data/linkedin_intel/raw_posts/archive')\n        archive_dir.mkdir(parents=True, exist_ok=True)\n        src.rename(archive_dir / src.name)\n\n# ── Update session to completed ───────────────────────────────────\ntry:\n    conn = sqlite3.connect('/data/linkedin_intel/leads.db')\n    conn.execute(\n        \"UPDATE sessions SET status='COMPLETED', ended_at=?, leads_stored=? WHERE session_id=?\",\n        (datetime.utcnow().isoformat(), leads_stored, session_id)\n    )\n    conn.commit()\n    conn.close()\nexcept Exception:\n    pass\n\n# ── Write final session log ───────────────────────────────────────\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SESSION_COMPLETE',\n        'sessionId': session_id,\n        'leadsStored': leads_stored,\n        'detectionEvent': str(detection_event),\n        'completedAt': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'sessionComplete': True,\n            'leadsStored': leads_stored,\n            'detectionHandled': bool(detection_event),\n            'completedAt': datetime.utcnow().isoformat()\n        }\n    }\n]"
      },
      "name": "Finalize Session",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [3080, 60],
      "id": "n23"
    }
  ],

  "connections": {
    "Cron Trigger": {
      "main": [
        [{ "node": "Build Session + Freeze Check", "type": "main", "index": 0 }]
      ]
    },
    "Build Session + Freeze Check": {
      "main": [
        [{ "node": "Frozen OR Blackout?", "type": "main", "index": 0 }]
      ]
    },
    "Frozen OR Blackout?": {
      "main": [
        [{ "node": "Log Skip Reason", "type": "main", "index": 0 }],
        [{ "node": "Init DB + Session Limit Check", "type": "main", "index": 0 }]
      ]
    },
    "Log Skip Reason": {
      "main": [
        [{ "node": "Stop — Skipped", "type": "main", "index": 0 }]
      ]
    },
    "Init DB + Session Limit Check": {
      "main": [
        [{ "node": "Can Proceed?", "type": "main", "index": 0 }]
      ]
    },
    "Can Proceed?": {
      "main": [
        [{ "node": "Log Limit Reached", "type": "main", "index": 0 }],
        [{ "node": "Rotate Tor IP", "type": "main", "index": 0 }]
      ]
    },
    "Log Limit Reached": {
      "main": [
        [{ "node": "Stop — Limit", "type": "main", "index": 0 }]
      ]
    },
    "Rotate Tor IP": {
      "main": [
        [{ "node": "IP Rotation OK?", "type": "main", "index": 0 }]
      ]
    },
    "IP Rotation OK?": {
      "main": [
        [{ "node": "Log IP Failure + Mark Failed", "type": "main", "index": 0 }],
        [{ "node": "Run Scraper Script", "type": "main", "index": 0 }]
      ]
    },
    "Log IP Failure + Mark Failed": {
      "main": [
        [{ "node": "Notify IP Failure", "type": "main", "index": 0 }]
      ]
    },
    "Notify IP Failure": {
      "main": [
        [{ "node": "Stop — IP Failed", "type": "main", "index": 0 }]
      ]
    },
    "Run Scraper Script": {
      "main": [
        [{ "node": "Parse Scraper Output", "type": "main", "index": 0 }]
      ]
    },
    "Parse Scraper Output": {
      "main": [
        [{ "node": "Posts Found?", "type": "main", "index": 0 }]
      ]
    },
    "Posts Found?": {
      "main": [
        [{ "node": "Log Scraper Failed", "type": "main", "index": 0 }],
        [{ "node": "Extract + Score + Save Leads", "type": "main", "index": 0 }]
      ]
    },
    "Log Scraper Failed": {
      "main": [
        [{ "node": "Finalize Session", "type": "main", "index": 0 }]
      ]
    },
    "Extract + Score + Save Leads": {
      "main": [
        [{ "node": "High Priority Found?", "type": "main", "index": 0 }]
      ]
    },
    "High Priority Found?": {
      "main": [
        [{ "node": "Finalize Session", "type": "main", "index": 0 }],
        [{ "node": "Build Notification Message", "type": "main", "index": 0 }]
      ]
    },
    "Build Notification Message": {
      "main": [
        [{ "node": "Send Phone Notification", "type": "main", "index": 0 }]
      ]
    },
    "Send Phone Notification": {
      "main": [
        [{ "node": "Finalize Session", "type": "main", "index": 0 }]
      ]
    }
  },

  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "errorWorkflow": ""
  },

  "staticData": null,
  "tags": ["linkedin-intel"]
}
```

---

## Two Things To Replace Before Importing

```
FIND AND REPLACE IN THE JSON:

1. YOUR_TOR_PASSWORD_HERE
   Replace with your actual Tor control password
   Appears in: Rotate Tor IP node

2. YOUR_GOTIFY_TOKEN_HERE
   Replace with your actual Gotify app token
   Appears in: Notify IP Failure node
               Send Phone Notification node

How to find your Gotify token:
  Open http://your-server:8080
  Login → Apps → your app → copy the token
```

---

## Node Map (What You Will See After Import)

```
CRON TRIGGER
    │
    ▼
BUILD SESSION + FREEZE CHECK
    │
    ▼
FROZEN OR BLACKOUT? ──── YES ──► LOG SKIP REASON ──► STOP SKIPPED
    │
    NO
    │
    ▼
INIT DB + SESSION LIMIT CHECK
    │
    ▼
CAN PROCEED? ──── NO ──► LOG LIMIT REACHED ──► STOP LIMIT
    │
    YES
    │
    ▼
ROTATE TOR IP
    │
    ▼
IP ROTATION OK? ──── NO ──► LOG IP FAILURE ──► NOTIFY IP FAILURE ──► STOP IP FAILED
    │
    YES
    │
    ▼
RUN SCRAPER SCRIPT   ← only external script called
    │
    ▼
PARSE SCRAPER OUTPUT
    │
    ▼
POSTS FOUND? ──── NO ──► LOG SCRAPER FAILED ──► FINALIZE SESSION
    │
    YES
    │
    ▼
EXTRACT + SCORE + SAVE LEADS
    │
    ▼
HIGH PRIORITY FOUND? ──── NO ──► FINALIZE SESSION
    │
    YES
    │
    ▼
BUILD NOTIFICATION MESSAGE
    │
    ▼
SEND PHONE NOTIFICATION
    │
    ▼
FINALIZE SESSION
```

---

## After Importing — Three Verification Steps

```
STEP 1: Check all nodes are recognized
  Import the JSON
  Every node should show its name with colored icon
  If any node shows grey with question mark = unknown node
  Most likely cause = Execute Command still disabled
  Fix: add NODES_EXCLUDE=[] to n8n environment variables
       then restart n8n

STEP 2: Test manually before activating cron
  Click Execute Workflow button (top right)
  Watch each node turn green one by one
  The scraper will try to run
  If scraper.py does not exist yet it will fail at that node
  That is expected at this stage

STEP 3: Check the IF node connections
  In n8n IF nodes have two outputs
  Output 1 (left/top) = TRUE path
  Output 2 (right/bottom) = FALSE path
  Verify each IF node connects to correct next node
  The JSON sets this but visual confirmation recommended

Ready for next workflow?
  Tell me which one to build next:
  A. Daily Maintenance + Backup workflow
  B. Freeze Guard Monitor workflow
  C. Error Handler workflow
  All three are smaller and faster to build
```