# pi-retry

A [pi](https://github.com/badlogic/pi) extension that retries failed LLM responses — automatically for transient streaming errors, manually via `/retry` or just pressing Enter.

## Features

### Auto-retry transient errors

When an LLM response fails with a transient streaming error (e.g. `"aborted"` from an upstream proxy/gateway), the extension automatically retries with exponential backoff:

- **Delays:** 2s → 4s → 8s
- **Max attempts:** 3
- **No history pollution:** The failed response is invisible to the model (pi's `transform-messages` strips aborted/errored assistant messages). The retry trigger uses `display: false` so it's hidden in the TUI.

Only errors *not* already handled by pi's built-in retry are retried (overloaded, rate limit, 429, 5xx, etc. are left to pi).

### Manual retry: `/retry`

Type `/retry` after any error or abort to re-invoke the LLM. The model starts fresh from the last user message — it never sees the failed partial response.

### Manual retry: press Enter

After an error or user-initiated abort (ESC), just press Enter on an empty editor to retry. This is the fastest path for the common "oops, I shouldn't have cancelled" scenario.

This works by intercepting raw terminal input via pi's `onTerminalInput` hook. The Enter keypress is consumed only when all of these are true:

- The editor is empty
- The agent is idle
- The last response was an error or abort

Otherwise Enter behaves normally.

## Installation

### As a pi package (recommended)

Add to your `~/.pi/agent/settings.json`:

```json
{
  "extensions": [
    "/path/to/pi-retry/index.ts"
  ]
}
```

### For development/testing

```bash
pi -e /path/to/pi-retry/index.ts
```

## Logging

Every retry attempt is logged to `~/.pi/logs/pi-retry.jsonl` with:

- Provider, model, API type, thinking level
- Stop reason and error message
- Attempt number and delay
- Working directory and session ID

Event types: `retry`, `retry_succeeded`, `retry_exhausted`, `manual_retry`.

## How it works

1. **`agent_end` event** — Checks if the last assistant message has a retryable error. If so, waits with backoff and sends a hidden `sendMessage` with `triggerTurn: true`.

2. **`context` event** — Strips the hidden retry trigger message before the LLM sees it. The aborted assistant message is already stripped by pi's `transform-messages`.

3. **`onTerminalInput` hook** — Intercepts Enter on empty editor to trigger manual retry. Consumes the keypress so it doesn't reach the editor.

4. **`/retry` command** — Explicit retry for when you want to be deliberate about it.

5. **`turn_end` event** — Resets the retry counter when a successful response comes through.
