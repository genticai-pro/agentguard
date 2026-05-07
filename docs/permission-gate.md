# PermissionGate — Full Documentation

## What It Does

PermissionGate intercepts every tool call your AI agent wants to make and evaluates it against a policy ruleset before execution. Rules are glob patterns matched against tool name + tool input. Each rule produces one of three decisions: allow, deny, or prompt.

The problem it solves: n8n AI agent nodes either have full access to all connected tools or none. There's no built-in way to say "the agent can read files but not delete them" or "it can run SELECT queries but not DROP TABLE." PermissionGate adds that layer without modifying your agent node.

---

## Permission Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `allow` | Execute immediately, log to audit | Safe read operations |
| `deny` | Block execution, return structured error to the model | Destructive commands |
| `prompt` | Hold execution, notify operator, wait for approval | Write operations, deploys, external API POSTs |

---

## Configuration

Open the **Evaluate Permission** Code node. The `rules` array and `config` object are at the top:

```javascript
const config = {
  defaultAction: 'prompt',   // What to do when no rule matches
  auditLog: true,
  logDestination: 'output',  // 'postgres', 'webhook', 'file', 'output'
  webhookUrl: '',            // Required if logDestination = 'webhook'
};
```

### Default Rule Set

The workflow ships with a comprehensive default ruleset covering bash, git, HTTP, database, and file operations:

```javascript
const rules = [
  // Bash — Read (allow)
  { tool: 'bash', pattern: 'ls *',       action: 'allow' },
  { tool: 'bash', pattern: 'cat *',      action: 'allow' },
  { tool: 'bash', pattern: 'grep *',     action: 'allow' },
  { tool: 'bash', pattern: 'find *',     action: 'allow' },
  { tool: 'bash', pattern: 'head *',     action: 'allow' },
  { tool: 'bash', pattern: 'tail *',     action: 'allow' },
  { tool: 'bash', pattern: 'pwd',        action: 'allow' },
  { tool: 'bash', pattern: 'whoami',     action: 'allow' },

  // Git — Read (allow)
  { tool: 'bash', pattern: 'git status*', action: 'allow' },
  { tool: 'bash', pattern: 'git log*',    action: 'allow' },
  { tool: 'bash', pattern: 'git diff*',   action: 'allow' },

  // Git — Write (gate)
  { tool: 'bash', pattern: 'git commit*', action: 'prompt' },
  { tool: 'bash', pattern: 'git push*',   action: 'prompt' },
  { tool: 'bash', pattern: 'git merge*',  action: 'prompt' },

  // Bash — Destructive (block)
  { tool: 'bash', pattern: 'rm -rf*',          action: 'deny' },
  { tool: 'bash', pattern: 'sudo rm*',          action: 'deny' },
  { tool: 'bash', pattern: '*DROP TABLE*',      action: 'deny' },
  { tool: 'bash', pattern: '*DROP DATABASE*',   action: 'deny' },
  { tool: 'bash', pattern: '*shutdown*',        action: 'deny' },
  { tool: 'bash', pattern: '*reboot*',          action: 'deny' },
  { tool: 'bash', pattern: 'chmod 777*',        action: 'deny' },
  { tool: 'bash', pattern: '*dd if=*',          action: 'deny' },

  // HTTP
  { tool: 'http', pattern: 'GET *',      action: 'allow'  },
  { tool: 'http', pattern: 'POST *',     action: 'prompt' },
  { tool: 'http', pattern: 'DELETE *',   action: 'deny'   },

  // Database
  { tool: 'database', pattern: 'SELECT *', action: 'allow'  },
  { tool: 'database', pattern: 'INSERT *', action: 'prompt' },
  { tool: 'database', pattern: 'UPDATE *', action: 'prompt' },
  { tool: 'database', pattern: 'DELETE *', action: 'deny'   },
  { tool: 'database', pattern: 'DROP *',   action: 'deny'   },

  // File Operations
  { tool: 'file', pattern: 'read *',   action: 'allow'  },
  { tool: 'file', pattern: 'write *',  action: 'prompt' },
  { tool: 'file', pattern: 'delete *', action: 'deny'   },

  // Catch-all
  { tool: '*', pattern: '*', action: 'prompt' }
];
```

Rules are evaluated **in order** — first match wins. Place more specific rules before broad wildcards.

---

## Glob Pattern Matching

Patterns use `*` as a wildcard matching any sequence of characters. Matching is case-insensitive.

```
'ls *'        matches  'ls /home/user'       ✅
'ls *'        matches  'ls -la /etc'         ✅
'ls *'        matches  'ls'                  ❌  (no trailing space+content)
'git*'        matches  'git status'          ✅
'*DROP*'      matches  'some DROP TABLE foo' ✅
```

> **Important:** Pattern matching runs against the full tool input string, not just the command name. `'rm -rf*'` will match `'rm -rf /var'` but not `'rm file.txt'` — which is intentional. Write your deny patterns broadly enough to catch variations.

---

## Input Shape

```json
{
  "toolName": "bash",
  "toolInput": "git push origin main",
  "requestId": "optional-trace-id"
}
```

The workflow also accepts `tool` as an alias for `toolName` and `input`/`command` as aliases for `toolInput`.

---

## Output Shape

### Allowed
```json
{
  "action": "allow",
  "approved": true,
  "blocked": false,
  "needsApproval": false,
  "modelMessage": null,
  "ruleMatched": true,
  "matchedPattern": "bash:git status*",
  "toolName": "bash",
  "toolInput": "git status",
  "audit": { "timestamp": "...", "tool": "bash", "input": "git status", "action": "allow", "decision": "APPROVED" }
}
```

### Blocked
```json
{
  "action": "deny",
  "approved": false,
  "blocked": true,
  "needsApproval": false,
  "modelMessage": "Permission denied: bash with input matching 'rm -rf*' is blocked by policy.",
  "matchedPattern": "bash:rm -rf*"
}
```

### Needs Approval
```json
{
  "action": "prompt",
  "approved": false,
  "blocked": false,
  "needsApproval": true,
  "modelMessage": "This action requires operator approval. Waiting for confirmation.",
  "matchedPattern": "bash:git push*"
}
```

---

## Wiring

```
[AI Agent produces tool call]
        |
        ▼
[Execute Workflow: PermissionGate]
        |
        ├── output 0: Allowed
        │       |
        │       ▼
        │   [Execute Tool]
        │       |
        │       ▼
        │   [Return result to agent]
        |
        ├── output 1: Blocked
        │       |
        │       ▼
        │   [Return modelMessage to agent]
        │   (agent tries a different approach)
        |
        └── output 2: Needs Approval
                |
                ▼
            [Notify operator via Slack/Telegram/webhook]
                |
                ▼
            [Wait for approval webhook]
                |
                ├── approved → [Execute Tool]
                └── denied  → [Return denial to agent]
```

---

## Operator Approval Flow

When `needsApproval: true`, you need to notify someone and wait. The recommended pattern:

1. Send a Slack/Telegram message with the tool name, input, and approve/deny buttons
2. Use n8n's **Wait** node (webhook resume) to pause the workflow
3. Wire your approval button to call the resume webhook
4. Continue execution based on the approval decision

For Telegram integration, use the `prompt` output to call your Telegram bot with an inline keyboard. This is how the Ceiba agent framework (Gentic AI) handles T3 approval gates for unsolicited outbound actions.

---

## Audit Logging

Every decision is logged. Configure `logDestination` in the config:

**`output`** (default): Audit entry included in workflow output for parent workflow to handle.

**`postgres`**: Add a Postgres node after PermissionGate on the audit output path:
```sql
INSERT INTO agent_audit (timestamp, tool, input, rule_matched, action, decision)
VALUES ($1, $2, $3, $4, $5, $6)
```

**`webhook`**: Set `webhookUrl` in config. The audit payload is POSTed as JSON automatically.

**`file`**: Add a Write Binary File node to persist the audit log to disk.

---

## Security Notes

**Rules are evaluated client-side in the n8n Code node.** This is fine for most use cases — the goal is to prevent accidental destructive actions by the model, not to defend against a malicious actor with access to your n8n instance.

**The catch-all rule `{ tool: '*', pattern: '*', action: 'prompt' }` is intentional.** Any tool call that doesn't match a more specific rule goes to operator approval. Remove it only if you've explicitly covered all tools your agent can access.

**Pattern matching is case-insensitive but not SQL-injection-aware.** For database tools, match on the query prefix (`SELECT *`, `DROP *`). An agent that constructs queries dynamically could potentially bypass prefix matching — combine PermissionGate with your database user's GRANT permissions for defense in depth.
