---
name: skill-feedback-collector
description: MCP-based human feedback collector that suspends AI threads via WebSocket UI to prevent premature conversation endings. Use when working with token-limited coding plans (Tencent Cloud, Alibaba Cloud), when you need human-in-the-loop confirmation before ending a task, or when you want to maximize each conversation round by collecting feedback instead of guessing.
metadata: {"clawdbot":{"emoji":"💬","requires":{"anyBins":["node","npm"]},"install":[{"id":"npm-install","kind":"node","package":"skill-feedback-collector","bins":["node"],"label":"Install dependencies: npm install && npm run build"}],"os":["linux","darwin","win32"]}}
---

# Skill Feedback Collector

A human-in-the-loop MCP skill that suspends AI threads and waits for real human feedback through a WebSocket-connected web UI. Designed for token-limited coding plans (Tencent Cloud, Alibaba Cloud) to maximize the value of each conversation round.

## When to Use

- After completing any coding task, before ending the conversation
- When encountering uncertain parameters (colors, layouts, business logic) — never guess
- When a task is done and you need the user to verify the result
- When you need clarification on ambiguous requirements
- When the user is on a per-round billing plan and wants to continue working in the same session
- Before making any destructive or irreversible changes

## Critical Rules

**YOU MUST FOLLOW THESE RULES WITHOUT EXCEPTION:**

1. **NEVER end a conversation on your own.** Always call `ask_human_feedback` to ask the user if they want to continue or end.
2. **NEVER guess uncertain parameters.** If you are unsure about colors, sizes, layouts, API choices, business logic, or any implementation detail, call `ask_human_feedback` with your question.
3. **After every completed task**, call `ask_human_feedback` with a summary of what you did and ask if the user wants to continue with more tasks.
4. **After every error fix**, call `ask_human_feedback` to confirm the fix works and ask for next steps.
5. **The user's response from `ask_human_feedback` is your next instruction.** Follow it precisely.

## How It Works

### Architecture

```
┌─────────────┐     stdio      ┌──────────────────┐     WebSocket     ┌──────────────┐
│  AI Agent   │◄──────────────►│  MCP Server      │◄────────────────►│  Browser UI  │
│  (OpenClaw) │  MCP Protocol  │  (Node.js)       │   Port 18061     │  (index.html)│
└─────────────┘                └──────────────────┘                   └──────────────┘
```

1. AI calls `ask_human_feedback` tool with a `reason` (summary + question)
2. MCP Server broadcasts the reason to all connected browser clients via WebSocket
3. The AI thread is **suspended** (Promise pending) — no tokens consumed while waiting
4. Human reads the question in the browser UI and types a response
5. Response is sent back through WebSocket → MCP Server resolves the Promise
6. AI receives the human's text and continues working

### Setup

```bash
# Install dependencies
npm install

# Build TypeScript
npm run build

# The MCP server starts via stdio, WebSocket UI on port 3001
# Configure in your OpenClaw MCP settings:
# Command: node
# Args: ["build/index.js"]
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `FEEDBACK_PORT` | `18061` | Port for WebSocket + HTTP UI server |

### Accessing the UI

Open your browser and navigate to:
```
http://<your-server-ip>:18061
```

The UI auto-connects via WebSocket. When the AI calls `ask_human_feedback`, the question appears in the UI and you can type your response.

## Tool Reference

### `ask_human_feedback`

Suspends the AI thread and waits for human input. If feedback mode is disabled, returns immediately without waiting.

**Parameters:**
- `reason` (string, required): A clear summary of completed work and what you need from the user.

**Returns:** The human's text response as a string (or bypass message if disabled).

### `set_feedback_mode`

Enable or disable feedback confirmation mode.

**Parameters:**
- `enabled` (boolean, required): `true` to enable, `false` to disable.

**When to use:**
- User says "自由模式" / "free mode" → call with `enabled: false`
- User says "确认模式" / "feedback mode" → call with `enabled: true`
- The mode can also be toggled from the browser UI

**Example usage pattern:**

```
After completing a task:
→ call ask_human_feedback with reason:
  "✅ I've completed the user login API endpoint.
   - Added POST /api/login route
   - Implemented JWT token generation
   - Added input validation

   Would you like me to:
   1. Add unit tests for this endpoint?
   2. Move on to the registration endpoint?
   3. Something else?
   
   Or type 'done' if you want to end this session."
```

```
When encountering uncertainty:
→ call ask_human_feedback with reason:
  "❓ I need your input on the database schema:
   - Should the `users` table use UUID or auto-increment for the primary key?
   - Do you need a `deleted_at` column for soft deletes?
   
   Please advise."
```

## Conversation Flow Example

```
AI: [completes task] → calls ask_human_feedback("✅ Task done. Continue?")
    ⏸️ AI thread suspended, no tokens consumed
Human: [reads in browser] → "Yes, now add error handling"
AI: [receives response] → works on error handling
AI: [completes task] → calls ask_human_feedback("✅ Error handling added. Next?")
    ⏸️ AI thread suspended again
Human: "Looks good, let's end here"
AI: [receives response] → ends gracefully
```

## Tips

- The WebSocket server binds to `0.0.0.0` so it's accessible from any network interface — make sure your server firewall allows the port.
- The UI auto-reconnects every 3 seconds if the connection drops, so you won't miss any AI questions.
- Press Enter to send feedback quickly (Shift+Enter for newlines).
- The conversation history is displayed in the UI so you can track the full interaction flow.
- Set `FEEDBACK_PORT` environment variable to change the default port if 18061 is occupied.
- For production servers, consider putting the UI behind a reverse proxy (nginx) with authentication.
- The `reason` parameter should be detailed — include what was done, what's pending, and specific options for the user.
- This skill is especially valuable for per-round billing plans where ending a conversation means starting a new billable round.
