# RetryClassifier — Full Documentation

## What It Does

RetryClassifier is a sub-workflow you call after any HTTP/API error. It parses the response, classifies the error into one of 10 categories, and returns an exact recovery decision: how long to wait, whether to switch models, and what action to take next.

Without it, your agents either don't retry at all, or retry everything with a flat 5-second wait — which hammers rate-limited endpoints, wastes time on non-retriable errors, and ignores the `Retry-After` header that providers explicitly send to tell you when to try again.

---

## Error Classes

| Class | Trigger | Recovery |
|-------|---------|----------|
| `rate_limited` | HTTP 429, `Retry-After` ≤ 20s | Wait exact `Retry-After` duration, retry |
| `rate_limited_long` | HTTP 429, `Retry-After` > 20s | Wait up to 30 min, retry |
| `quota_exceeded` | HTTP 429 + `x-overage-disabled: true` | Permanently switch to fallback model |
| `server_overload` | HTTP 529 or 503 | Exponential backoff; after 3 consecutive → switch model |
| `context_overflow` | HTTP 400 + message contains "context"/"token"/"too long" | Trigger compaction, retry |
| `auth_failure` | HTTP 401 or 403 | Clear credential cache, refresh token, retry once only |
| `network_error` | Status 0, ECONNRESET, EPIPE, timeout, ETIMEDOUT | Fresh connection (disable keep-alive), exponential backoff |
| `streaming_stall` | Error message contains "stream" + "stall" | Abort stream, retry non-streaming |
| `input_validation` | HTTP 400 (not context), 422 | Log and fail — do not retry, the input is wrong |
| `unknown` | Anything else | Log to audit, retry up to `maxRetries`, then fail |

---

## Configuration

Open the **Classify Error** Code node. All settings are in the `config` object at the top:

```javascript
const config = {
  maxRetries: 5,                   // Max retry attempts before failing
  baseDelay: 500,                  // Base backoff delay in ms
  maxDelay: 32000,                 // Cap on backoff delay (32s)
  jitterRatio: 0.25,               // Random jitter: ±25% of calculated delay
  fallbackModel: 'gemma4:e4b',     // Model to switch to when primary overloaded
  fallbackEndpoint: 'http://localhost:11434/v1/chat/completions',
  persistentRetry: false,          // true = keep retrying for long rate limits (CI/CD use)
  persistentMaxDelay: 300000,      // Max persistent retry wait: 5 min
  consecutiveOverloadThreshold: 3  // 529s before switching to fallback
};
```

**Backoff formula:** `min(baseDelay × 2^attempt, maxDelay) + (random × jitterRatio × delay)`

---

## Input Shape

Pass this to the workflow via Execute Workflow node:

```json
{
  "statusCode": 429,
  "headers": {
    "retry-after": "5",
    "x-overage-disabled": "false"
  },
  "body": {
    "error": {
      "message": "Rate limit exceeded. Please retry after 5 seconds."
    }
  },
  "_agRetryAttempt": 0,
  "_ag529Count": 0
}
```

| Field | Type | Description |
|-------|------|-------------|
| `statusCode` | number | HTTP status code from the failed request |
| `headers` | object | Response headers (used for `Retry-After`, overage flags) |
| `body` | object/string | Response body (used for error message classification) |
| `_agRetryAttempt` | number | Current retry attempt count (start at 0, increment each pass) |
| `_ag529Count` | number | Consecutive 529/503 count (for overload threshold tracking) |

---

## Output Shape

```json
{
  "errorClass": "rate_limited",
  "statusCode": 429,
  "action": "retry",
  "shouldRetry": true,
  "delay": 5000,
  "message": "Rate limited. Waiting 5s (from Retry-After header).",
  "switchModel": false,
  "targetModel": null,
  "targetEndpoint": null,
  "compactionNeeded": false,
  "disableKeepAlive": false,
  "disableStreaming": false,
  "clearAuthCache": false,
  "permanent": false,
  "_agRetryAttempt": 1,
  "_ag529Count": 0,
  "_agTimestamp": "2026-04-09T00:00:00.000Z"
}
```

| Field | Description |
|-------|-------------|
| `action` | `retry`, `switch_model`, `compact_and_retry`, `refresh_and_retry`, `retry_non_streaming`, `fail` |
| `shouldRetry` | Boolean — wire this to your If node |
| `delay` | Milliseconds to wait before retrying — wire to Wait node |
| `switchModel` | true = update your API call to use `targetModel` + `targetEndpoint` |
| `compactionNeeded` | true = call ContextBudget before retrying |
| `_agRetryAttempt` | Pass this back in on next attempt |
| `_ag529Count` | Pass this back in on next attempt |

---

## Wiring

```
[Your HTTP Request]
        |
        | (on error)
        ▼
[Map Error to RetryClassifier Input]  ← Code node
        |
        ▼
[Execute Workflow: RetryClassifier]
        |
        ├── shouldRetry = true
        │       |
        │       ▼
        │   [Wait: {{ $json.delay }}ms]
        │       |
        │       ▼ (loop back)
        │   [Your HTTP Request]
        |
        └── shouldRetry = false
                |
                ▼
            [Error Handler]
```

---

## Model Fallback Chain

When `switchModel: true`, update the downstream HTTP node before retrying:

```javascript
// In a Code node before your API call
const modelOverride = $json.switchModel ? {
  model: $json.targetModel,
  url: $json.targetEndpoint
} : {
  model: 'claude-haiku-4.5',
  url: 'https://api.anthropic.com/v1/messages'
};
```

**Example cascade:**
```
Anthropic Claude Haiku → Groq Llama → Local Ollama (gemma4:e4b) → Fail gracefully
```

---

## Edge Cases

**The `Retry-After` header isn't always a number.** Some providers send an HTTP date string (`Retry-After: Wed, 09 Apr 2026 12:00:00 GMT`). RetryClassifier uses `parseInt()` which returns `NaN` for date strings — it falls back to a 2-second default. If your provider sends date strings, parse them in the input mapping node.

**`persistent_retry`** keeps cycling even after `maxRetries`. Use for CI/CD pipelines that must eventually succeed, not for user-facing agents.

**Auth failure only retries once.** The logic is: first failure → refresh token → retry. If it fails again, the token refresh didn't work — escalate rather than loop.
