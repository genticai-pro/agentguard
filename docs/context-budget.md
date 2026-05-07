# ContextBudget — Full Documentation

## What It Does

ContextBudget solves two problems that kill long-running AI agent workflows:

1. **Oversized tool results** fill your context window in a single turn
2. **Accumulated conversation history** exhausts the context window after many turns

It runs a two-tier compaction pipeline on every agent loop iteration, using the cheapest possible method first. Tier 1 costs $0. Tier 2 costs ~$0.001. A circuit breaker prevents infinite compaction loops.

---

## Two Problems, One Pipeline

### Problem 1: Tool Result Bloat

A web scrape returns 80KB. A database query returns 500 rows. A file read returns a 10,000-line log. Any of these can consume your entire context window on a single tool call, leaving the model with no room to think.

**Solution: Budget Enforcement**

Every tool result is measured before it reaches the model. Results above `maxResultChars` get truncated to a head + tail preview:

```
[First 500 chars of result]

[... 73,200 chars truncated — full result saved to /tmp/agentguard/tool_results/1712620800_tool_0.txt ...]

[Last 200 chars of result]
```

The model sees enough to understand what the tool returned. If it needs the full content, it has a file path reference to retrieve it.

### Problem 2: Context Window Exhaustion

After 15-20 turns of an agent loop, the accumulated conversation history — including all previous tool calls and responses — fills the context window. The next API call either fails with a 400 or the model starts hallucinating from degraded attention.

**Solution: Two-Tier Compaction**

| Tier | Name | When | Cost | What Happens |
|------|------|------|------|--------------|
| 0 | Budget | Every turn | $0 | Oversized results are truncated before entering history |
| 1 | Microcompact | Every turn | $0 | Old tool results (>3 turns ago) replaced with `[Cleared — re-execute if needed]` |
| 2 | Auto-compact | Context > 80% | ~$0.001 | All but last 3 turns summarized by cheapest model |

Tier 1 runs automatically on every call. Tier 2 only fires when Tier 1 isn't enough.

---

## Configuration

Open the **Budget & Compact** Code node:

```javascript
const config = {
  maxResultChars: 8000,          // Truncate tool results above this size
  previewHeadChars: 500,         // First N chars to show in truncated preview
  previewTailChars: 200,         // Last N chars to show in truncated preview
  compactionThreshold: 0.80,     // Auto-compact when context is 80% full
  protectedTailTurns: 3,         // Never compact/clear the last N turns
  maxConsecutiveCompactions: 3,  // Circuit breaker: stop after N consecutive compaction failures
  summaryModel: 'gemma4:e4b',    // Model for Tier 2 auto-compact summaries
  summaryEndpoint: 'http://localhost:11434/v1/chat/completions',  // CONFIGURE for your stack
  persistPath: '/tmp/agentguard/tool_results',  // Where truncated full results are saved
  contextWindowTokens: 128000,   // Your model's context window size
  charsPerToken: 4               // Approximate chars-per-token ratio
};
```

> **summaryEndpoint**: Defaults to local Ollama. For cloud deployments, use any OpenAI-compatible endpoint (Groq, Together, Anyscale, etc.). The cheapest model you have access to is the right choice — it's just writing a 3-sentence summary.

---

## Input Shape

```json
{
  "toolResults": [
    {
      "toolName": "web_scrape",
      "content": "...full tool output string...",
      "_agTurn": 4
    }
  ],
  "conversationHistory": [
    { "role": "user", "content": "..." },
    { "role": "assistant", "content": "..." },
    { "role": "tool", "content": "...", "_agTurn": 1, "_agIsToolResult": true }
  ],
  "turnNumber": 7,
  "_agCompactionFailures": 0
}
```

| Field | Description |
|-------|-------------|
| `toolResults` | Tool results from this turn (before adding to history) |
| `conversationHistory` | Full accumulated history from your agent loop |
| `turnNumber` | Current turn number (increment each loop iteration) |
| `_agCompactionFailures` | Pass back from previous output — tracks circuit breaker state |

---

## Output Shape

```json
{
  "toolResults": [...],           // Budgeted results — use these instead of originals
  "filesToPersist": [             // Pipe to file writer node
    { "path": "/tmp/...", "content": "...full content..." }
  ],
  "conversationHistory": [...],   // Microcompacted history
  "microcompactedCount": 3,
  "needsAutoCompact": false,
  "compactionBlocked": false,
  "autoCompactRequest": null,     // Non-null when auto-compact should fire
  "estimatedTokens": 24500,
  "contextUsage": "0.191",
  "_agCompactionFailures": 0,
  "_agBudgetReport": {
    "totalResults": 2,
    "truncated": 1,
    "charsSaved": 68400,
    "microcompacted": 3
  }
}
```

---

## Wiring

```
[Agent Loop Start]
        |
        ▼
[Execute Workflow: ContextBudget]
        |
        ├── needsAutoCompact = true
        │       |
        │       ▼
        │   [HTTP: POST to summaryEndpoint]
        │       |
        │       ▼
        │   [Code: Replace History Prefix with Summary]
        │       |
        │       ▼ (back to agent)
        │   [AI Agent Node]
        |
        └── needsAutoCompact = false
                |
                ▼
            [AI Agent Node]
```

The auto-compact path hits your summary model, gets a compressed summary of old turns, and replaces them in the history before the next AI Agent call.

---

## The Circuit Breaker

The infinite compaction loop is a real failure mode. Here's how it happens:

1. Context is 85% full → auto-compact fires
2. The summary model returns a long response → context is still 82% full
3. Auto-compact fires again immediately
4. Repeat until you've spent $20 on summaries and the agent never made progress

**The fix:** `maxConsecutiveCompactions = 3`. After 3 consecutive auto-compactions without dropping below the threshold, the circuit breaker trips. `compactionBlocked = true` is set in the output. Your agent continues without compacting — you get a hard failure on context overflow rather than an infinite expensive loop.

When `compactionBlocked` is true, route to your error handler and notify the operator.

---

## File Persistence

Truncated tool results are referenced by file path but not actually written to disk by ContextBudget — the write happens in your parent workflow. The `filesToPersist` array in the output contains `{ path, content }` pairs. Add a Code node after ContextBudget to write them:

```javascript
// Write truncated tool results to disk
const files = $json.filesToPersist || [];
for (const file of files) {
  // Use n8n filesystem or an HTTP call to your storage
  // The path is already set: file.path
  // The content: file.content
}
return $input.all();
```

If you don't need full-result retrieval, set `persistPath` to `null` and remove this step — the truncated preview is all the model will see.

---

## Token Estimation

ContextBudget estimates tokens as `totalChars / charsPerToken`. This is intentionally approximate — the exact count depends on the tokenizer, which varies by model. Set `charsPerToken: 3` for code-heavy conversations, `4` for natural language, `4.5` for compressed/minified content.

The threshold fires at 80% (`compactionThreshold: 0.80`) to give headroom. You want to compact before you're at 95% with no room for the summary model's response to be added to history.
