---
name: skill-feedback-collector
description: Human-in-the-loop MCP feedback collector with task queue. Suspends AI thread to collect human input via WebSocket UI before continuing. Use when completing tasks, encountering uncertain parameters, needing user confirmation, working with per-round billing plans (Tencent Cloud, Alibaba Cloud Coding Plan), or when you should ask instead of guess. Also covers batch task execution via auto-dequeue.
metadata: {"clawdbot":{"emoji":"💬","requires":{"anyBins":["node","npm"]},"install":[{"id":"npm-install","kind":"node","package":"skill-feedback-collector","bins":["node"],"label":"npm install && npm run build"}],"os":["linux","darwin","win32"]}}
---

# Skill Feedback Collector

Suspend the AI agent loop and wait for real human feedback via WebSocket-connected browser UI. Supports batch task execution through an auto-dequeue task queue.

## When to Use

- After completing any task — ask user to verify before moving on
- When encountering uncertain parameters (colors, layouts, API choices, business logic)
- When you need user confirmation before destructive or irreversible changes
- When the user wants to maximize a single conversation round (per-round billing)
- When you want to batch-execute multiple tasks without manual intervention per task

## Instructions

**MANDATORY BEHAVIOR — follow without exception:**

1. **NEVER end your response without calling `ask_human_feedback`.** Your response must always end with a tool call to `ask_human_feedback`. This keeps the agent loop alive within the same session run.

2. **NEVER guess.** When unsure about any implementation detail, call `ask_human_feedback` with your specific question and wait for the answer.

3. **After every completed task**, call `ask_human_feedback` with:
   - A summary of what you did (be specific)
   - What you need from the user (verify, continue, or end)
   - Numbered options for next steps

4. **Treat the response from `ask_human_feedback` as your next instruction.** Execute it immediately, then call `ask_human_feedback` again when done. This creates a continuous work loop.

5. **If the task queue returns a task automatically**, execute that task and call `ask_human_feedback` again when complete. The queue will auto-feed the next task until empty.

6. **Mode control:**
   - User says "自由模式" / "free mode" → call `set_feedback_mode(enabled: false)` → work autonomously
   - User says "确认模式" / "feedback mode" → call `set_feedback_mode(enabled: true)` → resume confirmation loop

## Safety Rules

- Never end a session or conversation on your own — always ask the user first
- Never commit destructive changes (force push, drop table, delete files) without explicit user confirmation via `ask_human_feedback`
- If `ask_human_feedback` returns a bypass message (feedback mode disabled), continue working without pausing

## Tools

### `ask_human_feedback`

Suspends the agent loop. Broadcasts `reason` to browser UI via WebSocket. Returns human's text response. If task queue is non-empty, auto-dequeues the next task and returns it immediately (1.5s delay for UI visibility).

**Parameters:** `reason` (string) — summary of work done + what you need from the user.

**Example reason format:**
```
✅ Completed: [specific work done]
📋 Changes: [files modified, endpoints added, etc.]

What would you like me to do next?
1. [Option A]
2. [Option B]
3. Something else — please describe
```

### `set_feedback_mode`

Toggle feedback confirmation on/off. When off, `ask_human_feedback` returns immediately without suspending.

**Parameters:** `enabled` (boolean)

## Setup

```bash
npm install && npm run build
```

MCP configuration:
```json
{
  "command": "node",
  "args": ["build/index.js"],
  "cwd": "/path/to/skill-feedback-collector"
}
```

Browser UI: `http://<server-ip>:18061`

| Env Variable | Default | Description |
|---|---|---|
| `FEEDBACK_PORT` | `18061` | HTTP + WebSocket port |
| `FEEDBACK_TOKEN` | (empty) | Optional auth token |

## Agent Loop Flow

```
User message → Agent works → calls ask_human_feedback("Done. Next?")
                                    ↓
                    [Queue has tasks?] → YES → auto-return next task → Agent continues
                                    ↓ NO
                    [Wait for human input via browser UI]
                                    ↓
                    Human responds → Agent receives → works → calls ask_human_feedback again
                                    ↓
                    ... loop continues until user says "done" or "end" ...
```

## Tips

- The task queue lets users pre-load multiple tasks; AI executes them sequentially without pausing
- Users can add tasks to the queue while AI is working — they'll be picked up automatically
- WebSocket server binds to `0.0.0.0` — ensure firewall allows the port
- HTTP long-polling fallback activates automatically when WebSocket is unavailable
- Browser notifications + sound alert when AI asks a question
- History is persisted to `feedback-history.json` (max 500 entries)
- Use `FEEDBACK_TOKEN` to protect the UI when deployed on public servers
