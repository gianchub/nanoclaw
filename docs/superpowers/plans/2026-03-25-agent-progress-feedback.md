# Agent Progress Feedback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add immediate acknowledgment and throttled progress updates so users see what the agent is doing while it works.

**Architecture:** Two-layer feedback: host sends instant ack before container startup; agent-runner inside the container observes SDK tool_use and task_notification messages, emits progress markers via new stdout sentinel pairs, host parses and forwards to the channel. Existing output flow is untouched.

**Tech Stack:** TypeScript, Node.js, Claude Agent SDK, vitest

**Spec:** `docs/superpowers/specs/2026-03-25-agent-progress-feedback-design.md`

---

### Task 1: Add ACK_MESSAGE config constant

**Files:**
- Modify: `src/config.ts`

- [ ] **Step 1: Add the constant**

In `src/config.ts`, after the `TRIGGER_PATTERN` export (around line 72), add:

```typescript
export const ACK_MESSAGE = process.env.ACK_MESSAGE || 'On it!';
```

- [ ] **Step 2: Verify build**

Run: `npm run build`
Expected: Clean compilation, no errors.

- [ ] **Step 3: Commit**

```bash
git add src/config.ts
git commit -m "feat: add ACK_MESSAGE config constant"
```

---

### Task 2: Add progress marker constants and parsing to container-runner

**Files:**
- Modify: `src/container-runner.ts`
- Test: `src/container-runner.test.ts`

- [ ] **Step 1: Write the failing test for progress marker parsing**

In `src/container-runner.test.ts`, add a new describe block after the existing tests:

```typescript
const PROGRESS_START_MARKER = '---NANOCLAW_PROGRESS_START---';
const PROGRESS_END_MARKER = '---NANOCLAW_PROGRESS_END---';

function emitProgressMarker(
  proc: ReturnType<typeof createFakeProcess>,
  message: string,
) {
  const json = JSON.stringify({ message });
  proc.stdout.push(`${PROGRESS_START_MARKER}\n${json}\n${PROGRESS_END_MARKER}\n`);
}

describe('container-runner progress markers', () => {
  beforeEach(() => {
    vi.useFakeTimers();
    fakeProc = createFakeProcess();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('calls onProgress when a progress marker is emitted', async () => {
    const onOutput = vi.fn(async () => {});
    const onProgress = vi.fn(async () => {});
    const resultPromise = runContainerAgent(
      testGroup,
      testInput,
      () => {},
      onOutput,
      onProgress,
    );

    // Emit a progress marker
    emitProgressMarker(fakeProc, 'Searching the web...');
    await vi.advanceTimersByTimeAsync(10);

    expect(onProgress).toHaveBeenCalledWith('Searching the web...');

    // Emit output and close
    emitOutputMarker(fakeProc, { status: 'success', result: 'Done', newSessionId: 'sess-1' });
    await vi.advanceTimersByTimeAsync(10);
    fakeProc.emit('close', 0);
    await vi.advanceTimersByTimeAsync(10);

    const result = await resultPromise;
    expect(result.status).toBe('success');
  });

  it('handles multiple progress markers interleaved with output', async () => {
    const onOutput = vi.fn(async () => {});
    const onProgress = vi.fn(async () => {});
    const resultPromise = runContainerAgent(
      testGroup,
      testInput,
      () => {},
      onOutput,
      onProgress,
    );

    emitProgressMarker(fakeProc, 'Reading files...');
    await vi.advanceTimersByTimeAsync(10);
    emitProgressMarker(fakeProc, 'Running a command...');
    await vi.advanceTimersByTimeAsync(10);

    expect(onProgress).toHaveBeenCalledTimes(2);
    expect(onProgress).toHaveBeenCalledWith('Reading files...');
    expect(onProgress).toHaveBeenCalledWith('Running a command...');

    emitOutputMarker(fakeProc, { status: 'success', result: 'Done', newSessionId: 'sess-2' });
    await vi.advanceTimersByTimeAsync(10);
    fakeProc.emit('close', 0);
    await vi.advanceTimersByTimeAsync(10);

    await resultPromise;
  });

  it('works without onProgress callback (backwards compatible)', async () => {
    const onOutput = vi.fn(async () => {});
    const resultPromise = runContainerAgent(
      testGroup,
      testInput,
      () => {},
      onOutput,
      // no onProgress
    );

    emitProgressMarker(fakeProc, 'Searching the codebase...');
    await vi.advanceTimersByTimeAsync(10);

    emitOutputMarker(fakeProc, { status: 'success', result: 'Done', newSessionId: 'sess-3' });
    await vi.advanceTimersByTimeAsync(10);
    fakeProc.emit('close', 0);
    await vi.advanceTimersByTimeAsync(10);

    const result = await resultPromise;
    expect(result.status).toBe('success');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/container-runner.test.ts`
Expected: FAIL — `runContainerAgent` doesn't accept the 5th argument yet, and progress markers are not parsed.

- [ ] **Step 3: Add progress marker constants and parsing**

In `src/container-runner.ts`:

1. After the existing sentinel markers (around line 33), add the new constants:

```typescript
const PROGRESS_START_MARKER = '---NANOCLAW_PROGRESS_START---';
const PROGRESS_END_MARKER = '---NANOCLAW_PROGRESS_END---';
```

2. Add `onProgress` parameter to `runContainerAgent` signature. Change the function signature from:

```typescript
export async function runContainerAgent(
  group: RegisteredGroup,
  input: ContainerInput,
  onProcess: (proc: ChildProcess, containerName: string) => void,
  onOutput?: (output: ContainerOutput) => Promise<void>,
): Promise<ContainerOutput> {
```

to:

```typescript
export async function runContainerAgent(
  group: RegisteredGroup,
  input: ContainerInput,
  onProcess: (proc: ChildProcess, containerName: string) => void,
  onOutput?: (output: ContainerOutput) => Promise<void>,
  onProgress?: (message: string) => Promise<void>,
): Promise<ContainerOutput> {
```

3. Restructure the stdout stream-parsing block in `container.stdout.on('data', ...)`. The existing code has:

```typescript
      // Stream-parse for output markers
      if (onOutput) {
        parseBuffer += chunk;
        let startIdx: number;
        while ((startIdx = parseBuffer.indexOf(OUTPUT_START_MARKER)) !== -1) {
          // ... existing output marker parsing ...
        }
      }
```

Replace the entire `if (onOutput)` block with a widened guard that handles both output and progress markers. The full replacement:

```typescript
      // Stream-parse for output and progress markers
      if (onOutput || onProgress) {
        parseBuffer += chunk;

        // Parse output markers
        if (onOutput) {
          let startIdx: number;
          while ((startIdx = parseBuffer.indexOf(OUTPUT_START_MARKER)) !== -1) {
            const endIdx = parseBuffer.indexOf(OUTPUT_END_MARKER, startIdx);
            if (endIdx === -1) break; // Incomplete pair, wait for more data

            const jsonStr = parseBuffer
              .slice(startIdx + OUTPUT_START_MARKER.length, endIdx)
              .trim();
            parseBuffer = parseBuffer.slice(endIdx + OUTPUT_END_MARKER.length);

            try {
              const parsed: ContainerOutput = JSON.parse(jsonStr);
              if (parsed.newSessionId) {
                newSessionId = parsed.newSessionId;
              }
              hadStreamingOutput = true;
              // Activity detected — reset the hard timeout
              resetTimeout();
              // Call onOutput for all markers (including null results)
              // so idle timers start even for "silent" query completions.
              outputChain = outputChain.then(() => onOutput(parsed));
            } catch (err) {
              logger.warn(
                { group: group.name, error: err },
                'Failed to parse streamed output chunk',
              );
            }
          }
        }

        // Parse progress markers
        if (onProgress) {
          let progStartIdx: number;
          while ((progStartIdx = parseBuffer.indexOf(PROGRESS_START_MARKER)) !== -1) {
            const progEndIdx = parseBuffer.indexOf(PROGRESS_END_MARKER, progStartIdx);
            if (progEndIdx === -1) break;

            const progJsonStr = parseBuffer
              .slice(progStartIdx + PROGRESS_START_MARKER.length, progEndIdx)
              .trim();
            parseBuffer = parseBuffer.slice(progEndIdx + PROGRESS_END_MARKER.length);

            try {
              const parsed = JSON.parse(progJsonStr);
              if (parsed.message) {
                outputChain = outputChain.then(() => onProgress(parsed.message));
              }
            } catch (err) {
              logger.warn(
                { group: group.name, error: err },
                'Failed to parse progress marker',
              );
            }
          }
        }
      }
```

Key points:
- The outer guard is `if (onOutput || onProgress)` so `parseBuffer` is always populated when either callback is present.
- Output and progress parsing each check their own callback.
- Both share the same `parseBuffer` and `outputChain` to preserve message ordering.

4. Also update the streaming-mode resolution guard in the `container.on('close', ...)` handler. Find the block near the end:

```typescript
      // Streaming mode: wait for output chain to settle, return completion marker
      if (onOutput) {
```

Change to:

```typescript
      // Streaming mode: wait for output chain to settle, return completion marker
      if (onOutput || onProgress) {
```

This ensures that if `onProgress` callbacks are still in the output chain when the container exits, they settle before the promise resolves.

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/container-runner.test.ts`
Expected: All tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/container-runner.ts src/container-runner.test.ts
git commit -m "feat: add progress marker parsing to container-runner"
```

---

### Task 3: Thread onProgress through runAgent and add immediate ack

**Files:**
- Modify: `src/index.ts`

- [ ] **Step 1: Import ACK_MESSAGE**

In `src/index.ts`, add `ACK_MESSAGE` to the existing import from `./config.js` (line 6):

```typescript
import {
  ACK_MESSAGE,
  ASSISTANT_NAME,
  CREDENTIAL_PROXY_PORT,
  IDLE_TIMEOUT,
  POLL_INTERVAL,
  TELEGRAM_BOT_POOL,
  TIMEZONE,
  TRIGGER_PATTERN,
} from './config.js';
```

- [ ] **Step 2: Add onProgress parameter to runAgent**

In the `runAgent` function (around line 265), add `onProgress` to the signature. Change:

```typescript
async function runAgent(
  group: RegisteredGroup,
  prompt: string,
  chatJid: string,
  onOutput?: (output: ContainerOutput) => Promise<void>,
): Promise<'success' | 'error'> {
```

to:

```typescript
async function runAgent(
  group: RegisteredGroup,
  prompt: string,
  chatJid: string,
  onOutput?: (output: ContainerOutput) => Promise<void>,
  onProgress?: (message: string) => Promise<void>,
): Promise<'success' | 'error'> {
```

Then pass it through to `runContainerAgent`. Change the call (around line 311):

```typescript
    const output = await runContainerAgent(
      group,
      {
        prompt,
        sessionId,
        groupFolder: group.folder,
        chatJid,
        isMain,
        assistantName: ASSISTANT_NAME,
      },
      (proc, containerName) =>
        queue.registerProcess(chatJid, proc, containerName, group.folder),
      wrappedOnOutput,
    );
```

to:

```typescript
    const output = await runContainerAgent(
      group,
      {
        prompt,
        sessionId,
        groupFolder: group.folder,
        chatJid,
        isMain,
        assistantName: ASSISTANT_NAME,
      },
      (proc, containerName) =>
        queue.registerProcess(chatJid, proc, containerName, group.folder),
      wrappedOnOutput,
      onProgress,
    );
```

- [ ] **Step 3: Send immediate ack and wire onProgress in processGroupMessages**

In `processGroupMessages` (around line 148), after the `setTyping` call and before the `runAgent` call, add the ack. Also define and pass the `onProgress` callback to `runAgent`.

After `await channel.setTyping?.(chatJid, true);` (line 208), add:

```typescript
  // Send immediate acknowledgment (fire-and-forget)
  channel.sendMessage(chatJid, ACK_MESSAGE).catch((err) =>
    logger.warn({ chatJid, err }, 'Failed to send ack message'),
  );
```

Then change the `runAgent` call to include an `onProgress` callback. Replace:

```typescript
  const output = await runAgent(group, prompt, chatJid, async (result) => {
```

with:

```typescript
  const onProgress = async (message: string) => {
    channel.sendMessage(chatJid, message).catch((err) =>
      logger.warn({ chatJid, err }, 'Failed to send progress message'),
    );
  };

  const output = await runAgent(group, prompt, chatJid, async (result) => {
```

And add `onProgress` as the 5th argument to the `runAgent` call. The call's closing `});` (line 237 in the original) becomes `}, onProgress);`. The full edit: find the closing of the streaming callback:

```typescript
  });

  await channel.setTyping?.(chatJid, false);
```

Change to:

```typescript
  }, onProgress);

  await channel.setTyping?.(chatJid, false);
```

- [ ] **Step 4: Verify build**

Run: `npm run build`
Expected: Clean compilation. The task-scheduler call site still passes 4 arguments to `runContainerAgent` which is fine since `onProgress` is optional.

- [ ] **Step 5: Run all tests**

Run: `npx vitest run`
Expected: All tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/index.ts
git commit -m "feat: add immediate ack and progress forwarding to host"
```

---

### Task 4: Add progress emission to agent-runner

**Files:**
- Modify: `container/agent-runner/src/index.ts`

- [ ] **Step 1: Add progress marker constants and writeProgress function**

In `container/agent-runner/src/index.ts`, after the existing `OUTPUT_END_MARKER` constant (line 108) and `writeOutput` function (line 110-114), add:

```typescript
const PROGRESS_START_MARKER = '---NANOCLAW_PROGRESS_START---';
const PROGRESS_END_MARKER = '---NANOCLAW_PROGRESS_END---';

const PROGRESS_THROTTLE_MS = 12_000;
let lastProgressTime = 0;
let lastProgressMessage = '';

const TOOL_PROGRESS_MAP: Record<string, string> = {
  Bash: 'Running a command...',
  Read: 'Reading files...',
  Write: 'Writing code...',
  Edit: 'Writing code...',
  Glob: 'Searching the codebase...',
  Grep: 'Searching the codebase...',
  WebSearch: 'Searching the web...',
  WebFetch: 'Fetching a web page...',
  TodoWrite: 'Planning next steps...',
  Task: 'Spinning up a subagent...',
  TeamCreate: 'Spinning up a subagent...',
  SendMessage: 'Coordinating with agents...',
};

function getToolProgressMessage(toolName: string): string {
  return TOOL_PROGRESS_MAP[toolName] || `Using ${toolName}...`;
}

function maybeWriteProgress(message: string): void {
  const now = Date.now();
  // Deduplicate identical consecutive messages
  if (message === lastProgressMessage) return;
  // Throttle: at least PROGRESS_THROTTLE_MS between emissions
  if (now - lastProgressTime < PROGRESS_THROTTLE_MS) return;

  // Single console.log call to prevent interleaving with SDK stdout
  console.log(`${PROGRESS_START_MARKER}\n${JSON.stringify({ message })}\n${PROGRESS_END_MARKER}`);

  lastProgressTime = now;
  lastProgressMessage = message;
}
```

- [ ] **Step 2: Observe tool_use in assistant messages**

In the `runQuery` function, inside the `for await (const message of query(...))` loop (around line 432), after the existing `if (message.type === 'assistant' ...)` block (line 437-439), add tool_use observation:

```typescript
    if (message.type === 'assistant') {
      if ('uuid' in message) {
        lastAssistantUuid = (message as { uuid: string }).uuid;
      }
      // Observe tool_use for progress reporting
      try {
        const content = (message as { message?: { content?: unknown[] } }).message?.content;
        if (Array.isArray(content)) {
          for (const block of content) {
            if (block && typeof block === 'object' && 'type' in block && (block as { type: string }).type === 'tool_use') {
              const toolName = (block as { name?: string }).name;
              if (toolName) {
                maybeWriteProgress(getToolProgressMessage(toolName));
              }
            }
          }
        }
      } catch {
        // Defensive: progress observation must never throw
      }
    }
```

This replaces the existing `if (message.type === 'assistant' && 'uuid' in message)` block. The uuid tracking is preserved inside the new broader `if (message.type === 'assistant')` block.

- [ ] **Step 3: Observe task_notification for agent teams progress**

In the same `for await` loop, after the existing `task_notification` logging block (line 446-449), add progress emission:

```typescript
    if (message.type === 'system' && (message as { subtype?: string }).subtype === 'task_notification') {
      const tn = message as { task_id: string; status: string; summary: string };
      log(`Task notification: task=${tn.task_id} status=${tn.status} summary=${tn.summary}`);
      // Emit progress for agent team events
      try {
        if (tn.status === 'started' && tn.summary) {
          maybeWriteProgress(`Subagent working on: ${tn.summary}`);
        } else if (tn.status === 'completed' && tn.summary) {
          maybeWriteProgress(`Subagent finished: ${tn.summary}`);
        }
      } catch {
        // Defensive: progress observation must never throw
      }
    }
```

This replaces the existing `task_notification` block — the logging is preserved and progress emission is added.

- [ ] **Step 4: Reset progress timer on result emission**

In the `runQuery` function, inside the `if (message.type === 'result')` block (around line 451), after `writeOutput(...)`, add:

```typescript
      // Reset progress timer so first tool use after a result can report
      lastProgressTime = Date.now();
      lastProgressMessage = '';
```

- [ ] **Step 5: Verify the agent-runner builds inside the container context**

Run: `cd /Users/fab/nanoclaw/nanoclaw/container/agent-runner && npx tsc --noEmit`
Expected: Clean compilation.

If `tsc` isn't available directly, run:
```bash
cd /Users/fab/nanoclaw/nanoclaw/container/agent-runner && npx tsc --noEmit 2>&1 || echo "Check tsconfig — may need npm install first"
```

- [ ] **Step 6: Commit**

```bash
git add container/agent-runner/src/index.ts
git commit -m "feat: emit progress markers from agent-runner on tool use and agent teams"
```

---

### Task 5: Update agent-runner sync to always overwrite

**Files:**
- Modify: `src/container-runner.ts`

- [ ] **Step 1: Change the exists-check to unconditional copy**

In `src/container-runner.ts`, in the `buildVolumeMounts` function, find the agent-runner sync block (around line 193):

```typescript
  if (!fs.existsSync(groupAgentRunnerDir) && fs.existsSync(agentRunnerSrc)) {
    fs.cpSync(agentRunnerSrc, groupAgentRunnerDir, { recursive: true });
  }
```

Change to:

```typescript
  if (fs.existsSync(agentRunnerSrc)) {
    fs.cpSync(agentRunnerSrc, groupAgentRunnerDir, { recursive: true });
  }
```

This ensures all groups pick up agent-runner changes on every container startup.

- [ ] **Step 2: Verify build**

Run: `npm run build`
Expected: Clean compilation.

- [ ] **Step 3: Run all tests**

Run: `npx vitest run`
Expected: All tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/container-runner.ts
git commit -m "feat: always sync agent-runner source to group directories"
```

---

### Task 6: Rebuild container image

**Files:**
- None (build step only)

- [ ] **Step 1: Rebuild the container**

Run: `./container/build.sh`
Expected: Successful image build.

- [ ] **Step 2: Commit (if build script made changes)**

Only commit if the build script modified tracked files. Check with `git status` first.

---

### Task 7: End-to-end manual verification

- [ ] **Step 1: Start NanoClaw in dev mode**

Run: `npm run dev`
Expected: Starts without errors, channels connect.

- [ ] **Step 2: Send a test message via Telegram**

Send a message that triggers the bot (e.g. `@Andy what time is it?`).

Expected sequence:
1. Immediate `On it!` message appears in Telegram
2. One or more progress updates appear (e.g. `Running a command...`)
3. The actual agent response appears
4. All three types of messages are distinct in the chat

- [ ] **Step 3: Verify no regressions on follow-up messages**

Send a follow-up message while the container is still alive (piped path).

Expected: No ack for the follow-up, just the typing indicator and the agent's response.

- [ ] **Step 4: Stop dev mode**

Press Ctrl+C. Verify clean shutdown.
