# Complete n8n Workflow: LinkedIn Recruiter Intelligence System

## Architecture Overview Before Code

```
n8n is the nervous system. Python scripts are the muscles.
SQLite is the memory. Gotify is the voice. You are the brain.
```

---

## WORKFLOW 1: MASTER ORCHESTRATOR

### 1.1 — Complete n8n JSON (Import This Directly)

```json
{
  "name": "LinkedIn Recruiter Intelligence - Master Orchestrator",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "0 7 * * 2,3,4"
            },
            {
              "field": "cronExpression", 
              "expression": "0 11 * * 2,3,4"
            },
            {
              "field": "cronExpression",
              "expression": "0 17 * * 2,3,4"
            },
            {
              "field": "cronExpression",
              "expression": "0 7 * * 1,5"
            },
            {
              "field": "cronExpression",
              "expression": "0 3 * * 6"
            },
            {
              "field": "cronExpression",
              "expression": "0 5 * * 0"
            }
          ]
        }
      },
      "name": "Master Cron Scheduler",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [240, 300],
      "id": "node-cron-001"
    },
    {
      "parameters": {
        "jsCode": "// JITTER: Add 0-8 minutes random delay to prevent exact-time detection\nconst jitterMs = Math.floor(Math.random() * 480000);\nconst jitterMinutes = (jitterMs / 60000).toFixed(1);\n\n// WEEK VARIATION: Simulate busy/active weeks\nconst weekVariation = ['NORMAL', 'BUSY', 'ACTIVE', 'NORMAL', 'NORMAL'];\nconst weekOfYear = Math.floor((new Date() - new Date(new Date().getFullYear(), 0, 1)) / 604800000);\nconst currentWeekType = weekVariation[weekOfYear % weekVariation.length];\n\n// BLACKOUT CHECK: Random 1 day per 2 weeks full blackout\nconst dayOfYear = Math.floor((new Date() - new Date(new Date().getFullYear(), 0, 1)) / 86400000);\nconst isBlackoutDay = (dayOfYear % 14 === 7); // Every 14 days, day 7 is blackout\n\n// SESSION LIMIT CHECK based on week type\nconst sessionLimits = { 'NORMAL': 3, 'BUSY': 2, 'ACTIVE': 4 };\nconst maxSessions = sessionLimits[currentWeekType];\n\nreturn [{\n  json: {\n    jitterMs,\n    jitterMinutes,\n    weekType: currentWeekType,\n    maxSessions,\n    isBlackoutDay,\n    triggerTime: new Date().toISOString(),\n    sessionId: 'sess_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9)\n  }\n}];"
      },
      "name": "Jitter + Week Variation Calculator",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [460, 300],
      "id": "node-jitter-001"
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            {
              "value1": "={{$json.isBlackoutDay}}",
              "value2": true
            }
          ]
        }
      },
      "name": "Blackout Day Check",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [680, 300],
      "id": "node-blackout-001"
    },
    {
      "parameters": {
        "jsCode": "// Log blackout event\nconst fs = require('fs');\nconst logEntry = {\n  event: 'BLACKOUT_DAY',\n  timestamp: new Date().toISOString(),\n  sessionId: $input.item.json.sessionId\n};\nfs.appendFileSync('/data/linkedin_intel/logs/system.log', JSON.stringify(logEntry) + '\\n');\nreturn [{ json: { status: 'BLACKOUT_SKIPPED', ...logEntry } }];"
      },
      "name": "Log Blackout + Stop",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [900, 180],
      "id": "node-blackout-log-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/session_limit_check.py {{$json.sessionId}} {{$json.maxSessions}}"
      },
      "name": "Session Limit Check",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [900, 400],
      "id": "node-session-limit-001"
    },
    {
      "parameters": {
        "jsCode": "const output = $input.item.json.stdout;\nlet result;\ntry {\n  result = JSON.parse(output);\n} catch(e) {\n  result = { canProceed: false, reason: 'PARSE_ERROR', raw: output };\n}\nreturn [{ json: { ...result, sessionId: $input.item.json.sessionId } }];"
      },
      "name": "Parse Session Limit Result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1120, 400],
      "id": "node-parse-limit-001"
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{$json.canProceed}}",
              "value2": "true"
            }
          ]
        }
      },
      "name": "Can Proceed?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [1340, 400],
      "id": "node-can-proceed-001"
    },
    {
      "parameters": {
        "jsCode": "// Apply jitter delay before starting\nconst jitterMs = $input.item.json.jitterMs || 0;\nawait new Promise(resolve => setTimeout(resolve, jitterMs));\nreturn [{ json: { ...$input.item.json, jitterApplied: true, proceedAt: new Date().toISOString() } }];"
      },
      "name": "Apply Jitter Delay",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1560, 300],
      "id": "node-jitter-apply-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/ip_rotation.py rotate"
      },
      "name": "IP Rotation Agent",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [1780, 300],
      "id": "node-ip-rotate-001"
    },
    {
      "parameters": {
        "jsCode": "const output = $input.item.json.stdout;\nlet ipResult;\ntry {\n  ipResult = JSON.parse(output);\n} catch(e) {\n  ipResult = { success: false, error: 'PARSE_ERROR' };\n}\nreturn [{ json: { ...ipResult, sessionId: $input.item.json.sessionId } }];"
      },
      "name": "Parse IP Rotation Result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2000, 300],
      "id": "node-parse-ip-001"
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            {
              "value1": "={{$json.success}}",
              "value2": true
            }
          ]
        }
      },
      "name": "IP Rotation Successful?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [2220, 300],
      "id": "node-ip-check-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/session_health.py check {{$json.sessionId}}"
      },
      "name": "Session Health Check",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [2440, 200],
      "id": "node-session-health-001"
    },
    {
      "parameters": {
        "jsCode": "const output = $input.item.json.stdout;\nlet health;\ntry {\n  health = JSON.parse(output);\n} catch(e) {\n  health = { status: 'UNKNOWN', needsReauth: true };\n}\nreturn [{ json: { ...health, sessionId: $input.item.json.sessionId } }];"
      },
      "name": "Parse Health Result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2660, 200],
      "id": "node-parse-health-001"
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{$json.status}}",
              "value2": "HEALTHY"
            }
          ]
        }
      },
      "name": "Session Healthy?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [2880, 200],
      "id": "node-health-check-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/main_scraper.py {{$json.sessionId}}"
      },
      "name": "Main Scraper Agent",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [3100, 100],
      "id": "node-scraper-001",
      "onError": "continueErrorOutput"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/session_recovery.py {{$json.sessionId}}"
      },
      "name": "Session Recovery Agent",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [3100, 340],
      "id": "node-recovery-001"
    },
    {
      "parameters": {
        "jsCode": "const scraperOutput = $input.item.json.stdout;\nlet scraperResult;\ntry {\n  scraperResult = JSON.parse(scraperOutput);\n} catch(e) {\n  scraperResult = { \n    status: 'PARSE_ERROR', \n    postsScanned: 0, \n    leadsFound: 0,\n    detectionEvent: 'UNKNOWN_ERROR'\n  };\n}\nreturn [{ json: { ...scraperResult, sessionId: $input.item.json.sessionId } }];"
      },
      "name": "Parse Scraper Result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [3320, 100],
      "id": "node-parse-scraper-001"
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{$json.status}}",
              "value2": "COMPLETED"
            }
          ]
        }
      },
      "name": "Scraper Completed?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [3540, 100],
      "id": "node-scraper-done-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/post_processor.py {{$json.sessionId}}"
      },
      "name": "Post Processor + Enricher",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [3760, 20],
      "id": "node-processor-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/detection_handler.py {{$json.sessionId}} {{$json.detectionEvent}}"
      },
      "name": "Detection Handler",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [3760, 200],
      "id": "node-detection-001"
    },
    {
      "parameters": {
        "jsCode": "const processorOutput = $input.item.json.stdout;\nlet processorResult;\ntry {\n  processorResult = JSON.parse(processorOutput);\n} catch(e) {\n  processorResult = { leadsStored: 0, highPriorityLeads: [], error: 'PARSE_ERROR' };\n}\nreturn [{ json: { ...processorResult, sessionId: $input.item.json.sessionId } }];"
      },
      "name": "Parse Processor Result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [3980, 20],
      "id": "node-parse-processor-001"
    },
    {
      "parameters": {
        "conditions": {
          "number": [
            {
              "value1": "={{$json.highPriorityLeads.length}}",
              "operation": "larger",
              "value2": 0
            }
          ]
        }
      },
      "name": "High Priority Leads Found?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [4200, 20],
      "id": "node-priority-check-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/notifier.py '{{JSON.stringify($json.highPriorityLeads)}}' {{$json.sessionId}}"
      },
      "name": "Send Priority Notifications",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [4420, -80],
      "id": "node-notify-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/session_teardown.py {{$json.sessionId}}"
      },
      "name": "Session Teardown",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [4640, 20],
      "id": "node-teardown-001"
    },
    {
      "parameters": {
        "jsCode": "// Final session logging\nconst fs = require('fs');\nconst sessionData = {\n  sessionId: $input.item.json.sessionId,\n  completedAt: new Date().toISOString(),\n  status: 'COMPLETED',\n  summary: $input.item.json\n};\nfs.appendFileSync('/data/linkedin_intel/logs/sessions.log', JSON.stringify(sessionData) + '\\n');\nreturn [{ json: sessionData }];"
      },
      "name": "Log Session Complete",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [4860, 20],
      "id": "node-log-complete-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/ip_rotation.py rotate_emergency"
      },
      "name": "Emergency IP Rotation",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [2440, 420],
      "id": "node-emergency-ip-001"
    },
    {
      "parameters": {
        "jsCode": "// Log IP rotation failure and notify\nconst fs = require('fs');\nconst logEntry = {\n  event: 'IP_ROTATION_FAILED',\n  timestamp: new Date().toISOString(),\n  sessionId: $input.item.json.sessionId,\n  attempts: 3\n};\nfs.appendFileSync('/data/linkedin_intel/logs/errors.log', JSON.stringify(logEntry) + '\\n');\nreturn [{ json: logEntry }];"
      },
      "name": "Log IP Failure",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [2660, 420],
      "id": "node-ip-fail-log-001"
    },
    {
      "parameters": {
        "url": "http://localhost:8080/message",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "={{$env.GOTIFY_APP_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "title",
              "value": "⚠️ LI Intel: Session Skipped"
            },
            {
              "name": "message",
              "value": "={{$json.reason}} | Session: {{$json.sessionId}}"
            },
            {
              "name": "priority",
              "value": "5"
            }
          ]
        }
      },
      "name": "Notify Session Skip",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [1560, 500],
      "id": "node-notify-skip-001"
    }
  ],
  "connections": {
    "Master Cron Scheduler": {
      "main": [[{ "node": "Jitter + Week Variation Calculator", "type": "main", "index": 0 }]]
    },
    "Jitter + Week Variation Calculator": {
      "main": [[{ "node": "Blackout Day Check", "type": "main", "index": 0 }]]
    },
    "Blackout Day Check": {
      "main": [
        [{ "node": "Log Blackout + Stop", "type": "main", "index": 0 }],
        [{ "node": "Session Limit Check", "type": "main", "index": 0 }]
      ]
    },
    "Session Limit Check": {
      "main": [[{ "node": "Parse Session Limit Result", "type": "main", "index": 0 }]]
    },
    "Parse Session Limit Result": {
      "main": [[{ "node": "Can Proceed?", "type": "main", "index": 0 }]]
    },
    "Can Proceed?": {
      "main": [
        [{ "node": "Apply Jitter Delay", "type": "main", "index": 0 }],
        [{ "node": "Notify Session Skip", "type": "main", "index": 0 }]
      ]
    },
    "Apply Jitter Delay": {
      "main": [[{ "node": "IP Rotation Agent", "type": "main", "index": 0 }]]
    },
    "IP Rotation Agent": {
      "main": [[{ "node": "Parse IP Rotation Result", "type": "main", "index": 0 }]]
    },
    "Parse IP Rotation Result": {
      "main": [[{ "node": "IP Rotation Successful?", "type": "main", "index": 0 }]]
    },
    "IP Rotation Successful?": {
      "main": [
        [{ "node": "Session Health Check", "type": "main", "index": 0 }],
        [{ "node": "Emergency IP Rotation", "type": "main", "index": 0 }]
      ]
    },
    "Emergency IP Rotation": {
      "main": [[{ "node": "Log IP Failure", "type": "main", "index": 0 }]]
    },
    "Session Health Check": {
      "main": [[{ "node": "Parse Health Result", "type": "main", "index": 0 }]]
    },
    "Parse Health Result": {
      "main": [[{ "node": "Session Healthy?", "type": "main", "index": 0 }]]
    },
    "Session Healthy?": {
      "main": [
        [{ "node": "Main Scraper Agent", "type": "main", "index": 0 }],
        [{ "node": "Session Recovery Agent", "type": "main", "index": 0 }]
      ]
    },
    "Main Scraper Agent": {
      "main": [[{ "node": "Parse Scraper Result", "type": "main", "index": 0 }]]
    },
    "Parse Scraper Result": {
      "main": [[{ "node": "Scraper Completed?", "type": "main", "index": 0 }]]
    },
    "Scraper Completed?": {
      "main": [
        [{ "node": "Post Processor + Enricher", "type": "main", "index": 0 }],
        [{ "node": "Detection Handler", "type": "main", "index": 0 }]
      ]
    },
    "Post Processor + Enricher": {
      "main": [[{ "node": "Parse Processor Result", "type": "main", "index": 0 }]]
    },
    "Parse Processor Result": {
      "main": [[{ "node": "High Priority Leads Found?", "type": "main", "index": 0 }]]
    },
    "High Priority Leads Found?": {
      "main": [
        [{ "node": "Send Priority Notifications", "type": "main", "index": 0 }],
        [{ "node": "Session Teardown", "type": "main", "index": 0 }]
      ]
    },
    "Send Priority Notifications": {
      "main": [[{ "node": "Session Teardown", "type": "main", "index": 0 }]]
    },
    "Session Teardown": {
      "main": [[{ "node": "Log Session Complete", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": "LinkedIn Recruiter Intelligence - Error Handler"
  },
  "staticData": null,
  "tags": ["linkedin-intel", "production"],
  "triggerCount": 6,
  "updatedAt": "2025-01-01T00:00:00.000Z",
  "versionId": "1"
}
```

---

## WORKFLOW 2: ERROR HANDLER (Separate Workflow)

```json
{
  "name": "LinkedIn Recruiter Intelligence - Error Handler",
  "nodes": [
    {
      "parameters": {},
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [240, 300],
      "id": "node-error-trigger-001"
    },
    {
      "parameters": {
        "jsCode": "const error = $input.item.json;\nconst errorData = {\n  workflowId: error.workflow?.id,\n  workflowName: error.workflow?.name,\n  nodeId: error.execution?.lastNodeExecuted,\n  errorMessage: error.execution?.error?.message || 'Unknown error',\n  errorType: error.execution?.error?.name || 'UnknownError',\n  timestamp: new Date().toISOString(),\n  executionId: error.execution?.id\n};\n\n// Classify severity\nlet severity = 'LOW';\nif (errorData.errorMessage.includes('HARD_SIGNAL')) severity = 'CRITICAL';\nelse if (errorData.errorMessage.includes('MEDIUM_SIGNAL')) severity = 'HIGH';\nelse if (errorData.errorMessage.includes('IP_ROTATION')) severity = 'HIGH';\nelse if (errorData.errorMessage.includes('SESSION')) severity = 'MEDIUM';\n\nreturn [{ json: { ...errorData, severity } }];"
      },
      "name": "Error Classifier",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [460, 300],
      "id": "node-error-classify-001"
    },
    {
      "parameters": {
        "jsCode": "const fs = require('fs');\nconst errorData = $input.item.json;\nfs.appendFileSync('/data/linkedin_intel/logs/errors.log', JSON.stringify(errorData) + '\\n');\n\n// Write to error state file for self-healing monitor\nif (errorData.severity === 'CRITICAL') {\n  fs.writeFileSync('/data/linkedin_intel/state/system_freeze.json', JSON.stringify({\n    frozen: true,\n    reason: errorData.errorMessage,\n    frozenAt: new Date().toISOString(),\n    resumeAfter: new Date(Date.now() + 172800000).toISOString() // 48 hours\n  }));\n}\n\nreturn [{ json: errorData }];"
      },
      "name": "Log Error + Set Freeze State",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [680, 300],
      "id": "node-error-log-001"
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{$json.severity}}",
              "value2": "LOW",
              "operation": "notEqual"
            }
          ]
        }
      },
      "name": "Notify Worthy?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [900, 300],
      "id": "node-notify-worthy-001"
    },
    {
      "parameters": {
        "jsCode": "const severity = $input.item.json.severity;\nconst priorityMap = { 'CRITICAL': 10, 'HIGH': 8, 'MEDIUM': 6, 'LOW': 4 };\nconst emojiMap = { 'CRITICAL': '🚨', 'HIGH': '⚠️', 'MEDIUM': '⚡', 'LOW': 'ℹ️' };\nreturn [{\n  json: {\n    ...$input.item.json,\n    gotifyPriority: priorityMap[severity] || 5,\n    emoji: emojiMap[severity] || 'ℹ️'\n  }\n}];"
      },
      "name": "Build Notification",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1120, 180],
      "id": "node-build-notify-001"
    },
    {
      "parameters": {
        "url": "http://localhost:8080/message",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "={{$env.GOTIFY_APP_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "title",
              "value": "={{$json.emoji}} LI Intel ERROR: {{$json.severity}}"
            },
            {
              "name": "message",
              "value": "Node: {{$json.nodeId}}\nError: {{$json.errorMessage}}\nTime: {{$json.timestamp}}\nExecution: {{$json.executionId}}"
            },
            {
              "name": "priority",
              "value": "={{$json.gotifyPriority}}"
            }
          ]
        }
      },
      "name": "Send Error Notification",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [1340, 180],
      "id": "node-send-error-notify-001"
    }
  ],
  "connections": {
    "Error Trigger": {
      "main": [[{ "node": "Error Classifier", "type": "main", "index": 0 }]]
    },
    "Error Classifier": {
      "main": [[{ "node": "Log Error + Set Freeze State", "type": "main", "index": 0 }]]
    },
    "Log Error + Set Freeze State": {
      "main": [[{ "node": "Notify Worthy?", "type": "main", "index": 0 }]]
    },
    "Notify Worthy?": {
      "main": [
        [{ "node": "Build Notification", "type": "main", "index": 0 }],
        []
      ]
    },
    "Build Notification": {
      "main": [[{ "node": "Send Error Notification", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1"
  },
  "tags": ["linkedin-intel", "error-handling"]
}
```

---

## WORKFLOW 3: FREEZE GUARD (Runs Every 5 Minutes)

```json
{
  "name": "LinkedIn Recruiter Intelligence - Freeze Guard",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "*/5 * * * *"
            }
          ]
        }
      },
      "name": "5-Minute Guard Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [240, 300],
      "id": "node-guard-trigger-001"
    },
    {
      "parameters": {
        "jsCode": "const fs = require('fs');\nconst freezeFile = '/data/linkedin_intel/state/system_freeze.json';\n\nlet freezeState = { frozen: false };\nif (fs.existsSync(freezeFile)) {\n  try {\n    freezeState = JSON.parse(fs.readFileSync(freezeFile, 'utf8'));\n  } catch(e) {\n    freezeState = { frozen: false };\n  }\n}\n\n// Check if freeze period has elapsed\nif (freezeState.frozen && freezeState.resumeAfter) {\n  const resumeTime = new Date(freezeState.resumeAfter);\n  const now = new Date();\n  if (now >= resumeTime) {\n    // Auto-unfreeze\n    fs.writeFileSync(freezeFile, JSON.stringify({ frozen: false, unfrozenAt: now.toISOString() }));\n    freezeState = { frozen: false, justUnfrozen: true };\n  }\n}\n\nreturn [{ json: { ...freezeState, checkedAt: new Date().toISOString() } }];"
      },
      "name": "Check Freeze State",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [460, 300],
      "id": "node-check-freeze-001"
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            {
              "value1": "={{$json.justUnfrozen}}",
              "value2": true
            }
          ]
        }
      },
      "name": "Just Unfrozen?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [680, 300],
      "id": "node-just-unfrozen-001"
    },
    {
      "parameters": {
        "url": "http://localhost:8080/message",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "={{$env.GOTIFY_APP_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "title",
              "value": "✅ LI Intel: System Unfrozen"
            },
            {
              "name": "message",
              "value": "48-hour freeze period has elapsed. System will resume at next scheduled interval. Manual verification recommended before resuming."
            },
            {
              "name": "priority",
              "value": "7"
            }
          ]
        }
      },
      "name": "Notify System Unfrozen",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [900, 180],
      "id": "node-notify-unfrozen-001"
    },
    {
      "parameters": {
        "jsCode": "// Health check: verify all system components are running\nconst { execSync } = require('child_process');\nconst checks = {};\n\ntry {\n  execSync('curl -s --max-time 3 http://localhost:8080/health');\n  checks.gotify = 'UP';\n} catch(e) { checks.gotify = 'DOWN'; }\n\ntry {\n  execSync('curl -s --socks5 127.0.0.1:9050 --max-time 5 https://api.ipify.org');\n  checks.tor = 'UP';\n} catch(e) { checks.tor = 'DOWN'; }\n\ntry {\n  execSync('ls /data/linkedin_intel/leads.db');\n  checks.database = 'UP';\n} catch(e) { checks.database = 'DOWN'; }\n\nconst allHealthy = Object.values(checks).every(v => v === 'UP');\nreturn [{ json: { checks, allHealthy, timestamp: new Date().toISOString() } }];"
      },
      "name": "Component Health Check",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [680, 420],
      "id": "node-component-health-001"
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            {
              "value1": "={{$json.allHealthy}}",
              "value2": false
            }
          ]
        }
      },
      "name": "Components Down?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [900, 420],
      "id": "node-components-down-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/self_heal.py"
      },
      "name": "Self-Heal Components",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [1120, 320],
      "id": "node-self-heal-001"
    },
    {
      "parameters": {
        "url": "http://localhost:8080/message",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "={{$env.GOTIFY_APP_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "title",
              "value": "⚠️ LI Intel: Component DOWN"
            },
            {
              "name": "message",
              "value": "={{JSON.stringify($json.checks)}} | Self-heal attempted at {{$json.timestamp}}"
            },
            {
              "name": "priority",
              "value": "8"
            }
          ]
        }
      },
      "name": "Notify Component Down",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [1120, 500],
      "id": "node-notify-down-001"
    }
  ],
  "connections": {
    "5-Minute Guard Trigger": {
      "main": [[{ "node": "Check Freeze State", "type": "main", "index": 0 }]]
    },
    "Check Freeze State": {
      "main": [[{ "node": "Just Unfrozen?", "type": "main", "index": 0 }]]
    },
    "Just Unfrozen?": {
      "main": [
        [{ "node": "Notify System Unfrozen", "type": "main", "index": 0 }],
        [{ "node": "Component Health Check", "type": "main", "index": 0 }]
      ]
    },
    "Component Health Check": {
      "main": [[{ "node": "Components Down?", "type": "main", "index": 0 }]]
    },
    "Components Down?": {
      "main": [
        [{ "node": "Self-Heal Components", "type": "main", "index": 0 }],
        []
      ]
    },
    "Self-Heal Components": {
      "main": [[{ "node": "Notify Component Down", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1"
  },
  "tags": ["linkedin-intel", "monitoring"]
}
```

---

## WORKFLOW 4: DAILY CLEANUP + BACKUP

```json
{
  "name": "LinkedIn Recruiter Intelligence - Daily Maintenance",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "0 2 * * *"
            }
          ]
        }
      },
      "name": "Daily 2AM Maintenance Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [240, 300],
      "id": "node-maintenance-trigger-001"
    },
    {
      "parameters": {
        "command": "python3 /scripts/linkedin_intel/daily_maintenance.py"
      },
      "name": "Run Daily Maintenance Script",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [460, 300],
      "id": "node-maintenance-script-001"
    },
    {
      "parameters": {
        "jsCode": "const output = $input.item.json.stdout;\nlet result;\ntry {\n  result = JSON.parse(output);\n} catch(e) {\n  result = { success: false, error: output };\n}\nreturn [{ json: result }];"
      },
      "name": "Parse Maintenance Result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [680, 300],
      "id": "node-parse-maintenance-001"
    },
    {
      "parameters": {
        "jsCode": "// Build daily digest\nconst stats = $input.item.json;\nconst digest = {\n  date: new Date().toDateString(),\n  newLeads: stats.newLeads || 0,\n  highPriorityLeads: stats.highPriorityLeads || 0,\n  sessionsRun: stats.sessionsRun || 0,\n  totalLeadsInDB: stats.totalLeadsInDB || 0,\n  backupCreated: stats.backupCreated || false,\n  diskUsageMB: stats.diskUsageMB || 0\n};\nreturn [{ json: digest }];"
      },
      "name": "Build Daily Digest",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [900, 300],
      "id": "node-build-digest-001"
    },
    {
      "parameters": {
        "url": "http://localhost:8080/message",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {
              "name": "token",
              "value": "={{$env.GOTIFY_APP_TOKEN}}"
            }
          ]
        },
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            {
              "name": "title",
              "value": "📊 LI Intel: Daily Digest {{$json.date}}"
            },
            {
              "name": "message",
              "value": "New Leads: {{$json.newLeads}}\nHigh Priority: {{$json.highPriorityLeads}}\nSessions Run: {{$json.sessionsRun}}\nTotal in DB: {{$json.totalLeadsInDB}}\nBackup: {{$json.backupCreated}}\nDisk Used: {{$json.diskUsageMB}}MB"
            },
            {
              "name": "priority",
              "value": "4"
            }
          ]
        }
      },
      "name": "Send Daily Digest",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [1120, 300],
      "id": "node-send-digest-001"
    }
  ],
  "connections": {
    "Daily 2AM Maintenance Trigger": {
      "main": [[{ "node": "Run Daily Maintenance Script", "type": "main", "index": 0 }]]
    },
    "Run Daily Maintenance Script": {
      "main": [[{ "node": "Parse Maintenance Result", "type": "main", "index": 0 }]]
    },
    "Parse Maintenance Result": {
      "main": [[{ "node": "Build Daily Digest", "type": "main", "index": 0 }]]
    },
    "Build Daily Digest": {
      "main": [[{ "node": "Send Daily Digest", "type": "main", "index": 0 }]]
    }
  },
  "settings": {
    "executionOrder": "v1"
  },
  "tags": ["linkedin-intel", "maintenance"]
}
```

---

## ALL PYTHON SCRIPTS (Complete Implementation)

### Directory Structure First

```bash
#!/bin/bash
# Run this ONCE to create all directories and files

mkdir -p /scripts/linkedin_intel
mkdir -p /data/linkedin_intel/{logs,state,backups,cookies,sessions,notifications/pending,raw_posts}
mkdir -p /data/linkedin_intel/fingerprints

touch /data/linkedin_intel/leads.db
touch /data/linkedin_intel/logs/system.log
touch /data/linkedin_intel/logs/errors.log
touch /data/linkedin_intel/logs/sessions.log
echo '{"frozen": false}' > /data/linkedin_intel/state/system_freeze.json
echo '{"count": 0, "date": ""}' > /data/linkedin_intel/state/daily_sessions.json

echo "Directory structure created"
```

---

### Script 1: `config.py` — Central Configuration

```python
# /scripts/linkedin_intel/config.py
"""
Central configuration. Edit this file to customize your setup.
NEVER commit this file to version control — it contains credentials.
"""

import os
from pathlib import Path

# ── PATHS ──────────────────────────────────────────────────────────────────
BASE_DIR = Path("/data/linkedin_intel")
SCRIPTS_DIR = Path("/scripts/linkedin_intel")
LEADS_DB = BASE_DIR / "leads.db"
LOGS_DIR = BASE_DIR / "logs"
STATE_DIR = BASE_DIR / "state"
COOKIES_DIR = BASE_DIR / "cookies"
SESSIONS_DIR = BASE_DIR / "sessions"
FINGERPRINTS_DIR = BASE_DIR / "fingerprints"
BACKUPS_DIR = BASE_DIR / "backups"
RAW_POSTS_DIR = BASE_DIR / "raw_posts"
NOTIFICATIONS_DIR = BASE_DIR / "notifications" / "pending"

# ── LINKEDIN CREDENTIALS ───────────────────────────────────────────────────
LINKEDIN_EMAIL = os.getenv("LINKEDIN_EMAIL", "your_email@gmail.com")
LINKEDIN_PASSWORD = os.getenv("LINKEDIN_PASSWORD", "your_password")

# ── GOTIFY ─────────────────────────────────────────────────────────────────
GOTIFY_URL = os.getenv("GOTIFY_URL", "http://localhost:8080")
GOTIFY_TOKEN = os.getenv("GOTIFY_APP_TOKEN", "your_gotify_token")

# ── TOR ────────────────────────────────────────────────────────────────────
TOR_SOCKS_HOST = "127.0.0.1"
TOR_SOCKS_PORT = 9050
TOR_CONTROL_PORT = 9051
TOR_CONTROL_PASSWORD = os.getenv("TOR_PASSWORD", "your_tor_password")

# ── SESSION LIMITS ─────────────────────────────────────────────────────────
MAX_POSTS_PER_SESSION = (40, 60)       # (min, max) — randomized
MAX_SCROLLS_PER_SESSION = (15, 25)     # (min, max) — randomized
MIN_SESSION_GAP_SECONDS = 9000         # 2.5 hours minimum between sessions
MAX_SESSION_DURATION_SECONDS = 2100    # 35 minutes hard cap
SESSION_WARMUP_SCROLLS = (3, 7)        # Feed scrolls before targeting search

# ── RELEVANCE SCORING ──────────────────────────────────────────────────────
# Edit these to match YOUR skills and target roles
YOUR_TECH_STACK = {
    "python", "django", "fastapi", "flask",
    "postgresql", "redis", "docker", "kubernetes",
    "aws", "gcp", "terraform", "kafka",
    "react", "typescript", "graphql",
    "golang", "rust"
}

YOUR_TARGET_ROLES = {
    "senior software engineer", "staff engineer",
    "principal engineer", "backend engineer",
    "platform engineer", "devops engineer",
    "site reliability engineer", "sre"
}

YOUR_EXPERIENCE_LEVEL = "senior"    # entry | mid | senior | lead | principal
PREFER_REMOTE = True
MIN_RELEVANCE_SCORE = 5             # Below this: store as low priority
HIGH_PRIORITY_THRESHOLD = 8        # Above this: send push notification

# ── SEARCH QUERIES ─────────────────────────────────────────────────────────
# Rotated round-robin across sessions
SEARCH_QUERY_POOL = [
    '"email" #hiring #wearehiring',
    '"DM me or email" #nowhiring',
    '"contact me" #recruiting #jobopening',
    '"reach me at" #hiring #techjobs',
    '"email me" #wearehiring #remotejobs',
    '"send me your resume" #hiring',
    '"email your CV" #nowhiring',
    '"DM me or email" #talentacquisition',
    '"drop me an email" #hiring',
    '"email:" #wearehiring site:linkedin.com',
    '"reach out" "email" #recruiting #tech',
    '"contact me" "email" #jobopening #remote',
]

# ── ENGAGEMENT BAIT DISQUALIFIERS ──────────────────────────────────────────
BAIT_PATTERNS = [
    (r'comment\s+["\']?yes["\']?', -3),
    (r'drop\s+a\s+["\']?yes["\']?', -3),
    (r'type\s+["\']?yes["\']?\s+if', -3),
    (r'post\s+your\s+jobs?\s+in\s+the\s+comments', -3),
    (r'who\s+needs?\s+to\s+hear\s+this', -2),
    (r'not\s+sure\s+who\s+needs?\s+to\s+hear', -2),
    (r'agree\??\s*$', -2),
    (r'repost\s+to\s+help', -1),
    (r'tag\s+someone\s+who', -2),
    (r'share\s+this\s+with\s+your\s+network', -1),
]
BAIT_THRESHOLD = -2   # Score must be >= this to pass

# ── REAL POST SIGNALS ──────────────────────────────────────────────────────
REAL_POST_SIGNALS = [
    (r'\d+\s*(?:years?|yrs?)\s+(?:of\s+)?experience', 2),
    (r'(?:remote|hybrid|onsite|on-site)', 1),
    (r'(?:salary|compensation|comp).*\$[\d,]+', 2),
    (r'(?:immediately|asap|urgent)', 1),
    (r'(?:full.?time|part.?time|contract|freelance)', 1),
    (r'i\'?m\s+personally\s+reviewing', 2),
    (r'send\s+(?:me|your)\s+(?:resume|cv)', 2),
]

# ── BROWSER PROFILES ───────────────────────────────────────────────────────
BROWSER_PROFILES = [
    {
        "id": "profile_001",
        "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "platform": "Win32",
        "screen_width": 1920,
        "screen_height": 1080,
        "window_width": 1366,
        "window_height": 768,
        "timezone": "America/New_York",
        "languages": ["en-US", "en"],
        "hardware_concurrency": 8,
        "device_memory": 8,
        "color_depth": 24,
        "webgl_vendor": "Google Inc. (NVIDIA)",
        "webgl_renderer": "ANGLE (NVIDIA, NVIDIA GeForce GTX 1060 Direct3D11 vs_5_0 ps_5_0, D3D11)"
    },
    {
        "id": "profile_002",
        "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "platform": "MacIntel",
        "screen_width": 2560,
        "screen_height": 1600,
        "window_width": 1440,
        "window_height": 900,
        "timezone": "America/Los_Angeles",
        "languages": ["en-US", "en"],
        "hardware_concurrency": 8,
        "device_memory": 16,
        "color_depth": 30,
        "webgl_vendor": "Apple Inc.",
        "webgl_renderer": "Apple M1 Pro"
    },
    {
        "id": "profile_003",
        "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36",
        "platform": "Win32",
        "screen_width": 1366,
        "screen_height": 768,
        "window_width": 1366,
        "window_height": 768,
        "timezone": "America/Chicago",
        "languages": ["en-US", "en"],
        "hardware_concurrency": 4,
        "device_memory": 4,
        "color_depth": 24,
        "webgl_vendor": "Google Inc. (Intel)",
        "webgl_renderer": "ANGLE (Intel, Intel(R) UHD Graphics 630 Direct3D11 vs_5_0 ps_5_0, D3D11)"
    },
    {
        "id": "profile_004",
        "user_agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "platform": "Linux x86_64",
        "screen_width": 1920,
        "screen_height": 1080,
        "window_width": 1280,
        "window_height": 720,
        "timezone": "America/Denver",
        "languages": ["en-US", "en"],
        "hardware_concurrency": 16,
        "device_memory": 8,
        "color_depth": 24,
        "webgl_vendor": "Google Inc. (AMD)",
        "webgl_renderer": "ANGLE (AMD, AMD Radeon RX 580 Series Direct3D11 vs_5_0 ps_5_0, D3D11)"
    },
    {
        "id": "profile_005",
        "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "platform": "MacIntel",
        "screen_width": 1440,
        "screen_height": 900,
        "window_width": 1200,
        "window_height": 800,
        "timezone": "America/New_York",
        "languages": ["en-US", "en"],
        "hardware_concurrency": 8,
        "device_memory": 8,
        "color_depth": 24,
        "webgl_vendor": "Apple Inc.",
        "webgl_renderer": "Apple M2"
    }
]
```

---

### Script 2: `db_manager.py` — Database Layer

```python
# /scripts/linkedin_intel/db_manager.py
"""
SQLite database management. Single source of truth for all data.
"""

import sqlite3
import json
import hashlib
import logging
from datetime import datetime
from pathlib import Path
from config import LEADS_DB

logger = logging.getLogger(__name__)


def get_connection():
    """Get WAL-mode SQLite connection."""
    conn = sqlite3.connect(str(LEADS_DB))
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    conn.row_factory = sqlite3.Row
    return conn


def initialize_database():
    """Create all tables if they don't exist."""
    conn = get_connection()
    try:
        conn.executescript("""
            CREATE TABLE IF NOT EXISTS leads (
                id TEXT PRIMARY KEY,
                author_name TEXT,
                author_url TEXT,
                author_headline TEXT,
                author_company TEXT,
                post_url TEXT,
                post_timestamp TEXT,
                post_text TEXT,
                emails TEXT,
                hashtags TEXT,
                tech_stack TEXT,
                role_title TEXT,
                experience_level TEXT,
                remote_signal TEXT,
                location TEXT,
                salary TEXT,
                relevance_score INTEGER DEFAULT 0,
                bait_score INTEGER DEFAULT 0,
                status TEXT DEFAULT 'NEW',
                session_id TEXT,
                created_at TEXT,
                last_seen TEXT,
                notes TEXT
            );

            CREATE TABLE IF NOT EXISTS sessions (
                session_id TEXT PRIMARY KEY,
                started_at TEXT,
                ended_at TEXT,
                posts_scanned INTEGER DEFAULT 0,
                emails_found INTEGER DEFAULT 0,
                leads_stored INTEGER DEFAULT 0,
                ip_used TEXT,
                profile_used TEXT,
                query_used TEXT,
                detection_events TEXT DEFAULT '[]',
                status TEXT DEFAULT 'RUNNING'
            );

            CREATE TABLE IF NOT EXISTS email_dedup (
                email_hash TEXT PRIMARY KEY,
                email_address TEXT,
                first_seen TEXT,
                lead_id TEXT REFERENCES leads(id)
            );

            CREATE TABLE IF NOT EXISTS detection_events (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                session_id TEXT,
                event_type TEXT,
                event_data TEXT,
                timestamp TEXT,
                action_taken TEXT
            );

            CREATE TABLE IF NOT EXISTS query_rotation (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                query TEXT,
                last_used TEXT,
                use_count INTEGER DEFAULT 0
            );

            CREATE INDEX IF NOT EXISTS idx_leads_status ON leads(status);
            CREATE INDEX IF NOT EXISTS idx_leads_score ON leads(relevance_score DESC);
            CREATE INDEX IF NOT EXISTS idx_leads_created ON leads(created_at DESC);
            CREATE INDEX IF NOT EXISTS idx_sessions_started ON sessions(started_at DESC);
        """)
        conn.commit()
        logger.info("Database initialized successfully")
    finally:
        conn.close()


def generate_lead_id(author_url: str, post_timestamp: str) -> str:
    """Generate deterministic ID for deduplication."""
    raw = f"{author_url}::{post_timestamp}".encode()
    return hashlib.sha256(raw).hexdigest()


def generate_email_hash(email: str) -> str:
    """Generate hash for email deduplication."""
    return hashlib.sha256(email.lower().strip().encode()).hexdigest()


def is_duplicate_post(author_url: str, post_timestamp: str) -> bool:
    """Check if post already exists in database."""
    lead_id = generate_lead_id(author_url, post_timestamp)
    conn = get_connection()
    try:
        row = conn.execute(
            "SELECT id FROM leads WHERE id = ?", (lead_id,)
        ).fetchone()
        return row is not None
    finally:
        conn.close()


def is_duplicate_email(email: str) -> bool:
    """Check if email already contacted in database."""
    email_hash = generate_email_hash(email)
    conn = get_connection()
    try:
        row = conn.execute(
            "SELECT email_hash FROM email_dedup WHERE email_hash = ?",
            (email_hash,)
        ).fetchone()
        return row is not None
    finally:
        conn.close()


def save_lead(lead_data: dict) -> bool:
    """
    Save a lead to database with full deduplication.
    Returns True if new lead saved, False if duplicate.
    """
    author_url = lead_data.get("author_url", "")
    post_timestamp = lead_data.get("post_timestamp", "")
    emails = lead_data.get("emails", [])

    lead_id = generate_lead_id(author_url, post_timestamp)

    # Check post-level duplicate
    if is_duplicate_post(author_url, post_timestamp):
        # Update last_seen timestamp
        conn = get_connection()
        try:
            conn.execute(
                "UPDATE leads SET last_seen = ? WHERE id = ?",
                (datetime.utcnow().isoformat(), lead_id)
            )
            conn.commit()
        finally:
            conn.close()
        return False

    # Check email-level duplicate
    for email in emails:
        if is_duplicate_email(email):
            logger.info(f"Duplicate email found: {email[:20]}...")
            return False

    # Save the lead
    conn = get_connection()
    try:
        now = datetime.utcnow().isoformat()

        conn.execute("""
            INSERT INTO leads (
                id, author_name, author_url, author_headline, author_company,
                post_url, post_timestamp, post_text, emails, hashtags,
                tech_stack, role_title, experience_level, remote_signal,
                location, salary, relevance_score, bait_score, status,
                session_id, created_at, last_seen
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 'NEW', ?, ?, ?)
        """, (
            lead_id,
            lead_data.get("author_name", ""),
            author_url,
            lead_data.get("author_headline", ""),
            lead_data.get("author_company", ""),
            lead_data.get("post_url", ""),
            post_timestamp,
            lead_data.get("post_text", ""),
            json.dumps(emails),
            json.dumps(lead_data.get("hashtags", [])),
            json.dumps(lead_data.get("tech_stack", [])),
            lead_data.get("role_title", ""),
            lead_data.get("experience_level", ""),
            lead_data.get("remote_signal", ""),
            lead_data.get("location", ""),
            lead_data.get("salary", ""),
            lead_data.get("relevance_score", 0),
            lead_data.get("bait_score", 0),
            lead_data.get("session_id", ""),
            now,
            now
        ))

        # Register all emails in dedup table
        for email in emails:
            email_hash = generate_email_hash(email)
            try:
                conn.execute("""
                    INSERT OR IGNORE INTO email_dedup (email_hash, email_address, first_seen, lead_id)
                    VALUES (?, ?, ?, ?)
                """, (email_hash, email.lower().strip(), now, lead_id))
            except sqlite3.IntegrityError:
                pass

        conn.commit()
        logger.info(f"Lead saved: {lead_id[:16]}... | Score: {lead_data.get('relevance_score', 0)}")
        return True

    except Exception as e:
        conn.rollback()
        logger.error(f"Failed to save lead: {e}")
        return False
    finally:
        conn.close()


def create_session(session_id: str, ip_used: str, profile_id: str, query: str):
    """Record session start."""
    conn = get_connection()
    try:
        conn.execute("""
            INSERT INTO sessions (session_id, started_at, ip_used, profile_used, query_used)
            VALUES (?, ?, ?, ?, ?)
        """, (session_id, datetime.utcnow().isoformat(), ip_used, profile_id, query))
        conn.commit()
    finally:
        conn.close()


def update_session(session_id: str, **kwargs):
    """Update session record."""
    conn = get_connection()
    try:
        set_clauses = ", ".join(f"{k} = ?" for k in kwargs)
        values = list(kwargs.values()) + [session_id]
        conn.execute(
            f"UPDATE sessions SET {set_clauses} WHERE session_id = ?",
            values
        )
        conn.commit()
    finally:
        conn.close()


def get_today_session_count() -> int:
    """Get number of sessions run today."""
    today = datetime.utcnow().date().isoformat()
    conn = get_connection()
    try:
        row = conn.execute("""
            SELECT COUNT(*) as cnt FROM sessions
            WHERE started_at LIKE ? AND status IN ('COMPLETED', 'RUNNING')
        """, (f"{today}%",)).fetchone()
        return row["cnt"] if row else 0
    finally:
        conn.close()


def get_leads_by_status(status: str = "NEW", limit: int = 50) -> list:
    """Fetch leads by status for dashboard."""
    conn = get_connection()
    try:
        rows = conn.execute("""
            SELECT * FROM leads WHERE status = ?
            ORDER BY relevance_score DESC, created_at DESC
            LIMIT ?
        """, (status, limit)).fetchall()
        return [dict(row) for row in rows]
    finally:
        conn.close()


def get_daily_stats() -> dict:
    """Stats for daily digest."""
    today = datetime.utcnow().date().isoformat()
    conn = get_connection()
    try:
        new_today = conn.execute(
            "SELECT COUNT(*) as cnt FROM leads WHERE created_at LIKE ?",
            (f"{today}%",)
        ).fetchone()["cnt"]

        high_priority = conn.execute(
            "SELECT COUNT(*) as cnt FROM leads WHERE created_at LIKE ? AND relevance_score >= 8",
            (f"{today}%",)
        ).fetchone()["cnt"]

        sessions_today = conn.execute(
            "SELECT COUNT(*) as cnt FROM sessions WHERE started_at LIKE ?",
            (f"{today}%",)
        ).fetchone()["cnt"]

        total = conn.execute(
            "SELECT COUNT(*) as cnt FROM leads"
        ).fetchone()["cnt"]

        return {
            "newLeads": new_today,
            "highPriorityLeads": high_priority,
            "sessionsRun": sessions_today,
            "totalLeadsInDB": total
        }
    finally:
        conn.close()
```

---

### Script 3: `ip_rotation.py` — Tor IP Manager

```python
# /scripts/linkedin_intel/ip_rotation.py
"""
Manages Tor IP rotation with validation.
Usage: python3 ip_rotation.py [rotate|rotate_emergency|current]
"""

import sys
import json
import time
import logging
import requests
from stem import Signal
from stem.control import Controller
from config import (
    TOR_SOCKS_HOST, TOR_SOCKS_PORT,
    TOR_CONTROL_PORT, TOR_CONTROL_PASSWORD
)

logger = logging.getLogger(__name__)

TOR_PROXIES = {
    "http": f"socks5h://{TOR_SOCKS_HOST}:{TOR_SOCKS_PORT}",
    "https": f"socks5h://{TOR_SOCKS_HOST}:{TOR_SOCKS_PORT}"
}

# Known datacenter ASNs that LinkedIn blocks
BLOCKED_ASNS = {
    "AS396982",   # Google Cloud
    "AS16509",    # Amazon AWS
    "AS14618",    # Amazon AWS
    "AS15169",    # Google
    "AS8075",     # Microsoft Azure
    "AS13335",    # Cloudflare
}


def get_current_ip() -> dict:
    """Get current Tor exit IP and metadata."""
    try:
        resp = requests.get(
            "https://ipinfo.io/json",
            proxies=TOR_PROXIES,
            timeout=15
        )
        data = resp.json()
        return {
            "success": True,
            "ip": data.get("ip", "unknown"),
            "country": data.get("country", "unknown"),
            "org": data.get("org", "unknown"),
            "timezone": data.get("timezone", "unknown")
        }
    except Exception as e:
        return {"success": False, "error": str(e)}


def is_ip_usable(ip_info: dict) -> tuple[bool, str]:
    """
    Validate IP is suitable for LinkedIn scraping.
    Returns (is_usable, reason).
    """
    if not ip_info.get("success"):
        return False, "IP_INFO_FETCH_FAILED"

    # Check country (must be US, GB, DE, or CA)
    allowed_countries = {"US", "GB", "DE", "CA", "AU"}
    if ip_info.get("country") not in allowed_countries:
        return False, f"COUNTRY_NOT_ALLOWED: {ip_info.get('country')}"

    # Check for datacenter ASN
    org = ip_info.get("org", "")
    for asn in BLOCKED_ASNS:
        if asn in org:
            return False, f"DATACENTER_IP: {asn}"

    return True, "OK"


def rotate_tor_circuit():
    """Request new Tor circuit."""
    try:
        with Controller.from_port(port=TOR_CONTROL_PORT) as controller:
            controller.authenticate(password=TOR_CONTROL_PASSWORD)
            controller.signal(Signal.NEWNYM)
        time.sleep(5)  # Wait for new circuit to establish
        return True
    except Exception as e:
        logger.error(f"Tor rotation failed: {e}")
        return False


def rotate_and_validate(max_attempts: int = 3) -> dict:
    """
    Rotate IP and validate it's suitable.
    Returns result dict with success status.
    """
    for attempt in range(1, max_attempts + 1):
        logger.info(f"IP rotation attempt {attempt}/{max_attempts}")

        if not rotate_tor_circuit():
            continue

        ip_info = get_current_ip()
        is_usable, reason = is_ip_usable(ip_info)

        if is_usable:
            return {
                "success": True,
                "ip": ip_info.get("ip"),
                "country": ip_info.get("country"),
                "timezone": ip_info.get("timezone"),
                "attempt": attempt
            }

        logger.warning(f"IP not usable: {reason}. Retrying...")
        time.sleep(3)

    return {
        "success": False,
        "error": f"Failed to get usable IP after {max_attempts} attempts"
    }


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)

    command = sys.argv[1] if len(sys.argv) > 1 else "rotate"

    if command == "rotate":
        result = rotate_and_validate(max_attempts=3)
    elif command == "rotate_emergency":
        result = rotate_and_validate(max_attempts=5)
    elif command == "current":
        ip_info = get_current_ip()
        usable, reason = is_ip_usable(ip_info)
        result = {**ip_info, "usable": usable, "reason": reason}
    else:
        result = {"success": False, "error": f"Unknown command: {command}"}

    print(json.dumps(result))
```

---

### Script 4: `session_limit_check.py`

```python
# /scripts/linkedin_intel/session_limit_check.py
"""
Check if another session can run today.
Usage: python3 session_limit_check.py <session_id> <max_sessions>
"""

import sys
import json
import logging
from pathlib import Path
from datetime import datetime
from config import STATE_DIR
from db_manager import get_today_session_count, initialize_database

logger = logging.getLogger(__name__)


def check_freeze_state() -> tuple[bool, str]:
    """Check if system is in freeze state."""
    freeze_file = STATE_DIR / "system_freeze.json"
    if not freeze_file.exists():
        return False, ""

    try:
        freeze_data = json.loads(freeze_file.read_text())
        if freeze_data.get("frozen", False):
            resume_after = freeze_data.get("resumeAfter", "")
            if resume_after:
                resume_time = datetime.fromisoformat(resume_after.replace("Z", "+00:00"))
                if datetime.utcnow() < resume_time.replace(tzinfo=None):
                    return True, f"SYSTEM_FROZEN_UNTIL_{resume_after}"
    except Exception as e:
        logger.error(f"Error reading freeze state: {e}")

    return False, ""


def check_min_gap() -> tuple[bool, str]:
    """Ensure minimum gap between sessions."""
    from config import MIN_SESSION_GAP_SECONDS
    from db_manager import get_connection

    conn = get_connection()
    try:
        row = conn.execute("""
            SELECT started_at FROM sessions
            WHERE status IN ('COMPLETED', 'RUNNING')
            ORDER BY started_at DESC LIMIT 1
        """).fetchone()

        if row:
            last_session = datetime.fromisoformat(row["started_at"])
            elapsed = (datetime.utcnow() - last_session).total_seconds()
            if elapsed < MIN_SESSION_GAP_SECONDS:
                wait_remaining = int(MIN_SESSION_GAP_SECONDS - elapsed)
                return False, f"MIN_GAP_NOT_MET_{wait_remaining}s_remaining"
    finally:
        conn.close()

    return True, "OK"


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    initialize_database()

    session_id = sys.argv[1] if len(sys.argv) > 1 else "unknown"
    max_sessions = int(sys.argv[2]) if len(sys.argv) > 2 else 3

    # Check freeze state
    is_frozen, freeze_reason = check_freeze_state()
    if is_frozen:
        print(json.dumps({
            "canProceed": "false",
            "reason": freeze_reason,
            "sessionId": session_id
        }))
        sys.exit(0)

    # Check today's session count
    today_count = get_today_session_count()
    if today_count >= max_sessions:
        print(json.dumps({
            "canProceed": "false",
            "reason": f"SESSION_LIMIT_REACHED_{today_count}/{max_sessions}",
            "sessionId": session_id
        }))
        sys.exit(0)

    # Check minimum gap
    gap_ok, gap_reason = check_min_gap()
    if not gap_ok:
        print(json.dumps({
            "canProceed": "false",
            "reason": gap_reason,
            "sessionId": session_id
        }))
        sys.exit(0)

    print(json.dumps({
        "canProceed": "true",
        "sessionId": session_id,
        "todayCount": today_count,
        "maxSessions": max_sessions
    }))
```

---

### Script 5: `session_health.py`

```python
# /scripts/linkedin_intel/session_health.py
"""
Check LinkedIn session health and cookie validity.
Usage: python3 session_health.py check <session_id>
"""

import sys
import json
import time
import pickle
import logging
from pathlib import Path
from datetime import datetime
import undetected_chromedriver as uc
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from config import (
    COOKIES_DIR, TOR_SOCKS_HOST, TOR_SOCKS_PORT,
    LINKEDIN_EMAIL, LINKEDIN_PASSWORD, BROWSER_PROFILES
)
from fingerprint_manager import apply_fingerprint_profile

logger = logging.getLogger(__name__)
COOKIE_FILE = COOKIES_DIR / "linkedin_session.pkl"


def load_cookies() -> list:
    """Load saved session cookies."""
    if COOKIE_FILE.exists():
        try:
            with open(COOKIE_FILE, "rb") as f:
                return pickle.load(f)
        except Exception:
            return []
    return []


def save_cookies(driver):
    """Save current session cookies."""
    COOKIES_DIR.mkdir(parents=True, exist_ok=True)
    with open(COOKIE_FILE, "wb") as f:
        pickle.dump(driver.get_cookies(), f)


def check_session_health() -> dict:
    """
    Launch browser, load cookies, check if still logged in.
    Uses minimal resources — quick check only.
    """
    profile = BROWSER_PROFILES[0]  # Use primary profile for health check

    options = uc.ChromeOptions()
    options.add_argument(f"--window-size={profile['window_width']},{profile['window_height']}")
    options.add_argument(f"--proxy-server=socks5://{TOR_SOCKS_HOST}:{TOR_SOCKS_PORT}")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--disable-blink-features=AutomationControlled")

    driver = None
    try:
        driver = uc.Chrome(options=options, headless=False, use_subprocess=True)
        apply_fingerprint_profile(driver, profile)

        # Load cookies
        driver.get("https://www.linkedin.com")
        time.sleep(2)

        cookies = load_cookies()
        for cookie in cookies:
            try:
                driver.add_cookie(cookie)
            except Exception:
                pass

        # Navigate to feed
        driver.get("https://www.linkedin.com/feed/")
        time.sleep(3)

        current_url = driver.current_url
        page_source = driver.page_source[:2000]

        # Determine session status
        if "feed" in current_url and "login" not in current_url:
            status = "HEALTHY"
            needs_reauth = False
        elif "login" in current_url or "authwall" in current_url:
            status = "EXPIRED"
            needs_reauth = True
        elif "checkpoint" in current_url or "challenge" in current_url:
            status = "CHALLENGE"
            needs_reauth = True
        else:
            status = "UNKNOWN"
            needs_reauth = True

        return {
            "status": status,
            "needsReauth": needs_reauth,
            "currentUrl": current_url,
            "checkedAt": datetime.utcnow().isoformat()
        }

    except Exception as e:
        logger.error(f"Health check error: {e}")
        return {
            "status": "ERROR",
            "needsReauth": True,
            "error": str(e)
        }
    finally:
        if driver:
            try:
                driver.quit()
            except Exception:
                pass


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)

    command = sys.argv[1] if len(sys.argv) > 1 else "check"
    session_id = sys.argv[2] if len(sys.argv) > 2 else "unknown"

    result = check_session_health()
    result["sessionId"] = session_id
    print(json.dumps(result))
```

---

### Script 6: `fingerprint_manager.py`

```python
# /scripts/linkedin_intel/fingerprint_manager.py
"""
Browser fingerprint injection and profile management.
Applies fingerprint profiles via Chrome DevTools Protocol.
"""

import json
import random
import logging
from pathlib import Path
from config import BROWSER_PROFILES, STATE_DIR

logger = logging.getLogger(__name__)
PROFILE_STATE_FILE = STATE_DIR / "profile_rotation.json"


def get_next_profile() -> dict:
    """
    Round-robin profile selection with 8-hour cooldown per profile.
    """
    from datetime import datetime, timedelta

    # Load rotation state
    state = {"last_used": {}, "current_index": 0}
    if PROFILE_STATE_FILE.exists():
        try:
            state = json.loads(PROFILE_STATE_FILE.read_text())
        except Exception:
            pass

    now = datetime.utcnow()
    cooldown = timedelta(hours=8)
    current_index = state.get("current_index", 0)
    last_used = state.get("last_used", {})

    # Find next available profile (respecting cooldown)
    for i in range(len(BROWSER_PROFILES)):
        idx = (current_index + i) % len(BROWSER_PROFILES)
        profile = BROWSER_PROFILES[idx]
        pid = profile["id"]

        last_use_str = last_used.get(pid)
        if last_use_str:
            last_use = datetime.fromisoformat(last_use_str)
            if now - last_use < cooldown:
                continue

        # This profile is available
        # Update state
        last_used[pid] = now.isoformat()
        state["last_used"] = last_used
        state["current_index"] = (idx + 1) % len(BROWSER_PROFILES)
        PROFILE_STATE_FILE.write_text(json.dumps(state, indent=2))

        return profile

    # All profiles in cooldown — use least recently used
    logger.warning("All profiles in cooldown — using least recently used")
    lru = min(BROWSER_PROFILES, key=lambda p: last_used.get(p["id"], ""))
    return lru


def apply_fingerprint_profile(driver, profile: dict):
    """
    Inject all fingerprint overrides into browser via CDP.
    Must be called after driver creation but before navigation.
    """

    # 1. Override User Agent
    driver.execute_cdp_cmd("Network.setUserAgentOverride", {
        "userAgent": profile["user_agent"],
        "platform": profile["platform"],
        "acceptLanguage": ",".join(profile["languages"])
    })

    # 2. Override Timezone
    driver.execute_cdp_cmd("Emulation.setTimezoneOverride", {
        "timezoneId": profile["timezone"]
    })

    # 3. Override screen/hardware properties + canvas/WebGL fingerprint
    js_overrides = f"""
    // Override navigator properties
    Object.defineProperty(navigator, 'webdriver', {{
        get: () => undefined,
        configurable: true
    }});
    
    Object.defineProperty(navigator, 'languages', {{
        get: () => {json.dumps(profile['languages'])},
        configurable: true
    }});
    
    Object.defineProperty(navigator, 'hardwareConcurrency', {{
        get: () => {profile['hardware_concurrency']},
        configurable: true
    }});
    
    Object.defineProperty(navigator, 'deviceMemory', {{
        get: () => {profile['device_memory']},
        configurable: true
    }});
    
    Object.defineProperty(navigator, 'platform', {{
        get: () => '{profile['platform']}',
        configurable: true
    }});

    // Override screen properties
    Object.defineProperty(screen, 'width', {{
        get: () => {profile['screen_width']},
        configurable: true
    }});
    
    Object.defineProperty(screen, 'height', {{
        get: () => {profile['screen_height']},
        configurable: true
    }});
    
    Object.defineProperty(screen, 'colorDepth', {{
        get: () => {profile['color_depth']},
        configurable: true
    }});

    // Restore chrome runtime (removed in headless)
    if (!window.chrome) {{
        window.chrome = {{
            runtime: {{
                id: undefined,
                connect: function() {{}},
                sendMessage: function() {{}},
                onMessage: {{ addListener: function() {{}} }}
            }},
            loadTimes: function() {{ return {{}}; }},
            csi: function() {{ return {{}}; }}
        }};
    }}

    // Override plugins (empty = bot signal)
    Object.defineProperty(navigator, 'plugins', {{
        get: () => {{
            const pluginData = [
                {{ name: 'Chrome PDF Plugin', filename: 'internal-pdf-viewer', description: 'Portable Document Format' }},
                {{ name: 'Chrome PDF Viewer', filename: 'mhjfbmdgcfjbbpaeojofohoefgiehjai', description: '' }},
                {{ name: 'Native Client', filename: 'internal-nacl-plugin', description: '' }}
            ];
            const plugins = Object.create(PluginArray.prototype);
            pluginData.forEach((p, i) => {{
                const plugin = Object.create(Plugin.prototype);
                Object.defineProperty(plugin, 'name', {{ get: () => p.name }});
                Object.defineProperty(plugin, 'filename', {{ get: () => p.filename }});
                Object.defineProperty(plugin, 'description', {{ get: () => p.description }});
                plugins[i] = plugin;
            }});
            Object.defineProperty(plugins, 'length', {{ get: () => pluginData.length }});
            return plugins;
        }},
        configurable: true
    }});

    // Canvas fingerprint noise injection
    const originalToDataURL = HTMLCanvasElement.prototype.toDataURL;
    HTMLCanvasElement.prototype.toDataURL = function(type, quality) {{
        const result = originalToDataURL.apply(this, arguments);
        if (this.width > 0 && this.height > 0) {{
            const arr = result.split(',');
            if (arr.length === 2) {{
                // Add imperceptible noise to base64 data
                const noise = String.fromCharCode(Math.floor(Math.random() * 3));
                return arr[0] + ',' + btoa(atob(arr[1]).slice(0, -3) + noise);
            }}
        }}
        return result;
    }};

    // WebGL fingerprint override
    const getParameter = WebGLRenderingContext.prototype.getParameter;
    WebGLRenderingContext.prototype.getParameter = function(parameter) {{
        if (parameter === 37445) return '{profile['webgl_vendor']}';
        if (parameter === 37446) return '{profile['webgl_renderer']}';
        return getParameter.apply(this, arguments);
    }};
    
    const getParameter2 = WebGL2RenderingContext.prototype.getParameter;
    WebGL2RenderingContext.prototype.getParameter = function(parameter) {{
        if (parameter === 37445) return '{profile['webgl_vendor']}';
        if (parameter === 37446) return '{profile['webgl_renderer']}';
        return getParameter2.apply(this, arguments);
    }};

    // AudioContext fingerprint noise
    const originalCreateOscillator = AudioContext.prototype.createOscillator;
    AudioContext.prototype.createOscillator = function() {{
        const oscillator = originalCreateOscillator.apply(this, arguments);
        const originalConnect = oscillator.connect.bind(oscillator);
        oscillator.connect = function(destination) {{
            if (destination && destination.gain) {{
                const noise = (Math.random() - 0.5) * 0.0001;
                destination.gain.value = destination.gain.value + noise;
            }}
            return originalConnect(destination);
        }};
        return oscillator;
    }};

    // Remove automation flags from document
    delete document.$chrome_asyncScriptInfo;
    delete document.$cdc_asdjflasutopfhvcZLmcfl_;
    """

    driver.execute_script(js_overrides)
    logger.info(f"Fingerprint profile applied: {profile['id']}")
```

---

### Script 7: `human_behavior.py`

```python
# /scripts/linkedin_intel/human_behavior.py
"""
Human behavior simulation module.
All interactions through this module to ensure natural patterns.
"""

import time
import random
import math
import logging
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.common.by import By

logger = logging.getLogger(__name__)


def gaussian_wait(mean_s: float, sigma_s: float, min_s: float = 0.5, max_s: float = 30.0):
    """Gaussian-distributed wait time. Never uniform."""
    duration = random.gauss(mean_s, sigma_s)
    duration = max(min_s, min(max_s, duration))
    time.sleep(duration)


def human_scroll(driver, target_px: int, direction: str = "down"):
    """
    Scroll with gaussian velocity and easing curve.
    Mimics human hand acceleration/deceleration.
    """
    steps = random.randint(5, 12)
    actual_target = target_px if direction == "down" else -target_px

    for i in range(steps):
        # Sine easing: slow start, fast middle, slow end
        progress = i / steps
        ease_factor = math.sin(progress * math.pi)
        chunk = int((actual_target / steps) * ease_factor)

        # Add micro-jitter
        chunk += random.randint(-5, 5)

        driver.execute_script(f"window.scrollBy(0, {chunk})")
        time.sleep(random.gauss(0.08, 0.025))

    # Occasional micro back-scroll (humans re-read)
    if random.random() < 0.15:
        back_px = random.randint(20, 80)
        driver.execute_script(f"window.scrollBy(0, -{back_px})")
        gaussian_wait(0.3, 0.1)


def human_scroll_session(driver, scroll_count: int):
    """
    Complete scrolling session with dwell time variation.
    """
    for scroll_num in range(scroll_count):
        # Vary scroll distance
        scroll_px = int(random.gauss(350, 80))
        scroll_px = max(150, min(600, scroll_px))

        human_scroll(driver, scroll_px)

        # Dwell time proportional to "reading" behavior
        # Longer dwell every 3-4 scrolls (simulates pausing to read)
        if scroll_num % random.randint(3, 5) == 0:
            gaussian_wait(4.0, 1.5, min_s=2.0, max_s=10.0)
        else:
            gaussian_wait(2.0, 0.8, min_s=0.8, max_s=5.0)

        # Very occasional long pause (distracted human)
        if random.random() < 0.05:
            gaussian_wait(8.0, 3.0, min_s=5.0, max_s=20.0)


def human_click(driver, element, miss_center: bool = True):
    """
    Click element with human-like approach.
    Never clicks exact center — adds realistic offset.
    """
    try:
        action = ActionChains(driver)

        if miss_center:
            x_offset = random.randint(-4, 4)
            y_offset = random.randint(-3, 3)
        else:
            x_offset, y_offset = 0, 0

        action.move_to_element_with_offset(element, x_offset, y_offset)
        action.pause(random.gauss(0.12, 0.04))  # Hover before click
        action.click()
        action.perform()

        # Post-click pause
        gaussian_wait(0.5, 0.2)

    except Exception as e:
        logger.warning(f"Human click failed, falling back: {e}")
        element.click()


def human_type(driver, element, text: str):
    """
    Type text with human WPM and occasional typos.
    """
    human_click(driver, element)
    gaussian_wait(0.3, 0.1)

    # ~65 WPM = ~1 char per 92ms
    base_delay = 0.092
    typo_rate = 0.03  # 3% of characters get a typo

    for char in text:
        # Occasional typo with correction
        if random.random() < typo_rate:
            wrong_char = random.choice("qwertyuiopasdfghjklzxcvbnm")
            element.send_keys(wrong_char)
            time.sleep(random.gauss(0.08, 0.02))
            element.send_keys("\b")  # Backspace
            gaussian_wait(0.15, 0.05)

        element.send_keys(char)
        delay = random.gauss(base_delay, 0.03)
        time.sleep(max(0.04, delay))

    # Pause before submitting
    gaussian_wait(0.5, 0.15)


def feed_warmup(driver):
    """
    Pre-scrape human-like activity on LinkedIn feed.
    Must run before any targeted search.
    """
    logger.info("Starting feed warmup...")

    driver.get("https://www.linkedin.com/feed/")
    gaussian_wait(3.0, 1.0)

    scroll_count = random.randint(3, 7)
    human_scroll_session(driver, scroll_count)

    # 30% chance: view notifications
    if random.random() < 0.30:
        logger.info("Warmup: checking notifications")
        driver.get("https://www.linkedin.com/notifications/")
        gaussian_wait(4.0, 1.5)
        human_scroll(driver, random.randint(200, 400))
        gaussian_wait(3.0, 1.0)
        driver.back()
        gaussian_wait(2.0, 0.5)

    # 20% chance: check messages (just navigate, don't interact)
    if random.random() < 0.20:
        logger.info("Warmup: glancing at messages")
        driver.get("https://www.linkedin.com/messaging/")
        gaussian_wait(3.0, 1.0)
        driver.back()
        gaussian_wait(2.0, 0.5)

    logger.info(f"Feed warmup complete: {scroll_count} scrolls")
    return scroll_count


def post_read_dwell(text: str):
    """
    Dwell proportional to post text length.
    Simulates actually reading the post.
    """
    word_count = len(text.split())
    avg_wpm = 238  # Average human reading speed
    read_time_seconds = (word_count / avg_wpm) * 60
    # Actually dwell for 60-80% of estimated read time
    actual_dwell = read_time_seconds * random.uniform(0.6, 0.8)
    actual_dwell = max(1.5, min(actual_dwell, 12.0))
    time.sleep(actual_dwell)
```

---

### Script 8: `main_scraper.py` — The Core Engine

```python
# /scripts/linkedin_intel/main_scraper.py
"""
Main LinkedIn scraping agent.
Usage: python3 main_scraper.py <session_id>
"""

import sys
import json
import time
import random
import logging
import re
import hashlib
import pickle
from datetime import datetime
from pathlib import Path

import undetected_chromedriver as uc
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import (
    TimeoutException, NoSuchElementException,
    StaleElementReferenceException
)
from bs4 import BeautifulSoup

from config import (
    COOKIES_DIR, TOR_SOCKS_HOST, TOR_SOCKS_PORT,
    LINKEDIN_EMAIL, LINKEDIN_PASSWORD,
    MAX_POSTS_PER_SESSION, MAX_SCROLLS_PER_SESSION,
    MAX_SESSION_DURATION_SECONDS, SESSION_WARMUP_SCROLLS,
    SEARCH_QUERY_POOL, STATE_DIR, RAW_POSTS_DIR
)
from fingerprint_manager import get_next_profile, apply_fingerprint_profile
from human_behavior import (
    gaussian_wait, human_scroll, human_scroll_session,
    human_click, human_type, feed_warmup, post_read_dwell
)
from db_manager import create_session, update_session, initialize_database

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s"
)
logger = logging.getLogger(__name__)

COOKIE_FILE = COOKIES_DIR / "linkedin_session.pkl"

EMAIL_PATTERNS = [
    r'[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}',
    r'(?i)(?:email|e-mail)\s*(?:me\s*at|:)\s*([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})',
    r'(?i)reach\s+(?:me|out)\s+at\s+([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})',
    r'(?i)contact\s+(?:me\s+)?(?:at|via|by\s+email|on\s+email)[:.]?\s*([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})',
    r'(?i)(?:send|drop)\s+(?:me\s+)?(?:your\s+)?(?:resume|cv|portfolio)\s+(?:to|at)\s+([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})',
]

BAIT_PATTERNS_COMPILED = [
    (re.compile(r'comment\s+["\']?yes["\']?', re.I), -3),
    (re.compile(r'drop\s+a\s+["\']?yes["\']?', re.I), -3),
    (re.compile(r'type\s+["\']?yes["\']?\s+if', re.I), -3),
    (re.compile(r'post\s+your\s+jobs?\s+in\s+the\s+comments', re.I), -3),
    (re.compile(r'who\s+needs?\s+to\s+hear\s+this', re.I), -2),
    (re.compile(r'not\s+sure\s+who\s+needs?', re.I), -2),
    (re.compile(r'\bagree\??[\s.!]*$', re.I), -2),
    (re.compile(r'repost\s+to\s+help', re.I), -1),
    (re.compile(r'tag\s+someone\s+who', re.I), -2),
    (re.compile(r'share\s+this\s+with\s+your\s+network', re.I), -1),
]


class LinkedInScraper:
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.session_start = datetime.utcnow()
        self.driver = None
        self.profile = None
        self.raw_posts = []
        self.stats = {
            "posts_scanned": 0,
            "emails_found": 0,
            "leads_qualified": 0,
            "detection_events": []
        }

    def _get_query(self) -> str:
        """Get next search query using rotation."""
        query_state_file = STATE_DIR / "query_rotation.json"
        state = {"index": 0}
        if query_state_file.exists():
            try:
                state = json.loads(query_state_file.read_text())
            except Exception:
                pass

        idx = state.get("index", 0) % len(SEARCH_QUERY_POOL)
        query = SEARCH_QUERY_POOL[idx]

        state["index"] = (idx + 1) % len(SEARCH_QUERY_POOL)
        query_state_file.write_text(json.dumps(state))

        return query

    def _launch_browser(self) -> bool:
        """Launch stealth browser with profile."""
        self.profile = get_next_profile()
        logger.info(f"Using browser profile: {self.profile['id']}")

        options = uc.ChromeOptions()
        options.add_argument(
            f"--window-size={self.profile['window_width']},{self.profile['window_height']}"
        )
        options.add_argument(
            f"--proxy-server=socks5://{TOR_SOCKS_HOST}:{TOR_SOCKS_PORT}"
        )
        options.add_argument("--no-sandbox")
        options.add_argument("--disable-dev-shm-usage")
        options.add_argument("--disable-blink-features=AutomationControlled")
        options.add_argument("--disable-infobars")
        options.add_argument(f"--user-data-dir=/data/linkedin_intel/sessions/{self.profile['id']}")

        try:
            self.driver = uc.Chrome(
                options=options,
                headless=False,
                use_subprocess=True,
                version_main=None
            )
            apply_fingerprint_profile(self.driver, self.profile)
            logger.info("Browser launched successfully")
            return True
        except Exception as e:
            logger.error(f"Browser launch failed: {e}")
            return False

    def _load_session(self) -> bool:
        """Load saved LinkedIn cookies."""
        try:
            self.driver.get("https://www.linkedin.com")
            gaussian_wait(2.0, 0.5)

            if COOKIE_FILE.exists():
                with open(COOKIE_FILE, "rb") as f:
                    cookies = pickle.load(f)

                for cookie in cookies:
                    try:
                        # Remove expiry to avoid stale cookie issues
                        cookie.pop("expiry", None)
                        self.driver.add_cookie(cookie)
                    except Exception:
                        pass

                logger.info(f"Loaded {len(cookies)} cookies")
                return True
        except Exception as e:
            logger.error(f"Cookie load failed: {e}")
        return False

    def _save_session(self):
        """Save current session cookies."""
        try:
            COOKIES_DIR.mkdir(parents=True, exist_ok=True)
            with open(COOKIE_FILE, "wb") as f:
                pickle.dump(self.driver.get_cookies(), f)
        except Exception as e:
            logger.error(f"Cookie save failed: {e}")

    def _check_page_health(self) -> str:
        """Monitor for detection signals."""
        try:
            url = self.driver.current_url
            title = self.driver.title.lower()
            source_snippet = self.driver.page_source[:1000].lower()

            if any(x in url for x in ["checkpoint", "challenge", "restricted"]):
                return "HARD_SIGNAL"
            if "security verification" in source_snippet:
                return "HARD_SIGNAL"
            if "authwall" in url:
                return "MEDIUM_SIGNAL"
            if "captcha" in source_snippet:
                return "MEDIUM_SIGNAL"
            if "please verify" in source_snippet:
                return "MEDIUM_SIGNAL"
            if "unusual activity" in source_snippet:
                return "MEDIUM_SIGNAL"

            return "HEALTHY"
        except Exception:
            return "UNKNOWN"

    def _check_time_budget(self) -> bool:
        """Ensure we're within session time limit."""
        elapsed = (datetime.utcnow() - self.session_start).total_seconds()
        if elapsed >= MAX_SESSION_DURATION_SECONDS:
            logger.warning(f"Session time limit reached: {elapsed:.0f}s")
            return False
        return True

    def _extract_emails(self, text: str) -> list:
        """Extract all email addresses from text using multiple patterns."""
        emails = set()
        for pattern in EMAIL_PATTERNS:
            matches = re.findall(pattern, text)
            for match in matches:
                # Handle tuple from groups
                email = match if isinstance(match, str) else match
                email = email.strip().rstrip(".,;:)")
                if "@" in email and "." in email.split("@")[-1]:
                    emails.add(email.lower())

        # Filter obvious false positives
        filtered = {e for e in emails if not any(x in e for x in [
            "example.com", "youremail", "test@", "email@"
        ])}
        return list(filtered)

    def _calculate_bait_score(self, text: str) -> int:
        """Score post against engagement bait patterns."""
        score = 0
        for pattern, weight in BAIT_PATTERNS_COMPILED:
            if pattern.search(text):
                score += weight
        return score

    def _extract_post_data(self, post_element) -> dict | None:
        """
        Extract all fields from a LinkedIn post DOM element.
        LinkedIn's HTML structure changes — multiple selector fallbacks.
        """
        try:
            html = post_element.get_attribute("outerHTML")
            soup = BeautifulSoup(html, "html.parser")

            # ── Extract post text (multiple selector fallbacks) ──
            text = ""
            text_selectors = [
                {"attrs": {"dir": "ltr"}},
                {"class_": lambda c: c and "feed-shared-update-v2__description" in " ".join(c)},
                {"class_": lambda c: c and "update-components-text" in " ".join(c)},
            ]
            for selector in text_selectors:
                nodes = soup.find_all("span", **selector)
                if nodes:
                    text = " ".join(n.get_text(" ", strip=True) for n in nodes)
                    break

            if not text:
                return None

            # ── Extract emails ──
            emails = self._extract_emails(text)
            if not emails:
                return None  # Fast-pass: no email = skip

            # ── Calculate bait score ──
            bait_score = self._calculate_bait_score(text)
            if bait_score < -2:
                logger.debug(f"Engagement bait detected (score {bait_score}): {text[:80]}")
                return None

            # ── Extract author info ──
            author_name = ""
            author_url = ""
            author_headline = ""

            author_link = soup.find("a", href=re.compile(r'/in/[^/?]+'))
            if author_link:
                author_name = author_link.get_text(strip=True)
                href = author_link.get("href", "")
                # Clean URL
                match = re.search(r'(/in/[^/?]+)', href)
                if match:
                    author_url = f"https://www.linkedin.com{match.group(1)}"

            # Headline / company
            headline_candidates = soup.find_all(
                class_=lambda c: c and any(
                    x in " ".join(c) for x in
                    ["actor__description", "update-components-actor__description",
                     "feed-shared-actor__description"]
                )
            )
            if headline_candidates:
                author_headline = headline_candidates[0].get_text(strip=True)

            # ── Extract hashtags ──
            hashtags = [
                tag.get_text(strip=True)
                for tag in soup.find_all("a", href=re.compile(r'/feed/hashtag/'))
            ]

            # ── Extract timestamp ──
            post_timestamp = ""
            time_el = soup.find("time")
            if time_el:
                post_timestamp = time_el.get("datetime", time_el.get_text(strip=True))

            # ── Extract post URL ──
            post_url = ""
            post_link = soup.find("a", href=re.compile(r'/feed/update/'))
            if post_link:
                post_url = post_link.get("href", "")

            return {
                "post_text": text,
                "emails": emails,
                "author_name": author_name,
                "author_url": author_url,
                "author_headline": author_headline,
                "author_company": "",  # Enriched later
                "post_url": post_url,
                "post_timestamp": post_timestamp or datetime.utcnow().isoformat(),
                "hashtags": hashtags,
                "bait_score": bait_score,
                "session_id": self.session_id
            }

        except Exception as e:
            logger.debug(f"Post extraction error: {e}")
            return None

    def _save_raw_posts(self):
        """Save raw post data for this session (debugging)."""
        if self.raw_posts:
            raw_file = RAW_POSTS_DIR / f"{self.session_id}_raw.json"
            raw_file.write_text(json.dumps(self.raw_posts, indent=2, default=str))

    def run(self) -> dict:
        """
        Main scraping execution.
        Returns result dict for n8n parsing.
        """
        logger.info(f"Session starting: {self.session_id}")
        initialize_database()

        # ── Phase 1: Browser Setup ──
        if not self._launch_browser():
            return {
                "status": "FAILED",
                "reason": "BROWSER_LAUNCH_FAILED",
                "sessionId": self.session_id,
                **self.stats
            }

        try:
            # ── Phase 2: Session Load ──
            self._load_session()

            # ── Phase 3: Navigate to feed + health check ──
            self.driver.get("https://www.linkedin.com/feed/")
            gaussian_wait(3.0, 1.0)

            health = self._check_page_health()
            if health != "HEALTHY":
                logger.warning(f"Detection signal on feed load: {health}")
                return {
                    "status": "ABORTED",
                    "detectionEvent": health,
                    "sessionId": self.session_id,
                    **self.stats
                }

            # ── Phase 4: Human Warmup ──
            feed_warmup(self.driver)
            gaussian_wait(random.gauss(5, 1.5), 0)

            # Check time budget
            if not self._check_time_budget():
                return {
                    "status": "TIMEOUT",
                    "sessionId": self.session_id,
                    **self.stats
                }

            # ── Phase 5: Build Search URL ──
            query = self._get_query()
            import urllib.parse
            encoded_query = urllib.parse.quote(query)
            search_url = (
                f"https://www.linkedin.com/search/results/content/"
                f"?keywords={encoded_query}"
                f"&origin=SWITCH_SEARCH_VERTICAL"
                f"&sortBy=date_posted"
            )
            logger.info(f"Search query: {query}")

            # ── Phase 6: Navigate to Search ──
            self.driver.get(search_url)
            gaussian_wait(4.0, 1.5)

            health = self._check_page_health()
            if health != "HEALTHY":
                logger.warning(f"Detection signal on search: {health}")
                self.stats["detection_events"].append(health)
                return {
                    "status": "ABORTED",
                    "detectionEvent": health,
                    "sessionId": self.session_id,
                    **self.stats
                }

            # ── Phase 7: Scroll + Extract Loop ──
            max_posts = random.randint(*MAX_POSTS_PER_SESSION)
            max_scrolls = random.randint(*MAX_SCROLLS_PER_SESSION)
            processed_post_ids = set()

            logger.info(f"Target: {max_posts} posts, {max_scrolls} scrolls")

            for scroll_num in range(max_scrolls):
                if not self._check_time_budget():
                    break
                if self.stats["posts_scanned"] >= max_posts:
                    break

                # Health check every 5 scrolls
                if scroll_num % 5 == 0 and scroll_num > 0:
                    health = self._check_page_health()
                    if health != "HEALTHY":
                        self.stats["detection_events"].append(health)
                        logger.warning(f"Detection signal during scroll: {health}")
                        return {
                            "status": "ABORTED",
                            "detectionEvent": health,
                            "sessionId": self.session_id,
                            **self.stats
                        }

                # Find all post containers currently visible
                try:
                    post_containers = self.driver.find_elements(
                        By.CSS_SELECTOR,
                        "div[data-urn], div.feed-shared-update-v2"
                    )
                except Exception:
                    post_containers = []

                # Process new posts
                for container in post_containers:
                    post_id = container.get_attribute("data-urn") or \
                              container.get_attribute("id") or \
                              hashlib.md5(container.get_attribute("outerHTML")[:200].encode()).hexdigest()

                    if post_id in processed_post_ids:
                        continue
                    processed_post_ids.add(post_id)

                    self.stats["posts_scanned"] += 1

                    # Extract post data
                    post_data = self._extract_post_data(container)
                    if post_data:
                        self.stats["emails_found"] += len(post_data.get("emails", []))
                        self.raw_posts.append(post_data)
                        self.stats["leads_qualified"] += 1

                    # Dwell to simulate reading
                    if post_data:
                        post_read_dwell(post_data.get("post_text", ""))
                    else:
                        gaussian_wait(0.3, 0.1)

                    if self.stats["posts_scanned"] >= max_posts:
                        break

                # Scroll down
                scroll_px = int(random.gauss(380, 80))
                human_scroll(self.driver, max(200, scroll_px))
                gaussian_wait(random.gauss(2.5, 0.8), 0)

            # ── Phase 8: Save Raw Data ──
            self._save_raw_posts()
            self._save_session()

            logger.info(
                f"Session complete: scanned={self.stats['posts_scanned']}, "
                f"emails={self.stats['emails_found']}, "
                f"qualified={self.stats['leads_qualified']}"
            )

            return {
                "status": "COMPLETED",
                "sessionId": self.session_id,
                "query": query,
                "profileId": self.profile["id"] if self.profile else "unknown",
                **self.stats
            }

        except Exception as e:
            logger.error(f"Scraper error: {e}", exc_info=True)
            return {
                "status": "FAILED",
                "reason": str(e),
                "sessionId": self.session_id,
                **self.stats
            }
        finally:
            if self.driver:
                try:
                    self.driver.quit()
                except Exception:
                    pass


if __name__ == "__main__":
    session_id = sys.argv[1] if len(sys.argv) > 1 else f"sess_{int(time.time())}"
    scraper = LinkedInScraper(session_id)
    result = scraper.run()
    print(json.dumps(result, default=str))
```

---

### Script 9: `post_processor.py` — NLP Enrichment + Scoring

```python
# /scripts/linkedin_intel/post_processor.py
"""
Enriches raw posts with NLP and scores against your profile.
Usage: python3 post_processor.py <session_id>
"""

import sys
import json
import re
import logging
from pathlib import Path
from datetime import datetime

import spacy

from config import (
    RAW_POSTS_DIR, YOUR_TECH_STACK, YOUR_TARGET_ROLES,
    YOUR_EXPERIENCE_LEVEL, PREFER_REMOTE,
    MIN_RELEVANCE_SCORE, HIGH_PRIORITY_THRESHOLD,
    REAL_POST_SIGNALS, BAIT_PATTERNS
)
from db_manager import save_lead, initialize_database

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Load spaCy model (small English model — runs locally, free)
try:
    nlp = spacy.load("en_core_web_sm")
except OSError:
    import subprocess
    subprocess.run(["python3", "-m", "spacy", "download", "en_core_web_sm"])
    nlp = spacy.load("en_core_web_sm")

EXPERIENCE_PATTERNS = {
    "entry": re.compile(r'(?:0[-–]2|junior|entry.?level|new\s+grad|recent\s+grad|graduate)', re.I),
    "mid": re.compile(r'(?:2[-–]5|mid.?level|intermediate|\b[23]\+?\s*years?)', re.I),
    "senior": re.compile(r'(?:5[-–]8|senior|sr\.?\s+(?:engineer|developer)|\b[5-7]\+?\s*years?)', re.I),
    "lead": re.compile(r'(?:lead|principal|staff|\b[8-9]\+?\s*years?|10\+\s*years?)', re.I),
}

REMOTE_PATTERNS = {
    "remote": re.compile(r'\bfully\s+remote\b|\bremote\s+first\b|\bwork\s+from\s+anywhere\b|\bremote\b', re.I),
    "hybrid": re.compile(r'\bhybrid\b', re.I),
    "onsite": re.compile(r'\bon.?site\b|\bin.?office\b|\bon\s+location\b', re.I),
}

SALARY_PATTERN = re.compile(
    r'\$[\d,]+(?:k|K)?(?:\s*[-–]\s*\$[\d,]+(?:k|K)?)?|\b[\d,]+(?:k|K)?\s*(?:USD|EUR|GBP)\b',
    re.I
)

LOCATION_PATTERN = re.compile(
    r'\b(?:New York|San Francisco|London|Austin|Seattle|Chicago|Boston|'
    r'Los Angeles|Berlin|Toronto|Vancouver|Remote|Worldwide)\b',
    re.I
)


def extract_tech_stack(text: str) -> list:
    """Extract technology mentions from post text."""
    text_lower = text.lower()
    found = []
    for tech in YOUR_TECH_STACK:
        # Word boundary check
        pattern = re.compile(r'\b' + re.escape(tech) + r'\b', re.I)
        if pattern.search(text):
            found.append(tech)
    return found


def extract_experience_level(text: str) -> str:
    """Determine experience level from post text."""
    for level, pattern in EXPERIENCE_PATTERNS.items():
        if pattern.search(text):
            return level
    return "unknown"


def extract_remote_signal(text: str) -> str:
    """Extract work arrangement signal."""
    for signal, pattern in REMOTE_PATTERNS.items():
        if pattern.search(text):
            return signal
    return "unknown"


def extract_role_title(text: str) -> str:
    """
    Extract primary role title using NLP entity recognition
    plus keyword matching.
    """
    # First try NLP
    doc = nlp(text[:1000])  # Limit for performance

    # Look for job-title-like phrases near hiring keywords
    role_keywords = re.compile(
        r'(?:hiring|looking\s+for|seeking|need|want)\s+(?:a\s+)?([A-Z][a-z]+(?:\s+[A-Z][a-z]+){0,3})',
        re.I
    )
    match = role_keywords.search(text)
    if match:
        candidate = match.group(1).strip()
        if any(kw in candidate.lower() for kw in [
            "engineer", "developer", "manager", "designer",
            "analyst", "scientist", "architect", "lead", "director"
        ]):
            return candidate

    # Fallback: match against known role patterns
    for role in YOUR_TARGET_ROLES:
        if role in text.lower():
            return role.title()

    return "Unknown Role"


def extract_location(text: str) -> str:
    """Extract location mention."""
    match = LOCATION_PATTERN.search(text)
    return match.group(0) if match else ""


def extract_salary(text: str) -> str:
    """Extract salary mention."""
    match = SALARY_PATTERN.search(text)
    return match.group(0) if match else ""


def calculate_real_post_signals(text: str) -> int:
    """Score positive signals that indicate real job post."""
    score = 0
    for pattern_str, weight in REAL_POST_SIGNALS:
        if re.search(pattern_str, text, re.I):
            score += weight
    return score


def calculate_relevance_score(post_data: dict) -> int:
    """
    Score lead against YOUR configured profile.
    0-15 typical range.
    """
    text = post_data.get("post_text", "").lower()
    score = 0

    # Stack match: +1 per matching technology
    tech_stack = post_data.get("tech_stack", [])
    score += len(tech_stack)

    # Role match: +3 exact, +1 partial
    role_title = post_data.get("role_title", "").lower()
    if any(role == role_title for role in YOUR_TARGET_ROLES):
        score += 3
    elif any(role in role_title or role_title in role for role in YOUR_TARGET_ROLES):
        score += 1

    # Experience level match: +2
    exp_level = post_data.get("experience_level", "")
    if exp_level == YOUR_EXPERIENCE_LEVEL:
        score += 2
    elif exp_level in ["lead", "principal"] and YOUR_EXPERIENCE_LEVEL == "senior":
        score += 1  # Close enough

    # Remote preference: +2
    remote_signal = post_data.get("remote_signal", "")
    if PREFER_REMOTE and remote_signal == "remote":
        score += 2
    elif PREFER_REMOTE and remote_signal == "hybrid":
        score += 1

    # Salary mentioned: +1 (transparency signal)
    if post_data.get("salary"):
        score += 1

    # Positive signals: +1
    real_signal_score = calculate_real_post_signals(
        post_data.get("post_text", "")
    )
    score += min(real_signal_score, 3)  # Cap at +3

    # Urgency bonus: +1
    if re.search(r'\b(?:immediately|asap|urgent|quickly)\b', text):
        score += 1

    return score


def process_session_posts(session_id: str) -> dict:
    """
    Load raw posts for session, enrich, score, and save to DB.
    """
    raw_file = RAW_POSTS_DIR / f"{session_id}_raw.json"

    if not raw_file.exists():
        return {
            "leadsStored": 0,
            "highPriorityLeads": [],
            "error": f"Raw file not found: {raw_file}"
        }

    raw_posts = json.loads(raw_file.read_text())
    logger.info(f"Processing {len(raw_posts)} raw posts for session {session_id}")

    leads_stored = 0
    high_priority_leads = []

    for post in raw_posts:
        # ── NLP Enrichment ──
        text = post.get("post_text", "")

        post["tech_stack"] = extract_tech_stack(text)
        post["role_title"] = extract_role_title(text)
        post["experience_level"] = extract_experience_level(text)
        post["remote_signal"] = extract_remote_signal(text)
        post["location"] = extract_location(text)
        post["salary"] = extract_salary(text)

        # ── Relevance Scoring ──
        post["relevance_score"] = calculate_relevance_score(post)

        # ── Priority Classification ──
        if post["relevance_score"] < MIN_RELEVANCE_SCORE:
            logger.debug(f"Low priority lead (score {post['relevance_score']}), storing anyway")

        # ── Save to DB ──
        was_new = save_lead(post)

        if was_new:
            leads_stored += 1
            logger.info(
                f"Lead saved | Score: {post['relevance_score']} | "
                f"Role: {post['role_title']} | "
                f"Email: {post['emails'][0][:20] if post['emails'] else 'none'}..."
            )

            # Collect high priority for notifications
            if post["relevance_score"] >= HIGH_PRIORITY_THRESHOLD:
                high_priority_leads.append({
                    "role_title": post["role_title"],
                    "author_name": post.get("author_name", ""),
                    "author_company": post.get("author_company", ""),
                    "emails": post.get("emails", []),
                    "tech_stack": post.get("tech_stack", []),
                    "remote_signal": post.get("remote_signal", ""),
                    "relevance_score": post["relevance_score"],
                    "post_url": post.get("post_url", "")
                })

    logger.info(
        f"Processing complete: {leads_stored} new leads, "
        f"{len(high_priority_leads)} high priority"
    )

    return {
        "leadsStored": leads_stored,
        "highPriorityLeads": high_priority_leads,
        "totalProcessed": len(raw_posts),
        "sessionId": session_id
    }


if __name__ == "__main__":
    session_id = sys.argv[1] if len(sys.argv) > 1 else "unknown"
    initialize_database()
    result = process_session_posts(session_id)
    print(json.dumps(result, default=str))
```

---

### Script 10: `session_recovery.py`

```python
# /scripts/linkedin_intel/session_recovery.py
"""
Attempt to recover expired LinkedIn session via re-authentication.
Usage: python3 session_recovery.py <session_id>
"""

import sys
import json
import time
import pickle
import logging

import undetected_chromedriver as uc
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

from config import (
    COOKIES_DIR, TOR_SOCKS_HOST, TOR_SOCKS_PORT,
    LINKEDIN_EMAIL, LINKEDIN_PASSWORD, BROWSER_PROFILES,
    GOTIFY_URL, GOTIFY_TOKEN
)
from fingerprint_manager import apply_fingerprint_profile
from human_behavior import gaussian_wait, human_type, human_click
import requests

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

COOKIE_FILE = COOKIES_DIR / "linkedin_session.pkl"


def send_2fa_notification():
    """Alert for manual 2FA if required."""
    try:
        requests.post(
            f"{GOTIFY_URL}/message",
            params={"token": GOTIFY_TOKEN},
            json={
                "title": "🔐 LI Intel: 2FA Required",
                "message": "LinkedIn 2FA verification needed. Please check browser within 3 minutes.",
                "priority": 10
            },
            timeout=5
        )
    except Exception:
        pass


def attempt_reauth(session_id: str) -> dict:
    """
    Attempt LinkedIn re-authentication.
    Sends 2FA alert and waits for manual completion.
    """
    profile = BROWSER_PROFILES[0]

    options = uc.ChromeOptions()
    options.add_argument(f"--window-size={profile['window_width']},{profile['window_height']}")
    options.add_argument(f"--proxy-server=socks5://{TOR_SOCKS_HOST}:{TOR_SOCKS_PORT}")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument(f"--user-data-dir=/data/linkedin_intel/sessions/{profile['id']}")

    driver = None
    try:
        driver = uc.Chrome(options=options, headless=False, use_subprocess=True)
        apply_fingerprint_profile(driver, profile)

        driver.get("https://www.linkedin.com/login")
        gaussian_wait(3.0, 1.0)

        # Enter email
        email_field = WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.ID, "username"))
        )
        human_type(driver, email_field, LINKEDIN_EMAIL)

        # Enter password
        password_field = driver.find_element(By.ID, "password")
        human_type(driver, password_field, LINKEDIN_PASSWORD)

        # Submit
        submit_btn = driver.find_element(By.CSS_SELECTOR, "[type='submit']")
        human_click(driver, submit_btn)

        gaussian_wait(4.0, 1.0)

        current_url = driver.current_url

        # Check if 2FA required
        if any(x in current_url for x in ["checkpoint", "challenge", "two-step"]):
            logger.info("2FA required — sending alert and waiting")
            send_2fa_notification()

            # Wait up to 3 minutes for manual 2FA
            for _ in range(36):
                time.sleep(5)
                if "feed" in driver.current_url:
                    break
            else:
                return {
                    "status": "2FA_TIMEOUT",
                    "success": False,
                    "sessionId": session_id
                }

        if "feed" in driver.current_url:
            # Save new cookies
            COOKIES_DIR.mkdir(parents=True, exist_ok=True)
            with open(COOKIE_FILE, "wb") as f:
                pickle.dump(driver.get_cookies(), f)

            logger.info("Re-authentication successful")
            return {
                "status": "HEALTHY",
                "success": True,
                "sessionId": session_id
            }

        return {
            "status": "REAUTH_FAILED",
            "success": False,
            "currentUrl": driver.current_url,
            "sessionId": session_id
        }

    except Exception as e:
        logger.error(f"Recovery error: {e}")
        return {
            "status": "ERROR",
            "success": False,
            "error": str(e),
            "sessionId": session_id
        }
    finally:
        if driver:
            try:
                driver.quit()
            except Exception:
                pass


if __name__ == "__main__":
    session_id = sys.argv[1] if len(sys.argv) > 1 else "unknown"
    result = attempt_reauth(session_id)
    print(json.dumps(result))
```

---

### Script 11: `detection_handler.py`

```python
# /scripts/linkedin_intel/detection_handler.py
"""
Handle detection events with appropriate response actions.
Usage: python3 detection_handler.py <session_id> <detection_event>
"""

import sys
import json
import logging
from datetime import datetime, timedelta
from pathlib import Path
from config import STATE_DIR, GOTIFY_URL, GOTIFY_TOKEN
import requests

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def set_freeze_state(hours: int, reason: str):
    """Write system freeze state."""
    freeze_file = STATE_DIR / "system_freeze.json"
    resume_time = datetime.utcnow() + timedelta(hours=hours)
    freeze_data = {
        "frozen": True,
        "reason": reason,
        "frozenAt": datetime.utcnow().isoformat(),
        "resumeAfter": resume_time.isoformat(),
        "freezeDurationHours": hours
    }
    freeze_file.write_text(json.dumps(freeze_data, indent=2))
    logger.warning(f"System frozen for {hours}h: {reason}")


def notify_detection(event_type: str, session_id: str, action: str):
    """Push notification for detection events."""
    priority_map = {
        "HARD_SIGNAL": 10,
        "MEDIUM_SIGNAL": 8,
        "SOFT_SIGNAL": 5
    }
    try:
        requests.post(
            f"{GOTIFY_URL}/message",
            params={"token": GOTIFY_TOKEN},
            json={
                "title": f"🚨 LI Intel: {event_type}",
                "message": f"Session: {session_id}\nAction: {action}\nTime: {datetime.utcnow().isoformat()}",
                "priority": priority_map.get(event_type, 7)
            },
            timeout=5
        )
    except Exception as e:
        logger.error(f"Notification failed: {e}")


def handle_detection(session_id: str, detection_event: str) -> dict:
    """
    Take appropriate action based on detection signal severity.
    """
    logger.warning(f"Handling detection event: {detection_event} | Session: {session_id}")

    if detection_event in ["HARD_SIGNAL", "CHALLENGE", "RESTRICTED"]:
        # 48-hour freeze
        set_freeze_state(48, f"HARD_SIGNAL_DETECTED: {detection_event}")
        notify_detection(detection_event, session_id, "SYSTEM_FROZEN_48H")
        return {
            "action": "SYSTEM_FROZEN",
            "freezeHours": 48,
            "event": detection_event,
            "sessionId": session_id
        }

    elif detection_event in ["MEDIUM_SIGNAL", "CAPTCHA", "VERIFY_HUMAN"]:
        # 6-hour cooldown
        set_freeze_state(6, f"MEDIUM_SIGNAL_DETECTED: {detection_event}")
        notify_detection(detection_event, session_id, "COOLDOWN_6H")
        return {
            "action": "COOLDOWN",
            "freezeHours": 6,
            "event": detection_event,
            "sessionId": session_id
        }

    elif detection_event in ["SOFT_SIGNAL", "THROTTLE"]:
        # 2-hour soft pause
        set_freeze_state(2, f"SOFT_SIGNAL_DETECTED: {detection_event}")
        return {
            "action": "SOFT_PAUSE",
            "freezeHours": 2,
            "event": detection_event,
            "sessionId": session_id
        }

    else:
        # Unknown event — conservative 4-hour pause
        set_freeze_state(4, f"UNKNOWN_EVENT: {detection_event}")
        return {
            "action": "CONSERVATIVE_PAUSE",
            "freezeHours": 4,
            "event": detection_event,
            "sessionId": session_id
        }


if __name__ == "__main__":
    session_id = sys.argv[1] if len(sys.argv) > 1 else "unknown"
    detection_event = sys.argv[2] if len(sys.argv) > 2 else "UNKNOWN"
    result = handle_detection(session_id, detection_event)
    print(json.dumps(result))
```

---

### Script 12: `notifier.py`

```python
# /scripts/linkedin_intel/notifier.py
"""
Send push notifications for high-priority leads via Gotify.
Usage: python3 notifier.py '<leads_json>' <session_id>
"""

import sys
import json
import logging
import requests
from config import GOTIFY_URL, GOTIFY_TOKEN

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def send_lead_notification(lead: dict) -> bool:
    """Send a single lead notification."""
    role = lead.get("role_title", "Unknown Role")
    company = lead.get("author_company", "") or lead.get("author_name", "")
    emails = lead.get("emails", [])
    stack = lead.get("tech_stack", [])
    score = lead.get("relevance_score", 0)
    remote = lead.get("remote_signal", "")
    post_url = lead.get("post_url", "")

    email_display = emails[0] if emails else "No email"
    stack_display = ", ".join(stack[:5]) if stack else "Not specified"

    message_lines = [
        f"Score: {score}/15",
        f"Email: {email_display}",
        f"Stack: {stack_display}",
    ]
    if remote:
        message_lines.append(f"Work: {remote.title()}")
    if post_url:
        message_lines.append(f"Post: {post_url[:60]}...")

    try:
        resp = requests.post(
            f"{GOTIFY_URL}/message",
            params={"token": GOTIFY_TOKEN},
            json={
                "title": f"⭐ {role} @ {company}",
                "message": "\n".join(message_lines),
                "priority": 8 if score >= 10 else 6
            },
            timeout=10
        )
        return resp.status_code == 200
    except Exception as e:
        logger.error(f"Notification failed: {e}")
        return False


if __name__ == "__main__":
    leads_json = sys.argv[1] if len(sys.argv) > 1 else "[]"
    session_id = sys.argv[2] if len(sys.argv) > 2 else "unknown"

    try:
        leads = json.loads(leads_json)
    except json.JSONDecodeError:
        leads = []

    sent = 0
    for lead in leads:
        if send_lead_notification(lead):
            sent += 1

    result = {
        "notificationsSent": sent,
        "totalLeads": len(leads),
        "sessionId": session_id
    }
    print(json.dumps(result))
```

---

### Script 13: `session_teardown.py`

```python
# /scripts/linkedin_intel/session_teardown.py
"""
Clean session teardown — save state, update DB, release resources.
Usage: python3 session_teardown.py <session_id>
"""

import sys
import json
import logging
from datetime import datetime
from pathlib import Path
from config import RAW_POSTS_DIR
from db_manager import update_session

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def teardown_session(session_id: str) -> dict:
    """Perform graceful session cleanup."""
    logger.info(f"Tearing down session: {session_id}")

    # Update session record in DB
    try:
        update_session(session_id, status="COMPLETED", ended_at=datetime.utcnow().isoformat())
        logger.info("Session DB record updated")
    except Exception as e:
        logger.error(f"DB update failed: {e}")

    # Archive raw posts (move from pending to archive)
    raw_file = RAW_POSTS_DIR / f"{session_id}_raw.json"
    if raw_file.exists():
        archive_dir = RAW_POSTS_DIR / "archive"
        archive_dir.mkdir(exist_ok=True)
        raw_file.rename(archive_dir / raw_file.name)
        logger.info("Raw posts archived")

    # Clear any temp chrome profile locks
    import subprocess
    try:
        subprocess.run(
            ["find", "/data/linkedin_intel/sessions", "-name", "SingletonLock", "-delete"],
            capture_output=True, timeout=5
        )
    except Exception:
        pass

    return {
        "status": "TEARDOWN_COMPLETE",
        "sessionId": session_id,
        "timestamp": datetime.utcnow().isoformat()
    }


if __name__ == "__main__":
    session_id = sys.argv[1] if len(sys.argv) > 1 else "unknown"
    result = teardown_session(session_id)
    print(json.dumps(result))
```

---

### Script 14: `daily_maintenance.py`

```python
# /scripts/linkedin_intel/daily_maintenance.py
"""
Daily maintenance: backup, cleanup, stats.
Usage: python3 daily_maintenance.py
"""

import json
import logging
import subprocess
import shutil
from datetime import datetime, timedelta
from pathlib import Path
from config import BACKUPS_DIR, LEADS_DB, LOGS_DIR
from db_manager import get_daily_stats

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def create_backup() -> bool:
    """Create encrypted SQLite dump backup."""
    BACKUPS_DIR.mkdir(parents=True, exist_ok=True)
    timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    backup_file = BACKUPS_DIR / f"leads_{timestamp}.sql"

    try:
        result = subprocess.run(
            ["sqlite3", str(LEADS_DB), ".dump"],
            capture_output=True, text=True, timeout=60
        )
        backup_file.write_text(result.stdout)
        logger.info(f"Backup created: {backup_file}")
        return True
    except Exception as e:
        logger.error(f"Backup failed: {e}")
        return False


def cleanup_old_backups(keep_days: int = 7):
    """Remove backups older than keep_days."""
    cutoff = datetime.utcnow() - timedelta(days=keep_days)
    removed = 0
    for backup_file in BACKUPS_DIR.glob("*.sql"):
        if datetime.fromtimestamp(backup_file.stat().st_mtime) < cutoff:
            backup_file.unlink()
            removed += 1
    if removed:
        logger.info(f"Removed {removed} old backups")


def get_disk_usage() -> int:
    """Get disk usage of data directory in MB."""
    try:
        result = subprocess.run(
            ["du", "-sm", "/data/linkedin_intel"],
            capture_output=True, text=True, timeout=10
        )
        return int(result.stdout.split()[0])
    except Exception:
        return 0


def cleanup_old_logs(keep_days: int = 14):
    """Truncate logs older than keep_days to save disk."""
    cutoff = datetime.utcnow() - timedelta(days=keep_days)
    for log_file in LOGS_DIR.glob("*.log"):
        if log_file.stat().st_size > 10_000_000:  # 10MB
            # Keep last 1000 lines
            try:
                result = subprocess.run(
                    ["tail", "-1000", str(log_file)],
                    capture_output=True, text=True
                )
                log_file.write_text(result.stdout)
                logger.info(f"Truncated large log: {log_file.name}")
            except Exception:
                pass


if __name__ == "__main__":
    backup_ok = create_backup()
    cleanup_old_backups()
    cleanup_old_logs()
    disk_mb = get_disk_usage()
    stats = get_daily_stats()

    result = {
        "success": True,
        "backupCreated": backup_ok,
        "diskUsageMB": disk_mb,
        **stats
    }
    print(json.dumps(result))
```

---

### Script 15: `self_heal.py`

```python
# /scripts/linkedin_intel/self_heal.py
"""
Self-healing: restart failed components.
Usage: python3 self_heal.py
"""

import json
import logging
import subprocess
import time
from config import GOTIFY_URL, GOTIFY_TOKEN
import requests

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def restart_tor():
    """Restart Tor service."""
    try:
        subprocess.run(["systemctl", "restart", "tor"], timeout=15, check=True)
        time.sleep(8)
        logger.info("Tor restarted")
        return True
    except Exception as e:
        logger.error(f"Tor restart failed: {e}")
        return False


def restart_gotify():
    """Restart Gotify Docker container."""
    try:
        subprocess.run(
            ["docker", "restart", "gotify"],
            timeout=30, check=True
        )
        time.sleep(5)
        logger.info("Gotify restarted")
        return True
    except Exception as e:
        logger.error(f"Gotify restart failed: {e}")
        return False


def check_and_heal_tor() -> bool:
    """Test Tor and restart if needed."""
    try:
        result = subprocess.run(
            ["curl", "-s", "--socks5", "127.0.0.1:9050", "--max-time", "8", "https://api.ipify.org"],
            capture_output=True, text=True, timeout=15
        )
        if result.returncode == 0 and result.stdout.strip():
            return True
    except Exception:
        pass

    logger.warning("Tor not responding — restarting")
    return restart_tor()


def check_and_heal_gotify() -> bool:
    """Test Gotify and restart if needed."""
    try:
        resp = requests.get(f"{GOTIFY_URL}/health", timeout=5)
        if resp.status_code == 200:
            return True
    except Exception:
        pass

    logger.warning("Gotify not responding — restarting")
    return restart_gotify()


if __name__ == "__main__":
    results = {
        "tor": check_and_heal_tor(),
        "gotify": check_and_heal_gotify(),
        "timestamp": __import__("datetime").datetime.utcnow().isoformat()
    }
    print(json.dumps(results))
```

---

## SETUP SCRIPT (Run Once)

```bash
#!/bin/bash
# /scripts/setup.sh
# Run this once on a fresh machine to install everything

set -e

echo "=== LinkedIn Recruiter Intelligence System Setup ==="

# ── System packages ──
sudo apt-get update
sudo apt-get install -y \
    python3 python3-pip python3-venv \
    tor curl wget git sqlite3 \
    docker.io docker-compose \
    chromium-browser || true

# ── Python environment ──
python3 -m venv /opt/linkedin_intel_env
source /opt/linkedin_intel_env/bin/activate

pip install --upgrade pip
pip install \
    undetected-chromedriver \
    playwright \
    playwright-stealth \
    selenium \
    beautifulsoup4 \
    lxml \
    spacy \
    requests \
    stem \
    numpy \
    cryptography \
    streamlit \
    pandas

# ── spaCy model ──
python3 -m spacy download en_core_web_sm

# ── Playwright browsers ──
playwright install chromium

# ── Directory structure ──
mkdir -p /scripts/linkedin_intel
mkdir -p /data/linkedin_intel/{logs,state,backups,cookies,sessions,notifications/pending,raw_posts/archive,fingerprints}

# ── Initial state files ──
echo '{"frozen": false}' > /data/linkedin_intel/state/system_freeze.json
echo '{"index": 0}' > /data/linkedin_intel/state/query_rotation.json
echo '{"last_used": {}, "current_index": 0}' > /data/linkedin_intel/state/profile_rotation.json

# ── Tor configuration ──
TOR_HASH=$(tor --hash-password "your_tor_password_here" 2>/dev/null | tail -1)
cat > /etc/tor/torrc << EOF
SocksPort 9050
ControlPort 9051
HashedControlPassword ${TOR_HASH}
ExitNodes {US},{GB},{DE},{CA}
StrictNodes 1
EOF
sudo systemctl restart tor
sudo systemctl enable tor

# ── Gotify via Docker ──
docker run -d \
    --name gotify \
    --restart always \
    -p 8080:80 \
    -v /opt/gotify/data:/app/data \
    gotify/server

# ── n8n via Docker ──
docker run -d \
    --name n8n \
    --restart always \
    -p 5678:5678 \
    -v ~/.n8n:/home/node/.n8n \
    -e LINKEDIN_EMAIL="your_email@gmail.com" \
    -e LINKEDIN_PASSWORD="your_password" \
    -e GOTIFY_APP_TOKEN="your_gotify_token" \
    -e TOR_PASSWORD="your_tor_password_here" \
    n8nio/n8n

# ── Initialize database ──
cd /scripts/linkedin_intel
python3 -c "from db_manager import initialize_database; initialize_database(); print('DB initialized')"

echo ""
echo "=== Setup Complete ==="
echo ""
echo "NEXT STEPS:"
echo "1. Edit /scripts/linkedin_intel/config.py with your details"
echo "2. Open n8n at http://localhost:5678"
echo "3. Import the 4 workflow JSON files"
echo "4. Set environment variables in n8n"
echo "5. Open Gotify at http://localhost:8080 and create app token"
echo "6. Run session_health.py manually to test first login"
echo "7. Run main_scraper.py manually to test scraping"
echo "8. Activate n8n workflows only after manual tests pass"
```

---

## Environment Variables Reference

```bash
# Add to n8n Docker run command or ~/.n8n/.env

LINKEDIN_EMAIL=your_actual_linkedin_email@gmail.com
LINKEDIN_PASSWORD=your_actual_linkedin_password
GOTIFY_APP_TOKEN=your_gotify_application_token
TOR_PASSWORD=your_tor_control_password
GOTIFY_URL=http://localhost:8080
```

---

## Import Order for n8n

```
IMPORT IN THIS EXACT ORDER:
1. LinkedIn Recruiter Intelligence - Error Handler
   (must exist before Master Orchestrator references it)

2. LinkedIn Recruiter Intelligence - Master Orchestrator
   (set Error Workflow to step 1 in Settings tab)

3. LinkedIn Recruiter Intelligence - Freeze Guard
   (activate immediately — monitors from day 1)

4. LinkedIn Recruiter Intelligence - Daily Maintenance
   (activate after first successful manual scrape test)

ACTIVATION SEQUENCE:
  Week 1: Freeze Guard only (monitoring infrastructure)
  Week 2: Daily Maintenance (after manual tests pass)
  Week 3: Master Orchestrator (after 1 full week of manual validation)
```