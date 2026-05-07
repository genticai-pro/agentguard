<p align="center">
  <img src="./assets/banner.png" alt="AgentGuard — Production reliability for n8n AI agents" width="100%"/>
</p>

<p align="center">
  <a href="https://github.com/genticai-pro/agentguard/stargazers"><img src="https://img.shields.io/github/stars/genticai-pro/agentguard?style=for-the-badge&color=00ff88&labelColor=0d1117" alt="GitHub Stars"/></a>
  <a href="https://github.com/genticai-pro/agentguard/network/members"><img src="https://img.shields.io/github/forks/genticai-pro/agentguard?style=for-the-badge&color=4f9cf9&labelColor=0d1117" alt="GitHub Forks"/></a>
  <a href="https://github.com/genticai-pro/agentguard/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-MIT-8b5cf6?style=for-the-badge&labelColor=0d1117" alt="MIT License"/></a>
  <a href="https://n8n.io"><img src="https://img.shields.io/badge/n8n-1.70%2B-f97316?style=for-the-badge&labelColor=0d1117" alt="n8n 1.70+"/></a>
  <a href="https://github.com/genticai-pro/agentguard/commits/main"><img src="https://img.shields.io/github/last-commit/genticai-pro/agentguard?style=for-the-badge&color=ff6b6b&labelColor=0d1117" alt="Last Commit"/></a>
</p>

<p align="center">
  <strong>Your n8n agents are failing silently.</strong><br/>
  Context windows overflow. Rate limits crash without recovery. Tool outputs flood the conversation.<br/>
  <strong>AgentGuard fixes this in under 5 minutes — free, MIT, drop-in.</strong>
</p>

<p align="center">
  <a href="#quick-start"><img src="https://img.shields.io/badge/→_Quick_Start-00ff88?style=for-the-badge&labelColor=0d1117" alt="Quick Start"/></a>
  <a href="./workflows/"><img src="https://img.shields.io/badge/↓_Download_Workflows-4f9cf9?style=for-the-badge&labelColor=0d1117" alt="Download"/></a>
  <a href="https://genticai.pro"><img src="https://img.shields.io/badge/Enterprise_Stack-f97316?style=for-the-badge&labelColor=0d1117" alt="Enterprise"/></a>
</p>

---

## What's Inside

Three importable n8n sub-workflows. Use one, two, or all three:

| | Component | What it does | Deploy time |
|---|-----------|-------------|-------------|
| 🔄 | **[RetryClassifier](#-retryclassifier)** | 10-class error taxonomy. Specific recovery per error type, not generic retry. | 2 min |
| 📐 | **[ContextBudget](#-contextbudget)** | Tool result budgeting + two-tier compaction. Prevents silent context overflow. | 2 min |
| 🔒 | **[PermissionGate](#-permissiongate)** | Glob-pattern allow/deny/prompt per tool call. Full audit log. | 3 min |

---

## The Problem

<p align="center">
  <img src="./assets/timeline.png" alt="27-hour silent incident timeline — May 4th" width="100%"/>
</p>

May 4th. The SQLite layer underneath our n8n instance failed. The `/healthz` endpoint returned 200 the entire time. Four webhook callers had `.catch(() => {})`. Zero log lines. Zero alerts. We found out when a prospect mentioned they'd submitted a form two days earlier and never heard back.

**27 hours of silent lead loss.**

That forced an audit of every workflow. The results:

- **14 workflows** had no error handling whatsoever
- **8 workflows** had retry logic that treated a `429 rate limit` identically to a `401 auth failure`
- **All of them** would silently die on a large tool output (50KB+ web scrape, big DB dump)
- **None** had a permission layer — agents had full access to every tool, always

These aren't edge cases. They're what n8n agent workflows look like in production before someone fixes them.

**AgentGuard is the fix.**

---

## Quick Start

### 1. Import

```
n8n Settings → Import Workflow → Select JSON file
```

Download from [`/workflows`](./workflows/):
- [`retry-classifier.json`](./workflows/retry-classifier.json)
- [`context-budget.json`](./workflows/context-budget.json)
- [`permission-gate.json`](./workflows/permission-gate.json)

### 2. Wire

Add an **Execute Workflow** node in your agent workflow pointing to the AgentGuard sub-workflow:

```
[Your AI Agent] → [Execute Workflow: RetryClassifier] → [API Call]
[Tool Output]   → [Execute Workflow: ContextBudget]   → [Back to Agent]
[Tool Request]  → [Execute Workflow: PermissionGate]  → [Tool Execution]
```

### 3. Configure

Each component has a config object at the top of its Code node:

```javascript
// RetryClassifier — set your fallback model
const config = {
  maxRetries: 5,
  baseDelay: 500,
  maxDelay: 32000,
  fallbackModel: "llama3.2:3b",  // local Ollama = $0/call
  fallbackEndpoint: "http://localhost:11434/v1/chat/completions"
};

// ContextBudget — set your summary endpoint
const config = {
  maxResultChars: 8000,
  compactionThreshold: 0.80,
  summaryModel: "llama3.2:3b",
  summaryEndpoint: "http://localhost:11434/v1/chat/completions",  // update for cloud
  protectedTailTurns: 3
};

// PermissionGate — set default action
const config = {
  defaultAction: "prompt",  // "allow", "deny", or "prompt"
  auditLog: true,
  logDestination: "postgres"
};
```

---

## 🔄 RetryClassifier

<p align="center">
  <img src="./assets/error-classification.png" alt="10-class error taxonomy with recovery paths" width="100%"/>
</p>

Not all errors are the same. RetryClassifier parses every HTTP failure and routes it to a specific recovery strategy:

| Error Class | Status | Recovery Strategy |
|-------------|--------|------------------|
| Rate Limited | 429 | Parse `Retry-After` header → wait exact duration |
| Server Overload | 529/503 | Track consecutive count → switch model after 3 |
| Context Overflow | 400 | Trigger ContextBudget compaction → retry |
| Auth Failure | 401/403 | Clear cache → refresh token → retry once |
| Network Error | ECONNRESET/timeout | Disable keep-alive → fresh connection |
| Quota Exceeded | 429 + overage flag | Permanent model downgrade → notify operator |
| Input Validation | 422 | Log → fail (no retry — fix the input) |
| Streaming Stall | timeout (no chunks) | Abort → retry non-streaming |
| Unknown | * | Escalate to error workflow |

**Backoff formula:** `min(500ms × 2^attempt, 32s) + random jitter` — no thundering herd.

**Model fallback chain:**
```
Primary API → Backup API → Local Ollama (any model) → Fail gracefully
```

When your primary API is rate-limited or down, your agent keeps running on local inference at $0/call.

[Full docs →](./docs/retry-classifier.md)

---

## 📐 ContextBudget

<p align="center">
  <img src="./assets/compaction.png" alt="Two-tier compaction — cheapest first" width="100%"/>
</p>

**Two problems. One component. Zero dollars for Tier 1.**

**Problem 1:** Your agent runs a web scrape that returns 50KB. That's ~12,500 tokens — half your context window, gone on one tool call.

**Problem 2:** After 15 turns, your context is full. The next API call fails or the model loses coherence.

**Solution: Two-tier compaction, cheapest first**

| Tier | Trigger | Cost | What Happens |
|------|---------|------|-------------|
| **Microcompact** | Every turn | **$0** | Old tool results cleared, file ref saved |
| **Auto-compact** | Context > 80% | **~$0.001** | History summarized by cheapest model |

**The circuit breaker** (`MAX_CONSECUTIVE_COMPACTION_FAILURES = 3`) prevents the infinite-compaction loop that cost real users hundreds of dollars in wasted API calls before they noticed.

[Full docs →](./docs/context-budget.md)

---

## 🔒 PermissionGate

Your agent has access to Bash, HTTP, database, and file tools. Right now it's all-or-nothing. PermissionGate adds **glob-pattern matching** on every tool call:

```json
{ "tool": "bash",     "pattern": "ls *",        "action": "allow"  },
{ "tool": "bash",     "pattern": "rm -rf*",      "action": "deny"   },
{ "tool": "bash",     "pattern": "git push*",    "action": "prompt" },
{ "tool": "database", "pattern": "SELECT *",     "action": "allow"  },
{ "tool": "database", "pattern": "DROP *",       "action": "deny"   },
{ "tool": "http",     "pattern": "DELETE *",     "action": "deny"   },
{ "tool": "*",        "pattern": "*",            "action": "prompt" }
```

**Three modes:**

| Mode | Behavior |
|------|----------|
| `allow` | Execute immediately + log |
| `deny` | Block + return structured error to model |
| `prompt` | Hold + notify operator (Slack/Telegram/webhook) + wait for approval |

Every decision logged. Enterprise-ready audit trail.

[Full docs →](./docs/permission-gate.md)

---

## Architecture

<p align="center">
  <img src="./assets/architecture.png" alt="AgentGuard architecture" width="100%"/>
</p>

Each component is a **standalone sub-workflow**. They compose but don't depend on each other. Add one at a time — start with RetryClassifier (biggest immediate impact), then ContextBudget, then PermissionGate.

Full integration patterns in [integration-guide.md](./integration-guide.md).

---

## Compatibility

| | Support |
|:--|:--------|
| **n8n** | 1.70+ (full) · 1.50–1.69 (no streaming) |
| **LLM Providers** | OpenAI · Anthropic · Google · Groq · Ollama · Azure · any OpenAI-compatible endpoint |
| **Agent Types** | Tools Agent · OpenAI Functions Agent · ReAct Agent · Custom Agent |
| **Self-hosted** | Yes — runs entirely on your infrastructure, zero external dependencies |

---

## Why Open Source?

We run **55 active workflows** across three businesses — real estate investment, AI automation, and voice AI scheduling. Every pattern here was extracted from a real production failure: a rate limit crash that lost a lead, a context overflow that corrupted a pipeline, an agent that ran `rm -rf` on a staging server.

We got tired of it. We built the reliability layer. Then we open-sourced it.

**Built by [Gentic AI](https://genticai.pro)** — production AI infrastructure for businesses that can't afford agents that fail silently.

---

## Want More?

AgentGuard is the free foundation. If you need the full production stack:

| | AgentGuard | Managed Stack |
|:--|:-----------|:--------------|
| RetryClassifier | ✅ | ✅ |
| ContextBudget | ✅ | ✅ |
| PermissionGate | ✅ | ✅ |
| Agent monitoring dashboard | — | Real-time visibility across all agents |
| Multi-agent orchestration | — | Coordinated execution with shared state |
| Voice AI integration | — | Inbound/outbound voice pipeline |
| Dedicated support | — | Direct Slack channel, same-day response |
| **Price** | **Free** | **[Let's talk →](https://genticai.pro)** |

**Premium n8n templates** — [gentic-n8n.deals](https://gentic-n8n.deals)
**Enterprise infrastructure** — [genticai.pro](https://genticai.pro)

---

## Contributing

PRs welcome. If you've hit an agent failure mode we haven't covered, open an issue with the error output and we'll add a recovery strategy.

```bash
git clone https://github.com/genticai-pro/agentguard.git
cd agentguard
# Import workflows into your n8n instance
# Break things. Improve them. Open a PR.
```

---

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=genticai-pro/agentguard&type=Date)](https://star-history.com/#genticai-pro/agentguard&Date)

---

<p align="center">
  <sub>MIT licensed · Built with 🦞 by <a href="https://genticai.pro">Gentic AI</a> in Las Vegas</sub>
</p>
