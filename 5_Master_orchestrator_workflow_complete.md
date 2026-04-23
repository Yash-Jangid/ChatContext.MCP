## Architecture Shift Understood

```
BEFORE (broken):
  n8n Code node → subprocess → scraper.py
  Fails because n8n container has no Python

AFTER (working):
  n8n HTTP Request node → scraper-service:8000 → JSON response
  Works because scraper runs in its own Python container

This changes ONE node in the entire workflow.
Everything else stays identical.
```

---

## What Changes In The Workflow

```
REMOVE:  Run Scraper Script (Code node with subprocess)
ADD:     Call Scraper Service (HTTP Request node)

Input:   sessionId from previous node
Output:  status, postsFound, detectionEvent, rawFile, query
Same output structure as before.
Parse Scraper Output node reads it identically.
Zero other changes needed.
```

---

## The Replacement Node JSON

```json
{
  "parameters": {
    "method": "GET",
    "url": "=http://scraper-service:8000/?sessionId={{ $json.sessionId }}",
    "options": {
      "response": {
        "response": {
          "responseFormat": "json"
        }
      }
    }
  },
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [1540, 160],
  "id": "ffffffff-ffff-ffff-ffff-ffffffffffff",
  "name": "Call Scraper Service"
}
```

---

## Updated Parse Scraper Output Node

```
IMPORTANT CHANGE:
Execute Command returned: { stdout, stderr, exitCode }
HTTP Request returns: the JSON directly as $json

So Parse Scraper Output node must change too.
It no longer reads stdout.
It reads $json directly.
```

```json
{
  "parameters": {
    "language": "pythonNative",
    "pythonCode": "import json\n\ndata = _items[0]['json']\n\nstatus = data.get('status', 'UNKNOWN')\nsession_id = data.get('sessionId', 'unknown')\nposts_found = data.get('postsFound', 0)\ndetection_event = data.get('detectionEvent', None)\nraw_file = data.get('rawFile', '')\nquery_used = data.get('query', '')\nerror = data.get('error', '')\n\ncan_process = status in ['COMPLETED', 'PARTIAL'] and posts_found > 0\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'scraperStatus': status,\n            'postsFound': posts_found,\n            'detectionEvent': detection_event,\n            'rawFile': raw_file,\n            'queryUsed': query_used,\n            'canProcess': can_process,\n            'scraperError': error\n        }\n    }\n]"
  },
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [1760, 160],
  "id": "11111111-aaaa-1111-aaaa-111111111111",
  "name": "Parse Scraper Output"
}
```

---

## Complete Final Workflow JSON

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
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [0, 300],
      "id": "11111111-1111-1111-1111-111111111111",
      "name": "Cron Trigger"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nimport random\nimport time\nfrom datetime import datetime\nfrom pathlib import Path\n\nts = int(datetime.utcnow().timestamp())\nrand = random.randint(1000, 9999)\nsession_id = f'sess_{ts}_{rand}'\n\nfreeze_file = Path('/data/linkedin_intel/state/freeze.json')\nfrozen = False\nfreeze_reason = ''\n\nif freeze_file.exists():\n    try:\n        state = json.loads(freeze_file.read_text())\n        if state.get('frozen', False):\n            resume_str = state.get('resumeAfter', '')\n            if resume_str:\n                resume_dt = datetime.fromisoformat(resume_str)\n                now_dt = datetime.utcnow()\n                if now_dt < resume_dt:\n                    frozen = True\n                    freeze_reason = state.get('reason', 'UNKNOWN')\n                else:\n                    freeze_file.write_text(\n                        json.dumps({\n                            'frozen': False,\n                            'unfrozenAt': now_dt.isoformat()\n                        })\n                    )\n    except Exception as ex:\n        freeze_reason = f'FREEZE_READ_ERROR: {str(ex)}'\n\nday_of_year = datetime.utcnow().timetuple().tm_yday\nis_blackout = (day_of_year % 14 == 0)\n\njitter_seconds = random.randint(0, 480)\ntime.sleep(jitter_seconds)\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'frozen': frozen,\n            'freezeReason': freeze_reason,\n            'isBlackout': is_blackout,\n            'startedAt': datetime.utcnow().isoformat(),\n            'jitterSeconds': jitter_seconds\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [220, 300],
      "id": "22222222-2222-2222-2222-222222222222",
      "name": "Build Session + Freeze Check"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "id": "aa111111-1111-1111-1111-111111111111",
              "leftValue": "={{ $json.frozen }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            },
            {
              "id": "aa222222-2222-2222-2222-222222222222",
              "leftValue": "={{ $json.isBlackout }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "or"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [440, 300],
      "id": "33333333-3333-3333-3333-333333333333",
      "name": "Frozen OR Blackout?"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\nreason = _items[0]['json'].get('freezeReason', '')\nis_blackout = _items[0]['json'].get('isBlackout', False)\n\nlog_reason = 'BLACKOUT_DAY' if is_blackout else f'FROZEN: {reason}'\n\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\n\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SESSION_SKIPPED',\n        'sessionId': session_id,\n        'reason': log_reason,\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'skipped': True,\n            'reason': log_reason,\n            'sessionId': session_id\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [660, 160],
      "id": "44444444-4444-4444-4444-444444444444",
      "name": "Log Skip Reason"
    },

    {
      "parameters": {},
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [880, 160],
      "id": "55555555-5555-5555-5555-555555555555",
      "name": "No Operation, do nothing1"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nimport sqlite3\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\n\ndb_path = '/data/linkedin_intel/leads.db'\nPath(db_path).parent.mkdir(parents=True, exist_ok=True)\n\nconn = sqlite3.connect(db_path)\nconn.execute('PRAGMA journal_mode=WAL')\n\nconn.executescript(\"\"\"\n    CREATE TABLE IF NOT EXISTS leads (\n        id TEXT PRIMARY KEY,\n        author_name TEXT DEFAULT '',\n        author_url TEXT DEFAULT '',\n        post_text TEXT DEFAULT '',\n        emails TEXT DEFAULT '[]',\n        hashtags TEXT DEFAULT '[]',\n        tech_stack TEXT DEFAULT '[]',\n        role_title TEXT DEFAULT '',\n        experience_level TEXT DEFAULT '',\n        remote_signal TEXT DEFAULT '',\n        location TEXT DEFAULT '',\n        salary TEXT DEFAULT '',\n        relevance_score INTEGER DEFAULT 0,\n        bait_score INTEGER DEFAULT 0,\n        status TEXT DEFAULT 'NEW',\n        session_id TEXT DEFAULT '',\n        created_at TEXT DEFAULT '',\n        notes TEXT DEFAULT ''\n    );\n    CREATE TABLE IF NOT EXISTS sessions (\n        session_id TEXT PRIMARY KEY,\n        started_at TEXT,\n        ended_at TEXT DEFAULT '',\n        posts_scanned INTEGER DEFAULT 0,\n        leads_stored INTEGER DEFAULT 0,\n        detection_event TEXT DEFAULT '',\n        status TEXT DEFAULT 'RUNNING'\n    );\n    CREATE TABLE IF NOT EXISTS email_seen (\n        email_hash TEXT PRIMARY KEY,\n        email TEXT,\n        first_seen TEXT\n    );\n    CREATE INDEX IF NOT EXISTS idx_leads_status ON leads(status);\n    CREATE INDEX IF NOT EXISTS idx_leads_score ON leads(relevance_score DESC);\n    CREATE INDEX IF NOT EXISTS idx_leads_created ON leads(created_at DESC);\n\"\"\")\n\ntoday = datetime.utcnow().date().isoformat()\nrow = conn.execute(\n    \"SELECT COUNT(*) FROM sessions WHERE started_at LIKE ? AND status != 'RUNNING'\",\n    (f'{today}%',)\n).fetchone()\ntoday_count = row[0]\n\nconn.execute(\n    'INSERT OR IGNORE INTO sessions (session_id, started_at, status) VALUES (?, ?, ?)',\n    (session_id, datetime.utcnow().isoformat(), 'RUNNING')\n)\nconn.commit()\nconn.close()\n\nmax_sessions = 3\ncan_proceed = today_count < max_sessions\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'frozen': _items[0]['json'].get('frozen', False),\n            'isBlackout': _items[0]['json'].get('isBlackout', False),\n            'startedAt': _items[0]['json'].get('startedAt', ''),\n            'canProceed': can_proceed,\n            'todaySessionCount': today_count,\n            'maxSessionsPerDay': max_sessions\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [660, 440],
      "id": "66666666-6666-6666-6666-666666666666",
      "name": "Init DB + Session Limit Check"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "id": "bb111111-1111-1111-1111-111111111111",
              "leftValue": "={{ $json.canProceed }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [880, 440],
      "id": "77777777-7777-7777-7777-777777777777",
      "name": "Can Proceed?"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\ncount = _items[0]['json'].get('todaySessionCount', 0)\nmax_s = _items[0]['json'].get('maxSessionsPerDay', 3)\n\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\n\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SESSION_LIMIT_REACHED',\n        'sessionId': session_id,\n        'todayCount': count,\n        'maxAllowed': max_s,\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'skipped': True,\n            'reason': f'SESSION_LIMIT_{count}_of_{max_s}',\n            'sessionId': session_id\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1100, 580],
      "id": "88888888-8888-8888-8888-888888888888",
      "name": "Log Limit Reached"
    },

    {
      "parameters": {},
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [1320, 580],
      "id": "99999999-9999-9999-9999-999999999999",
      "name": "No Operation, do nothing2"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nimport time\nimport requests\nfrom stem import Signal\nfrom stem.control import Controller\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\n\nTOR_PASSWORD = 'YOUR_TOR_PASSWORD_HERE'\nTOR_CONTROL_PORT = 9051\nALLOWED_COUNTRIES = {'US', 'GB', 'DE', 'CA', 'AU'}\nBLOCKED_ORGS = ['Amazon', 'Google', 'Microsoft', 'DigitalOcean', 'Cloudflare', 'Linode']\n\nTOR_PROXIES = {\n    'http': 'socks5h://127.0.0.1:9050',\n    'https': 'socks5h://127.0.0.1:9050'\n}\n\ndef rotate_circuit():\n    try:\n        with Controller.from_port(port=TOR_CONTROL_PORT) as ctrl:\n            ctrl.authenticate(password=TOR_PASSWORD)\n            ctrl.signal(Signal.NEWNYM)\n        time.sleep(5)\n        return True\n    except Exception:\n        return False\n\ndef get_ip_info():\n    try:\n        r = requests.get(\n            'https://ipinfo.io/json',\n            proxies=TOR_PROXIES,\n            timeout=15\n        )\n        return r.json()\n    except Exception:\n        return {}\n\ndef is_usable(info):\n    country = info.get('country', '')\n    org = info.get('org', '')\n    if country not in ALLOWED_COUNTRIES:\n        return False, f'Country not allowed: {country}'\n    if any(b in org for b in BLOCKED_ORGS):\n        return False, f'Datacenter org: {org}'\n    return True, 'OK'\n\nip_result = {\n    'success': False,\n    'ip': None,\n    'country': None,\n    'timezone': 'America/New_York',\n    'attempts': 0\n}\n\nfor attempt in range(1, 4):\n    ip_result['attempts'] = attempt\n    if not rotate_circuit():\n        continue\n    info = get_ip_info()\n    if not info:\n        continue\n    usable, reason = is_usable(info)\n    if usable:\n        ip_result['success'] = True\n        ip_result['ip'] = info.get('ip', 'unknown')\n        ip_result['country'] = info.get('country', 'unknown')\n        ip_result['timezone'] = info.get('timezone', 'America/New_York')\n        break\n\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'IP_ROTATION',\n        'sessionId': session_id,\n        'result': ip_result\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'startedAt': _items[0]['json'].get('startedAt', ''),\n            'ipRotated': ip_result['success'],\n            'currentIp': ip_result['ip'],\n            'ipCountry': ip_result['country'],\n            'sessionTimezone': ip_result['timezone'],\n            'rotationAttempts': ip_result['attempts']\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1100, 300],
      "id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
      "name": "Rotate Tor IP"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "id": "cc111111-1111-1111-1111-111111111111",
              "leftValue": "={{ $json.ipRotated }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [1320, 300],
      "id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
      "name": "IP Rotation OK?"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nimport sqlite3\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\nattempts = _items[0]['json'].get('rotationAttempts', 3)\n\ntry:\n    conn = sqlite3.connect('/data/linkedin_intel/leads.db')\n    conn.execute(\n        \"UPDATE sessions SET status='FAILED', ended_at=? WHERE session_id=?\",\n        (datetime.utcnow().isoformat(), session_id)\n    )\n    conn.commit()\n    conn.close()\nexcept Exception:\n    pass\n\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'IP_ROTATION_FAILED',\n        'sessionId': session_id,\n        'attempts': attempts,\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'failed': True,\n            'reason': 'IP_ROTATION_FAILED',\n            'sessionId': session_id\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1540, 440],
      "id": "cccccccc-cccc-cccc-cccc-cccccccccccc",
      "name": "Log IP Failure"
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
        "body": "={{ JSON.stringify({ \"title\": \"LI Intel: IP Rotation Failed\", \"message\": \"Session \" + $json.sessionId + \" could not get usable Tor IP after 3 attempts. Aborted.\", \"priority\": 6 }) }}"
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1760, 440],
      "id": "dddddddd-dddd-dddd-dddd-dddddddddddd",
      "name": "Notify IP Failure"
    },

    {
      "parameters": {},
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [1980, 440],
      "id": "eeeeeeee-eeee-eeee-eeee-eeeeeeeeeeee",
      "name": "No Operation, do nothing3"
    },

    {
      "parameters": {
        "method": "GET",
        "url": "=http://scraper-service:8000/?sessionId={{ $json.sessionId }}",
        "options": {
          "response": {
            "response": {
              "responseFormat": "json"
            }
          }
        }
      },
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1540, 160],
      "id": "ffffffff-ffff-ffff-ffff-ffffffffffff",
      "name": "Call Scraper Service"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\n\ndata = _items[0]['json']\n\nstatus = data.get('status', 'UNKNOWN')\nsession_id = data.get('sessionId', 'unknown')\nposts_found = data.get('postsFound', 0)\ndetection_event = data.get('detectionEvent', None)\nraw_file = data.get('rawFile', '')\nquery_used = data.get('query', '')\nerror = data.get('error', '')\n\ncan_process = status in ['COMPLETED', 'PARTIAL'] and posts_found > 0\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'scraperStatus': status,\n            'postsFound': posts_found,\n            'detectionEvent': detection_event,\n            'rawFile': raw_file,\n            'queryUsed': query_used,\n            'canProcess': can_process,\n            'scraperError': error\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1760, 160],
      "id": "11111111-aaaa-1111-aaaa-111111111111",
      "name": "Parse Scraper Output"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "id": "dd111111-1111-1111-1111-111111111111",
              "leftValue": "={{ $json.canProcess }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [1980, 160],
      "id": "22222222-aaaa-2222-aaaa-222222222222",
      "name": "Posts Found?"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nimport sqlite3\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\nstatus = _items[0]['json'].get('scraperStatus', 'FAILED')\ndetection_event = _items[0]['json'].get('detectionEvent', None)\n\ntry:\n    conn = sqlite3.connect('/data/linkedin_intel/leads.db')\n    conn.execute(\n        \"UPDATE sessions SET status='FAILED', ended_at=?, detection_event=? WHERE session_id=?\",\n        (datetime.utcnow().isoformat(), str(detection_event), session_id)\n    )\n    conn.commit()\n    conn.close()\nexcept Exception:\n    pass\n\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SCRAPER_NO_POSTS',\n        'sessionId': session_id,\n        'scraperStatus': status,\n        'detectionEvent': str(detection_event),\n        'timestamp': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'failed': True,\n            'reason': status,\n            'detectionEvent': str(detection_event),\n            'leadsStored': 0,\n            'rawFile': _items[0]['json'].get('rawFile', '')\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2200, 300],
      "id": "33333333-aaaa-3333-aaaa-333333333333",
      "name": "Log Scraper No Posts"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nimport re\nimport sqlite3\nimport hashlib\nfrom datetime import datetime\nfrom pathlib import Path\n\nsession_id = _items[0]['json']['sessionId']\nraw_file = _items[0]['json'].get('rawFile', '')\n\nYOUR_STACK = {\n    'python', 'django', 'fastapi', 'flask',\n    'postgresql', 'postgres', 'redis', 'docker',\n    'kubernetes', 'aws', 'gcp', 'react',\n    'typescript', 'golang', 'kafka', 'terraform'\n}\n\nYOUR_ROLES = {\n    'senior software engineer',\n    'backend engineer',\n    'senior backend engineer',\n    'platform engineer',\n    'staff engineer',\n    'senior developer',\n    'software architect',\n    'devops engineer',\n    'site reliability engineer',\n    'sre'\n}\n\nYOUR_LEVEL = 'senior'\nPREFER_REMOTE = True\nHIGH_PRIORITY_THRESHOLD = 7\nMIN_SCORE_TO_STORE = 2\n\nBAIT = [\n    (r'comment\\s+[\"\\']?yes[\"\\']?', -3),\n    (r'type\\s+[\"\\']?yes[\"\\']?\\s+if', -3),\n    (r'drop\\s+a\\s+[\"\\']?yes[\"\\']?', -3),\n    (r'post\\s+your\\s+jobs?\\s+in\\s+the\\s+comments', -3),\n    (r'who\\s+needs?\\s+to\\s+hear\\s+this', -2),\n    (r'not\\s+sure\\s+who\\s+needs?\\s+to\\s+hear', -2),\n    (r'tag\\s+someone\\s+who', -2),\n    (r'repost\\s+to\\s+help', -1),\n    (r'share\\s+this\\s+with\\s+your\\s+network', -1)\n]\n\nREAL = [\n    (r'\\d+\\s*(?:years?|yrs?)\\s+(?:of\\s+)?experience', 2),\n    (r'(?:remote|hybrid|onsite|on-site)', 1),\n    (r'(?:salary|compensation).*\\$[\\d,]+', 2),\n    (r\"i'?m\\s+personally\\s+reviewing\", 2),\n    (r'send\\s+(?:me|your)\\s+(?:resume|cv)', 2),\n    (r'(?:immediately|asap|urgent)', 1),\n    (r'(?:full.?time|contract|freelance)', 1)\n]\n\ndef extract_emails(text):\n    patterns = [\n        r'[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}',\n        r'(?i)email\\s*(?:me\\s*at|:)\\s*([a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,})',\n        r'(?i)reach\\s+(?:me|out)\\s+at\\s+([a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,})',\n        r'(?i)contact\\s+me\\s+at\\s*:?\\s*([a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,})'\n    ]\n    found = set()\n    for p in patterns:\n        for m in re.findall(p, text):\n            e = (m if isinstance(m, str) else m).strip().rstrip('.,;:)')\n            if '@' in e and '.' in e.split('@')[-1]:\n                if not any(x in e for x in ['example.com', 'test@', 'youremail']):\n                    found.add(e.lower())\n    return list(found)\n\ndef get_bait_score(text):\n    score = 0\n    for pattern, weight in BAIT:\n        if re.search(pattern, text, re.I):\n            score += weight\n    return score\n\ndef enrich_and_score(post):\n    text = post.get('text', '').lower()\n    score = 0\n    found_stack = []\n    for tech in YOUR_STACK:\n        if re.search(r'\\b' + re.escape(tech) + r'\\b', text, re.I):\n            found_stack.append(tech)\n    score += len(found_stack)\n    post['tech_stack'] = found_stack\n    role_found = ''\n    for role in YOUR_ROLES:\n        if role in text:\n            role_found = role.title()\n            score += 3\n            break\n    if not role_found:\n        for role in YOUR_ROLES:\n            words = role.split()\n            if any(w in text for w in words if len(w) > 4):\n                role_found = role.title()\n                score += 1\n                break\n    post['role_title'] = role_found\n    exp = 'unknown'\n    if re.search(r'(?:senior|sr\\.?|5\\+\\s*years?|6\\+\\s*years?)', text, re.I):\n        exp = 'senior'\n        if YOUR_LEVEL == 'senior':\n            score += 2\n    elif re.search(r'(?:lead|principal|staff|8\\+\\s*years?)', text, re.I):\n        exp = 'lead'\n        score += 1\n    elif re.search(r'(?:junior|entry|0-2\\s*years?)', text, re.I):\n        exp = 'entry'\n    elif re.search(r'(?:mid.?level|2-4\\s*years?|3-5\\s*years?)', text, re.I):\n        exp = 'mid'\n    post['experience_level'] = exp\n    remote = 'unknown'\n    if re.search(r'\\bfully\\s+remote\\b|\\b100%\\s+remote\\b', text, re.I):\n        remote = 'remote'\n        if PREFER_REMOTE:\n            score += 2\n    elif re.search(r'\\bremote\\b', text, re.I):\n        remote = 'remote'\n        if PREFER_REMOTE:\n            score += 1\n    elif re.search(r'\\bhybrid\\b', text, re.I):\n        remote = 'hybrid'\n        if PREFER_REMOTE:\n            score += 1\n    elif re.search(r'\\bonsite\\b|\\bon-site\\b', text, re.I):\n        remote = 'onsite'\n    post['remote_signal'] = remote\n    loc_match = re.search(\n        r'\\b(?:New York|San Francisco|London|Austin|Seattle|Chicago|Boston|Los Angeles|Berlin|Toronto|Vancouver|Remote|Worldwide|Anywhere)\\b',\n        post.get('text', ''), re.I\n    )\n    post['location'] = loc_match.group(0) if loc_match else ''\n    sal_match = re.search(\n        r'\\$[\\d,]+(?:k|K)?(?:\\s*[-]\\s*\\$[\\d,]+(?:k|K)?)?',\n        post.get('text', '')\n    )\n    post['salary'] = sal_match.group(0) if sal_match else ''\n    for pattern, weight in REAL:\n        if re.search(pattern, text, re.I):\n            score += weight\n    return score\n\nif not raw_file or not Path(raw_file).exists():\n    return [\n        {\n            'json': {\n                'sessionId': session_id,\n                'leadsStored': 0,\n                'highPriorityLeads': [],\n                'highPriorityCount': 0,\n                'totalProcessed': 0,\n                'rawFile': raw_file,\n                'error': 'Raw file not found'\n            }\n        }\n    ]\n\ntry:\n    posts = json.loads(Path(raw_file).read_text())\nexcept Exception as e:\n    return [\n        {\n            'json': {\n                'sessionId': session_id,\n                'leadsStored': 0,\n                'highPriorityLeads': [],\n                'highPriorityCount': 0,\n                'totalProcessed': 0,\n                'rawFile': raw_file,\n                'error': f'Parse error: {str(e)}'\n            }\n        }\n    ]\n\ndb_path = '/data/linkedin_intel/leads.db'\nconn = sqlite3.connect(db_path)\nconn.execute('PRAGMA journal_mode=WAL')\nnow = datetime.utcnow().isoformat()\nleads_stored = 0\nhigh_priority = []\n\nfor post in posts:\n    text = post.get('text', '')\n    if not text:\n        continue\n    emails = extract_emails(text)\n    if not emails:\n        continue\n    bait_sc = get_bait_score(text)\n    if bait_sc < -2:\n        continue\n    rel_sc = enrich_and_score(post)\n    if rel_sc < MIN_SCORE_TO_STORE:\n        continue\n    author_url = post.get('author_url', '')\n    found_at = post.get('found_at', now)\n    lead_id = hashlib.sha256(\n        f'{author_url}::{found_at}'.encode()\n    ).hexdigest()\n    row = conn.execute(\n        'SELECT id FROM leads WHERE id = ?', (lead_id,)\n    ).fetchone()\n    if row:\n        continue\n    dup_email = False\n    for email in emails:\n        email_hash = hashlib.sha256(email.encode()).hexdigest()\n        row = conn.execute(\n            'SELECT email_hash FROM email_seen WHERE email_hash = ?',\n            (email_hash,)\n        ).fetchone()\n        if row:\n            dup_email = True\n            break\n    if dup_email:\n        continue\n    try:\n        conn.execute(\"\"\"\n            INSERT INTO leads (\n                id, author_name, author_url, post_text,\n                emails, hashtags, tech_stack, role_title,\n                experience_level, remote_signal, location,\n                salary, relevance_score, bait_score,\n                status, session_id, created_at\n            ) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)\n        \"\"\", (\n            lead_id,\n            post.get('author_name', ''),\n            author_url,\n            text,\n            json.dumps(emails),\n            json.dumps(post.get('hashtags', [])),\n            json.dumps(post.get('tech_stack', [])),\n            post.get('role_title', ''),\n            post.get('experience_level', ''),\n            post.get('remote_signal', ''),\n            post.get('location', ''),\n            post.get('salary', ''),\n            rel_sc,\n            bait_sc,\n            'NEW',\n            session_id,\n            now\n        ))\n        for email in emails:\n            email_hash = hashlib.sha256(email.encode()).hexdigest()\n            conn.execute(\n                'INSERT OR IGNORE INTO email_seen (email_hash, email, first_seen) VALUES (?,?,?)',\n                (email_hash, email, now)\n            )\n        conn.commit()\n        leads_stored += 1\n        if rel_sc >= HIGH_PRIORITY_THRESHOLD:\n            high_priority.append({\n                'role_title': post.get('role_title', 'Unknown Role'),\n                'author_name': post.get('author_name', ''),\n                'emails': emails,\n                'tech_stack': post.get('tech_stack', []),\n                'remote_signal': post.get('remote_signal', ''),\n                'relevance_score': rel_sc,\n                'salary': post.get('salary', '')\n            })\n    except Exception:\n        conn.rollback()\n        continue\n\nconn.execute(\n    'UPDATE sessions SET posts_scanned=?, leads_stored=? WHERE session_id=?',\n    (len(posts), leads_stored, session_id)\n)\nconn.commit()\nconn.close()\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'leadsStored': leads_stored,\n            'highPriorityLeads': high_priority,\n            'highPriorityCount': len(high_priority),\n            'totalProcessed': len(posts),\n            'rawFile': raw_file\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2200, 60],
      "id": "44444444-aaaa-4444-aaaa-444444444444",
      "name": "Extract + Score + Save Leads"
    },

    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 3
          },
          "conditions": [
            {
              "id": "ee111111-1111-1111-1111-111111111111",
              "leftValue": "={{ $json.highPriorityCount }}",
              "rightValue": 0,
              "operator": {
                "type": "number",
                "operation": "gt"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [2420, 60],
      "id": "55555555-aaaa-5555-aaaa-555555555555",
      "name": "High Priority Found?"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\n\nleads = _items[0]['json'].get('highPriorityLeads', [])\nsession_id = _items[0]['json'].get('sessionId', 'unknown')\ntotal = _items[0]['json'].get('leadsStored', 0)\nraw_file = _items[0]['json'].get('rawFile', '')\n\nlines = []\nfor lead in leads[:5]:\n    role = lead.get('role_title', 'Unknown Role')\n    emails = lead.get('emails', [])\n    score = lead.get('relevance_score', 0)\n    remote = lead.get('remote_signal', '')\n    stack = ', '.join(lead.get('tech_stack', [])[:3])\n    email_str = emails[0] if emails else 'no email'\n    lines.append(\n        f'{role} | Score:{score} | {email_str} | {remote} | {stack}'\n    )\n\nmessage = (\n    f'Session: {session_id}\\nTotal stored: {total}\\n\\n'\n    + '\\n'.join(lines)\n)\n\nif len(leads) > 5:\n    message += f'\\n\\n...and {len(leads) - 5} more. Check dashboard.'\n\nreturn [\n    {\n        'json': {\n            'notificationTitle': f'{len(leads)} High Priority Lead(s) Found',\n            'notificationMessage': message,\n            'notificationPriority': 8,\n            'sessionId': session_id,\n            'leadsStored': total,\n            'highPriorityCount': len(leads),\n            'highPriorityLeads': leads,\n            'rawFile': raw_file\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2640, -80],
      "id": "66666666-aaaa-6666-aaaa-666666666666",
      "name": "Build Notification Message"
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
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [2860, -80],
      "id": "77777777-aaaa-7777-aaaa-777777777777",
      "name": "Send Phone Notification"
    },

    {
      "parameters": {
        "language": "pythonNative",
        "pythonCode": "import json\nimport sqlite3\nfrom datetime import datetime, timedelta\nfrom pathlib import Path\n\ninput_data = _items[0]['json'] if _items else {}\nsession_id = input_data.get('sessionId', 'unknown')\ndetection_event = input_data.get('detectionEvent', None)\nleads_stored = input_data.get('leadsStored', 0)\nraw_file = input_data.get('rawFile', '')\n\nif detection_event and str(detection_event) not in ['None', '']:\n    freeze_hours = 48 if 'HARD' in str(detection_event) else 6\n    resume_time = (\n        datetime.utcnow() + timedelta(hours=freeze_hours)\n    ).isoformat()\n    freeze_file = Path('/data/linkedin_intel/state/freeze.json')\n    freeze_file.parent.mkdir(parents=True, exist_ok=True)\n    freeze_file.write_text(json.dumps({\n        'frozen': True,\n        'reason': str(detection_event),\n        'frozenAt': datetime.utcnow().isoformat(),\n        'resumeAfter': resume_time,\n        'freezeHours': freeze_hours\n    }))\n\nif raw_file:\n    src = Path(raw_file)\n    if src.exists():\n        archive_dir = Path('/data/linkedin_intel/raw_posts/archive')\n        archive_dir.mkdir(parents=True, exist_ok=True)\n        src.rename(archive_dir / src.name)\n\ntry:\n    conn = sqlite3.connect('/data/linkedin_intel/leads.db')\n    conn.execute(\n        \"UPDATE sessions SET status='COMPLETED', ended_at=?, leads_stored=? WHERE session_id=?\",\n        (datetime.utcnow().isoformat(), leads_stored, session_id)\n    )\n    conn.commit()\n    conn.close()\nexcept Exception:\n    pass\n\nlog_file = Path('/data/linkedin_intel/logs/sessions.log')\nlog_file.parent.mkdir(parents=True, exist_ok=True)\nwith open(log_file, 'a') as f:\n    f.write(json.dumps({\n        'event': 'SESSION_COMPLETE',\n        'sessionId': session_id,\n        'leadsStored': leads_stored,\n        'detectionEvent': str(detection_event),\n        'completedAt': datetime.utcnow().isoformat()\n    }) + '\\n')\n\nreturn [\n    {\n        'json': {\n            'sessionId': session_id,\n            'sessionComplete': True,\n            'leadsStored': leads_stored,\n            'detectionHandled': bool(\n                detection_event\n                and str(detection_event) not in ['None', '']\n            ),\n            'completedAt': datetime.utcnow().isoformat()\n        }\n    }\n]"
      },
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [3080, 60],
      "id": "88888888-aaaa-8888-aaaa-888888888888",
      "name": "Finalize Session"
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
        [{ "node": "No Operation, do nothing1", "type": "main", "index": 0 }]
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
        [{ "node": "No Operation, do nothing2", "type": "main", "index": 0 }]
      ]
    },
    "Rotate Tor IP": {
      "main": [
        [{ "node": "IP Rotation OK?", "type": "main", "index": 0 }]
      ]
    },
    "IP Rotation OK?": {
      "main": [
        [{ "node": "Log IP Failure", "type": "main", "index": 0 }],
        [{ "node": "Call Scraper Service", "type": "main", "index": 0 }]
      ]
    },
    "Log IP Failure": {
      "main": [
        [{ "node": "Notify IP Failure", "type": "main", "index": 0 }]
      ]
    },
    "Notify IP Failure": {
      "main": [
        [{ "node": "No Operation, do nothing3", "type": "main", "index": 0 }]
      ]
    },
    "Call Scraper Service": {
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
        [{ "node": "Log Scraper No Posts", "type": "main", "index": 0 }],
        [{ "node": "Extract + Score + Save Leads", "type": "main", "index": 0 }]
      ]
    },
    "Log Scraper No Posts": {
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

  "pinData": {},
  "meta": {
    "instanceId": "558d88703fb65b2d0e44613bc35916258b0f0bf983c5d4730c00c424b77ca36a"
  }
}
```

---

## Complete scraper.py For Your Python Container

```python
# /home/dev/scripts/scraper.py
# Runs as HTTP service inside scraper-service container
# Receives sessionId via GET parameter
# Returns JSON response to n8n

from http.server import BaseHTTPRequestHandler, HTTPServer
import json
import urllib.parse
import time
import random
import re
import hashlib
import pickle
import traceback
from datetime import datetime
from pathlib import Path

# ── Search query pool ──────────────────────────────────────────────
QUERIES = [
    '"email" #hiring #wearehiring',
    '"DM me or email" #nowhiring',
    '"contact me" #recruiting #jobopening',
    '"reach me at" #hiring #techjobs',
    '"email me" #wearehiring #remotejobs',
    '"send me your resume" #hiring',
    '"email your CV" #nowhiring',
    '"drop me an email" #hiring',
    '"email:" #wearehiring',
    '"reach out" "email" #recruiting',
]

# ── Email extraction patterns ──────────────────────────────────────
EMAIL_PATTERNS = [
    r'[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}',
    r'(?i)email\s*(?:me\s*at|:)\s*([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})',
    r'(?i)reach\s+(?:me|out)\s+at\s+([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})',
    r'(?i)contact\s+me\s+at\s*:?\s*([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})',
]

# ── Paths ──────────────────────────────────────────────────────────
COOKIE_FILE = Path('/data/linkedin_intel/cookies/session.pkl')
RAW_POSTS_DIR = Path('/data/linkedin_intel/raw_posts')
STATE_DIR = Path('/data/linkedin_intel/state')
LOG_FILE = Path('/data/linkedin_intel/logs/scraper.log')

# ── Settings ───────────────────────────────────────────────────────
TOR_HOST = '127.0.0.1'
TOR_PORT = 9050
MAX_POSTS = random.randint(40, 60)
MAX_SCROLLS = random.randint(15, 25)
MAX_DURATION_SECONDS = 2100

def log(message):
    LOG_FILE.parent.mkdir(parents=True, exist_ok=True)
    timestamp = datetime.utcnow().isoformat()
    with open(LOG_FILE, 'a') as f:
        f.write(f'[{timestamp}] {message}\n')

def get_query(session_id):
    idx = int(time.time() / 3600) % len(QUERIES)
    return QUERIES[idx]

def extract_emails(text):
    found = set()
    for p in EMAIL_PATTERNS:
        for m in re.findall(p, text):
            e = (m if isinstance(m, str) else m).strip().rstrip('.,;:)')
            if '@' in e and '.' in e.split('@')[-1]:
                if not any(x in e for x in ['example.com', 'test@']):
                    found.add(e.lower())
    return list(found)

def human_wait(mean, sigma=0.5):
    t = random.gauss(mean, sigma)
    t = max(0.5, min(t, 20.0))
    time.sleep(t)

def human_scroll(driver, pixels):
    import math
    steps = random.randint(4, 8)
    for i in range(steps):
        chunk = int((pixels / steps) * math.sin((i / steps) * math.pi))
        chunk += random.randint(-8, 8)
        driver.execute_script(f'window.scrollBy(0, {chunk})')
        time.sleep(random.gauss(0.08, 0.02))

def inject_fingerprint(driver):
    driver.execute_script("""
        Object.defineProperty(navigator, 'webdriver', {
            get: () => undefined, configurable: true
        });
        Object.defineProperty(navigator, 'plugins', {
            get: () => [1, 2, 3], configurable: true
        });
        window.chrome = { runtime: {} };
        const orig = HTMLCanvasElement.prototype.toDataURL;
        HTMLCanvasElement.prototype.toDataURL = function(type) {
            const r = orig.apply(this, arguments);
            return r.slice(0, -4) + Math.floor(Math.random() * 9);
        };
    """)

def check_health(driver):
    url = driver.current_url
    src = driver.page_source[:300].lower()
    if any(x in url for x in ['checkpoint', 'challenge', 'restricted']):
        return 'HARD_SIGNAL'
    if any(x in src for x in ['captcha', 'unusual activity', 'verify']):
        return 'MEDIUM_SIGNAL'
    if 'authwall' in url or ('login' in url and 'feed' not in url):
        return 'MEDIUM_SIGNAL'
    return 'HEALTHY'

def run_scraper(session_id):
    log(f'Session starting: {session_id}')
    session_start = datetime.utcnow()

    try:
        import undetected_chromedriver as uc
        from bs4 import BeautifulSoup
        from selenium.webdriver.common.by import By
    except ImportError as e:
        return {
            'status': 'FAILED',
            'sessionId': session_id,
            'postsFound': 0,
            'detectionEvent': None,
            'rawFile': '',
            'query': '',
            'error': f'Import error: {str(e)}'
        }

    options = uc.ChromeOptions()
    options.add_argument('--window-size=1366,768')
    options.add_argument(f'--proxy-server=socks5://{TOR_HOST}:{TOR_PORT}')
    options.add_argument('--no-sandbox')
    options.add_argument('--disable-dev-shm-usage')
    options.add_argument('--disable-blink-features=AutomationControlled')
    options.add_argument('--user-data-dir=/data/linkedin_intel/chrome_profile')

    driver = None
    posts_found = []
    detection_event = None

    try:
        driver = uc.Chrome(
            options=options,
            headless=False,
            use_subprocess=True
        )
        inject_fingerprint(driver)

        # Load cookies
        driver.get('https://www.linkedin.com')
        human_wait(2.0)
        if COOKIE_FILE.exists():
            with open(COOKIE_FILE, 'rb') as f:
                cookies = pickle.load(f)
            for cookie in cookies:
                try:
                    cookie.pop('expiry', None)
                    driver.add_cookie(cookie)
                except Exception:
                    pass

        # Feed warmup
        driver.get('https://www.linkedin.com/feed/')
        human_wait(4.0, 1.0)

        health = check_health(driver)
        if health != 'HEALTHY':
            log(f'Detection on feed load: {health}')
            return {
                'status': 'ABORTED',
                'sessionId': session_id,
                'postsFound': 0,
                'detectionEvent': health,
                'rawFile': '',
                'query': ''
            }

        for _ in range(random.randint(3, 6)):
            human_scroll(driver, random.randint(300, 500))
            human_wait(random.gauss(3.0, 1.0))

        # Search
        query = get_query(session_id)
        import urllib.parse as up
        encoded = up.quote(query)
        search_url = (
            f'https://www.linkedin.com/search/results/content/'
            f'?keywords={encoded}&sortBy=date_posted'
        )
        driver.get(search_url)
        human_wait(4.0, 1.0)

        health = check_health(driver)
        if health != 'HEALTHY':
            detection_event = health
            log(f'Detection on search: {health}')
        else:
            seen_ids = set()
            for scroll_num in range(MAX_SCROLLS):
                elapsed = (datetime.utcnow() - session_start).total_seconds()
                if elapsed >= MAX_DURATION_SECONDS:
                    log('Session time limit reached')
                    break
                if len(posts_found) >= MAX_POSTS:
                    break

                if scroll_num > 0 and scroll_num % 5 == 0:
                    health = check_health(driver)
                    if health != 'HEALTHY':
                        detection_event = health
                        log(f'Detection during scroll {scroll_num}: {health}')
                        break

                try:
                    containers = driver.find_elements(
                        By.CSS_SELECTOR,
                        'div[data-urn], div.feed-shared-update-v2'
                    )
                except Exception:
                    containers = []

                for container in containers:
                    try:
                        raw_html = container.get_attribute('outerHTML')
                        post_id = hashlib.md5(
                            raw_html[:300].encode()
                        ).hexdigest()

                        if post_id in seen_ids:
                            continue
                        seen_ids.add(post_id)

                        soup = BeautifulSoup(raw_html, 'html.parser')
                        text_nodes = soup.find_all(
                            'span', attrs={'dir': 'ltr'}
                        )
                        text = ' '.join(
                            n.get_text(' ', strip=True)
                            for n in text_nodes
                        )

                        if not text:
                            continue

                        emails = extract_emails(text)
                        if not emails:
                            continue

                        author_name = ''
                        author_url = ''
                        link = soup.find(
                            'a', href=re.compile(r'/in/[^/?]+')
                        )
                        if link:
                            author_name = link.get_text(strip=True)
                            href = link.get('href', '')
                            m = re.search(r'(/in/[^/?]+)', href)
                            if m:
                                author_url = f'https://www.linkedin.com{m.group(1)}'

                        hashtags = [
                            t.get_text(strip=True)
                            for t in soup.find_all(
                                'a',
                                href=re.compile(r'/feed/hashtag/')
                            )
                        ]

                        posts_found.append({
                            'post_id': post_id,
                            'text': text,
                            'emails': emails,
                            'author_name': author_name,
                            'author_url': author_url,
                            'hashtags': hashtags,
                            'found_at': datetime.utcnow().isoformat(),
                            'query_used': query
                        })

                        words = len(text.split())
                        dwell = min((words / 238) * 60 * 0.7, 10.0)
                        time.sleep(max(1.5, dwell))

                    except Exception:
                        continue

                human_scroll(driver, random.randint(300, 500))
                human_wait(random.gauss(2.5, 0.8))

        # Save cookies
        COOKIE_FILE.parent.mkdir(parents=True, exist_ok=True)
        with open(COOKIE_FILE, 'wb') as f:
            pickle.dump(driver.get_cookies(), f)

        # Write raw output
        RAW_POSTS_DIR.mkdir(parents=True, exist_ok=True)
        raw_file = str(RAW_POSTS_DIR / f'{session_id}.json')
        with open(raw_file, 'w') as f:
            json.dump(posts_found, f, indent=2)

        log(f'Session complete: {len(posts_found)} posts found')

        return {
            'status': 'COMPLETED' if not detection_event else 'PARTIAL',
            'sessionId': session_id,
            'postsFound': len(posts_found),
            'detectionEvent': detection_event,
            'rawFile': raw_file,
            'query': query
        }

    except Exception as e:
        log(f'Scraper error: {traceback.format_exc()}')
        return {
            'status': 'FAILED',
            'sessionId': session_id,
            'postsFound': len(posts_found),
            'detectionEvent': None,
            'rawFile': '',
            'query': '',
            'error': str(e)
        }

    finally:
        if driver:
            try:
                driver.quit()
            except Exception:
                pass


class ScraperHandler(BaseHTTPRequestHandler):

    def log_message(self, format, *args):
        # Suppress default HTTP server logs
        # Use our own log function instead
        pass

    def do_GET(self):
        try:
            parsed = urllib.parse.urlparse(self.path)
            params = urllib.parse.parse_qs(parsed.query)
            session_id = params.get('sessionId', ['unknown'])[0]

            log(f'HTTP request received: sessionId={session_id}')

            # Health check endpoint
            if parsed.path == '/health':
                response = {'status': 'ok', 'service': 'scraper-service'}
                self._send_json(200, response)
                return

            # Main scrape endpoint
            result = run_scraper(session_id)
            self._send_json(200, result)

        except Exception as e:
            error_response = {
                'status': 'FAILED',
                'sessionId': 'unknown',
                'postsFound': 0,
                'detectionEvent': None,
                'rawFile': '',
                'query': '',
                'error': str(e)
            }
            self._send_json(500, error_response)

    def _send_json(self, code, data):
        body = json.dumps(data).encode()
        self.send_response(code)
        self.send_header('Content-Type', 'application/json')
        self.send_header('Content-Length', str(len(body)))
        self.end_headers()
        self.wfile.write(body)


if __name__ == '__main__':
    port = 8000
    log(f'Scraper service starting on port {port}')
    server = HTTPServer(('0.0.0.0', port), ScraperHandler)
    log(f'Scraper service ready')
    server.serve_forever()
```

---

## Updated Docker Compose Service Block

```yaml
scraper-service:
  image: python:3.11-slim
  restart: always
  working_dir: /app
  volumes:
    - '/home/dev/scripts:/app'
    - '/data/linkedin_intel:/data/linkedin_intel'
  command: >
    sh -c "pip install undetected-chromedriver
    selenium beautifulsoup4 requests stem
    && python3 scraper.py"
  networks:
    - coolify
```

---

## Three Values To Replace In Workflow

```
1. YOUR_TOR_PASSWORD_HERE
   Node: Rotate Tor IP
   Replace with your actual Tor password

2. YOUR_GOTIFY_TOKEN_HERE
   Node: Notify IP Failure
   Node: Send Phone Notification
   Replace with your Gotify app token

3. Verify scraper-service URL
   Node: Call Scraper Service
   Current: http://scraper-service:8000/?sessionId=...
   Change service name if yours is different
   Test with:
   docker exec -it n8n wget -qO-
   "http://scraper-service:8000/health"
   Should return: {"status": "ok"}
```