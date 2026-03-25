# Agent Progress Feedback

When a user sends a message to the bot, there is no feedback between the moment the message is received and the moment the agent produces its final response. The user is left wondering whether anything is happening, especially when the agent takes 10-30+ seconds (container startup, complex queries, multi-agent tasks). This design adds two layers of feedback: an immediate acknowledgment from the host and progress updates streamed from the agent-runner.

## Architecture

Two independent feedback mechanisms, layered on top of the existing output flow (which is completely unchanged):

1. **Immediate acknowledgment** — the host sends a short message to the channel before spawning the container. Instant, no container dependency.
2. **Progress updates** — the agent-runner observes SDK messages inside the container and emits progress markers via stdout. The host parses and forwards these as one-liners to the channel.

```
User: @Andy what's the weather in London?
Andy: On it!                             <- immediate ack (host, instant)
Andy: Searching the web...               <- progress (agent-runner, streamed)
Andy: Here's the weather in London...    <- normal agent response (unchanged)
```

Progress updates are additive. They do not replace any existing agent communication. The agent's normal responses via `result` messages continue to flow exactly as before.

## Immediate Acknowledgment

### Location

`processGroupMessages()` in `src/index.ts`, after the typing indicator is set (line 208) and before `runAgent()` is called.

### Behavior

- Send a brief ack message directly via the channel (`channel.sendMessage(chatJid, ackMessage)`)
- The ack message is defined as `ACK_MESSAGE` in `src/config.ts`, defaulting to `"On it!"`
- Only for new container startups (the slow path through `processGroupMessages`)
- Follow-up messages piped to an already-running container do not get an ack (the container is warm, response is faster, and the typing indicator suffices)

## Progress Protocol

### New marker type

A new stdout marker pair, parallel to the existing output markers:

```
---NANOCLAW_PROGRESS_START---
{"message":"Searching the web..."}
---NANOCLAW_PROGRESS_END---
```

The payload is a single JSON object with one field: `message` (a human-readable one-liner).

### Parsing

The existing stdout stream parser in `runContainerAgent()` (`src/container-runner.ts`) already scans for `OUTPUT_START_MARKER` / `OUTPUT_END_MARKER` pairs. The same scanning logic is extended to also detect `PROGRESS_START_MARKER` / `PROGRESS_END_MARKER` pairs in the same buffer.

When a progress marker pair is found, a new `onProgress` callback is invoked (passed alongside the existing `onOutput` callback).

## Agent-Runner Progress Observation

### Location

The `query()` loop in `container/agent-runner/src/index.ts`, inside the `for await (const message of query(...))` loop.

### Observed events

**1. Tool use**

`assistant` messages contain content blocks. When a content block has `type: 'tool_use'`, the tool name is mapped to a friendly description:

| Tool | Progress message |
|------|-----------------|
| `Bash` | Running a command... |
| `Read` | Reading files... |
| `Write`, `Edit` | Writing code... |
| `Glob`, `Grep` | Searching the codebase... |
| `WebSearch` | Searching the web... |
| `WebFetch` | Fetching a web page... |
| `TodoWrite` | Planning next steps... |
| `Task`, `TeamCreate` | Spinning up a subagent... |
| `SendMessage` | Coordinating with agents... |
| Any other tool | Using {toolName}... |

**2. Agent teams**

`system` messages with `subtype: 'task_notification'` carry a `status` and `summary`:

| Status | Progress message |
|--------|-----------------|
| `started` | Subagent working on: {summary} |
| `completed` | Subagent finished: {summary} |

### Throttling

The agent-runner tracks `lastProgressTime` (timestamp of the last progress or result emission). A progress marker is only emitted if at least **12 seconds** have elapsed since the last emission. This keeps updates sparse without suppressing meaningful transitions. The timer resets on every emission (progress or result).

## Host Forwarding

### Location

`processGroupMessages()` in `src/index.ts`.

### Behavior

- `runContainerAgent()` receives a new `onProgress` callback alongside the existing `onOutput`
- The callback sends the progress message to the channel via `channel.sendMessage(chatJid, progressMessage)`
- No additional throttling on the host side; the agent-runner's 12-second throttle is the single source of rate control
- The existing typing indicator logic (`setTyping` at start/end) is unchanged

### Error path

Progress messages already sent are fine if the agent later errors. They were accurate at the time. The existing error handling (cursor rollback, retry) is untouched.

## Files Changed

| File | Change |
|------|--------|
| `src/config.ts` | Add `ACK_MESSAGE` constant (default `"On it!"`) |
| `src/index.ts` | Send immediate ack in `processGroupMessages()` before `runAgent()`. Pass `onProgress` callback to container runner. |
| `src/container-runner.ts` | Add `PROGRESS_START/END` marker constants and parsing in the stdout stream parser. Add `onProgress` callback parameter to `runContainerAgent()`. |
| `container/agent-runner/src/index.ts` | Add `writeProgress()` function. Observe `tool_use` content blocks and `task_notification` messages in the `query()` loop. Throttle at 12s intervals. Tool-name-to-friendly-message mapping with fallback. |

## What Doesn't Change

Router, channels, IPC, queue, types, database — all untouched. The existing output flow (`OUTPUT_START/END` markers, `onOutput` callback, `result` message handling) is completely preserved.

## Deployment Note

Since `container/agent-runner/src/index.ts` changes, the container image needs a rebuild (`./container/build.sh`). The agent-runner source is synced per-group into `data/sessions/{group}/agent-runner-src/` at container startup, but only if the directory doesn't already exist (`container-runner.ts` line 193). Existing per-group copies must be force-refreshed (delete the `agent-runner-src/` directories or update the sync logic to always overwrite).
