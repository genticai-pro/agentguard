# Integration Examples

## Basic: Add RetryClassifier to Any AI Agent Workflow

Your existing workflow:
```
[Webhook] → [AI Agent] → [Tool Execution] → [Response]
```

With AgentGuard:
```
[Webhook] → [AI Agent] → [Tool Execution] → [RetryClassifier] → [Response]
                                    ↑                    |
                                    └────── retry ───────┘
```

### Step-by-Step

1. Import `workflows/retry-classifier.json` into n8n
2. In your existing workflow, add an **Execute Workflow** node after your HTTP/API call
3. Point it at the RetryClassifier workflow
4. Connect the "retry" output back to your API call node
5. Connect the "failed" output to your error handler

### Passing Error Data

The RetryClassifier expects this input shape:

```json
{
  "statusCode": 429,
  "headers": {
    "retry-after": "5"
  },
  "body": {
    "error": {
      "message": "Rate limit exceeded"
    }
  },
  "_agRetryAttempt": 0,
  "_ag529Count": 0
}
```

Map your HTTP Request node's error output to this format using a Code node:

```javascript
const response = $input.first().json;
return [{
  json: {
    statusCode: response.statusCode || response.status || 0,
    headers: response.headers || {},
    body: response.body || response.data || {},
    _agRetryAttempt: $json._agRetryAttempt || 0,
    _ag529Count: $json._ag529Count || 0
  }
}];
```

---

## Intermediate: Context Budget + AI Agent Loop

```
[Trigger] → [AI Agent] → [Tool Call]
                 ↑              |
                 |        [ContextBudget]
                 |              |
                 └──── loop ────┘
```

Add a Code node before the AI Agent that calls ContextBudget:

```javascript
// Before each AI Agent iteration
const history = $json.conversationHistory || [];
const toolResults = $json.lastToolResults || [];

// Call ContextBudget sub-workflow
return [{
  json: {
    toolResults,
    conversationHistory: history,
    turnNumber: $json.turnNumber || 0
  }
}];
```

The ContextBudget returns budgeted results and compacted history.
Feed these back into your AI Agent node's messages.

---

## Advanced: Full Stack (All Three Components)

```
[Trigger]
    |
    ▼
[AI Agent] ←──────────────────────────┐
    |                                  |
    ▼                                  |
[PermissionGate] ── denied ──→ [Error] |
    |                                  |
    | approved                         |
    ▼                                  |
[Tool Execution]                       |
    |                                  |
    ├── success ──→ [ContextBudget] ───┘
    |
    └── error ──→ [RetryClassifier]
                       |
                       ├── retry ──→ [Tool Execution]
                       └── fail ──→ [Error Handler]
```

This gives you:
- **Permission check** before every tool call
- **Result budgeting** after every tool response
- **Context compaction** when window fills up
- **Classified retry** on every API error
- **Audit log** of every permission decision

---

## Model Fallback Chain Example

Configure RetryClassifier to cascade through your models:

```javascript
// In RetryClassifier config
const config = {
  fallbackChain: [
    { model: 'claude-haiku-4.5',   endpoint: 'https://api.anthropic.com/v1/messages' },
    { model: 'gemini-flash-lite',  endpoint: 'https://generativelanguage.googleapis.com/v1beta/...' },
    { model: 'gemma4:e4b',         endpoint: 'http://localhost:11434/v1/chat/completions' }
  ]
};
```

First failure → try next model. All cloud fails → fall back to local Ollama at $0. Your agent keeps working even during API outages.

---

## Monitoring with Langfuse

If you're running Langfuse, add trace logging to each component:

```javascript
// After any AgentGuard decision
const trace = {
  name: 'agentguard_decision',
  metadata: {
    component: 'RetryClassifier',
    errorClass: $json.errorClass,
    action: $json.action,
    attempt: $json._agRetryAttempt,
    timestamp: $json._agTimestamp
  }
};

// POST to your Langfuse instance
// http://your-langfuse:3000/api/public/traces
```

This gives you a full audit trail of every error, every retry, every compaction, and every permission decision across all your workflows.
