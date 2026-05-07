# n8n Community Post — AgentGuard Launch

> Copy-paste into https://community.n8n.io — post under "Show & Tell"

---

**Title:** I open-sourced our production agent reliability toolkit — RetryClassifier, ContextBudget, PermissionGate (free, MIT)

---

We run 55 active n8n workflows across three businesses. A few months ago I audited all of them for agent reliability.

The results were bad:
- 14 had no error handling whatsoever
- 8 had retry logic that treated a 429 rate limit the same as a 401 auth failure
- All of them would silently crash on a large tool output (50KB+ web scrape, big API dump, etc.)
- None had any permission layer — agents had full access to every connected tool, all the time

So we built the reliability layer we needed. Then we open-sourced it.

---

## What We Shipped

**AgentGuard** — three importable n8n sub-workflows, MIT licensed, zero dependencies:

### 🔄 RetryClassifier

10-class error taxonomy with specific recovery per class:

| Error | Recovery |
|-------|----------|
| 429 rate limit | Parse `Retry-After` header → wait exact duration |
| 529 server overload | Track consecutive → switch model after 3 |
| 400 context overflow | Trigger compaction → retry |
| 401/403 auth failure | Refresh token → retry once only |
| Network error | Disable keep-alive → fresh connection |
| Quota exceeded | Permanent model downgrade |
| Streaming stall | Abort → retry non-streaming |
| Input validation | Log + fail (no retry — fix the input) |

Backoff formula: `min(500ms × 2^attempt, 32s) + jitter`. No thundering herd.

When your primary API is down, it cascades to local Ollama at $0/call.

### 📐 ContextBudget

Two problems, one component:

**Problem 1:** One large tool result (50KB web scrape) fills your context window.
**Solution:** Budget enforcement — truncated preview + file ref. Model sees enough, can retrieve full result if needed.

**Problem 2:** After 15 turns, accumulated history fills context.
**Solution:** Two-tier compaction — Tier 1 clears old tool results for free, Tier 2 summarizes with cheapest model (~$0.001) only when needed.

Includes a circuit breaker that prevents the infinite-compaction loop that cost a few people I know hundreds of dollars before they noticed.

### 🔒 PermissionGate

Glob-pattern matching on every tool call before execution:

```json
{ "tool": "bash",     "pattern": "ls *",     "action": "allow" }
{ "tool": "bash",     "pattern": "rm -rf*",  "action": "deny"  }
{ "tool": "bash",     "pattern": "git push*","action": "prompt" }
{ "tool": "database", "pattern": "SELECT *", "action": "allow" }
{ "tool": "database", "pattern": "DROP *",   "action": "deny"  }
```

Three modes: `allow` (execute + log), `deny` (block + return error to model), `prompt` (hold + notify operator + wait for approval).

Every decision logged to PostgreSQL, webhook, or file.

---

## How to Use

1. Download the workflow JSONs from the GitHub repo
2. Import into n8n (Settings → Import Workflow)
3. Add an Execute Workflow node in your existing agent workflow
4. Set your config (fallback model, context threshold, permission rules)
5. Done

Each component is standalone. Use one, two, or all three.

---

## What It Works With

- n8n 1.70+ (full support)
- Any LLM provider — OpenAI, Anthropic, Google, Groq, Ollama, Azure, any OpenAI-compatible endpoint
- Any agent type — Tools Agent, OpenAI Functions Agent, ReAct Agent, Custom Agent
- Self-hosted — zero external dependencies

---

## Why We Open-Sourced It

We got tired of our own agents failing silently. Every pattern in AgentGuard was extracted from a real production failure — a lead lost to a rate limit crash, a pipeline corrupted by context overflow, an agent that ran a destructive command it shouldn't have been able to run.

If you've hit any of these problems, this should help. And if you've hit failure modes we haven't covered, open an issue with the error output and we'll add a recovery strategy.

**GitHub:** https://github.com/genticai-pro/agentguard

---

Happy to answer questions about implementation, wiring patterns, or how we integrated this with our Bland AI voice agents and DealiQ real estate pipeline.
