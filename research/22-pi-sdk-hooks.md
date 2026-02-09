# Pi SDK Hooks & Extension System — Deep Dive

**Date:** 2025-02-09
**Source:** https://github.com/badlogic/pi-mono (packages: `pi-ai`, `pi-agent-core`, `pi-coding-agent`)

## Architecture Overview

Three layers:
1. **`@mariozechner/pi-ai`** — Low-level LLM streaming (no hooks, just stream/complete)
2. **`@mariozechner/pi-agent-core`** — Agent loop with event stream (`AgentEvent`)
3. **`@mariozechner/pi-coding-agent`** — Full extension system with rich lifecycle events

## TL;DR — Yes, We Can Do This

**Write a Pi extension** that hooks into `turn_end` and `agent_end` events. Every `AssistantMessage` contains full `Usage` data (tokens, cost). The extension system gives us everything we need.

---

## Layer 1: `@mariozechner/pi-ai` (Low-level)

No plugin/hook system. Just functions:
- `stream(model, context, options)` → `AssistantMessageEventStream`
- `complete(model, context, options)` → `AssistantMessage`
- `streamSimple()` / `completeSimple()` — with reasoning support

### Key Types

```typescript
interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: Api;          // e.g. "anthropic-messages"
  provider: Provider; // e.g. "anthropic"
  model: string;      // e.g. "claude-sonnet-4-20250514"
  usage: Usage;
  stopReason: StopReason; // "stop" | "length" | "toolUse" | "error" | "aborted"
  errorMessage?: string;
  timestamp: number;
}

interface Model<TApi> {
  id: string;
  name: string;
  api: TApi;
  provider: Provider;
  contextWindow: number;
  maxTokens: number;
  cost: { input, output, cacheRead, cacheWrite }; // $/million tokens
}
```

### Stream Events (AssistantMessageEvent)
```
start → text_start → text_delta → text_end → thinking_* → toolcall_* → done/error
```
Each event carries `partial: AssistantMessage` with running usage stats.

---

## Layer 2: `@mariozechner/pi-agent-core` (Agent Loop)

### AgentEvent Types (emitted by the agent loop)

```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId, toolName, args }
  | { type: "tool_execution_update"; toolCallId, toolName, args, partialResult }
  | { type: "tool_execution_end"; toolCallId, toolName, result, isError }
```

### Agent class (`Agent`)
- `subscribe(fn: (e: AgentEvent) => void)` — listen to all events
- State accessible via `agent.state` (messages, model, tools, isStreaming, etc.)
- Hooks: `convertToLlm`, `transformContext`, `getSteeringMessages`, `getFollowUpMessages`, `getApiKey`

---

## Layer 3: `@mariozechner/pi-coding-agent` — THE Extension System ⭐

This is where it all comes together. Full plugin architecture.

### Extension Structure

Extensions are `.ts` files placed in:
- `~/.pi/extensions/` (global)
- `<project>/.pi/extensions/` (project-local)
- Configured paths in pi config

An extension exports a factory function:

```typescript
import type { ExtensionFactory } from "@mariozechner/pi-coding-agent";

const extension: ExtensionFactory = (pi) => {
  // pi is ExtensionAPI — register handlers, tools, commands, etc.
};

export default extension;
```

### All Available Events (ExtensionAPI.on())

#### Agent Lifecycle Events
| Event | When | Data Available |
|-------|------|----------------|
| `before_agent_start` | After user submits prompt, before agent loop | `prompt`, `images`, `systemPrompt` |
| `agent_start` | Agent loop begins | — |
| `agent_end` | Agent loop ends | `messages: AgentMessage[]` (all new messages from this run) |
| `turn_start` | Each LLM call begins | `turnIndex`, `timestamp` |
| `turn_end` | Each LLM call + tool execution ends | `turnIndex`, `message: AgentMessage`, `toolResults: ToolResultMessage[]` |
| `context` | Before each LLM call (can modify messages) | `messages: AgentMessage[]` |

#### Tool Events
| Event | When | Data Available |
|-------|------|----------------|
| `tool_call` | Before tool executes (can block!) | `toolCallId`, `toolName`, `input` |
| `tool_result` | After tool executes (can modify result!) | `toolCallId`, `toolName`, `input`, `content`, `details`, `isError` |

#### Session Events
| Event | When |
|-------|------|
| `session_start` | Initial session load |
| `session_before_switch` / `session_switch` | Before/after switching sessions |
| `session_before_fork` / `session_fork` | Before/after forking |
| `session_before_compact` / `session_compact` | Before/after context compaction |
| `session_before_tree` / `session_tree` | Before/after tree navigation |
| `session_shutdown` | Process exit |

#### Other Events
| Event | When |
|-------|------|
| `model_select` | Model changed (set/cycle/restore) — gives `model`, `previousModel`, `source` |
| `input` | User input received (can transform or handle) |
| `user_bash` | User runs shell command via `!` prefix |
| `resources_discover` | Startup/reload — provide skill/prompt/theme paths |

### ExtensionContext (available in all handlers)

```typescript
interface ExtensionContext {
  ui: ExtensionUIContext;       // UI methods (notify, select, confirm, etc.)
  hasUI: boolean;
  cwd: string;                  // Current working directory
  sessionManager: ReadonlySessionManager;
  modelRegistry: ModelRegistry;
  model: Model<any> | undefined;
  isIdle(): boolean;
  abort(): void;
  hasPendingMessages(): boolean;
  shutdown(): void;
  getContextUsage(): ContextUsage | undefined;  // ⭐ Current context window usage
  compact(options?): void;
  getSystemPrompt(): string;
}

interface ContextUsage {
  tokens: number;           // Current token count
  contextWindow: number;    // Max context window
  percent: number;          // Usage percentage
  usageTokens: number;
  trailingTokens: number;
  lastUsageIndex: number | null;
}
```

### ExtensionAPI (available during registration)

Beyond event subscription, extensions can:
- `registerTool()` — add LLM-callable tools
- `registerCommand()` — add slash commands
- `registerShortcut()` — add keyboard shortcuts
- `registerFlag()` — add CLI flags
- `registerMessageRenderer()` — custom message rendering
- `registerProvider()` — register new LLM providers
- `sendMessage()` / `sendUserMessage()` — inject messages
- `appendEntry()` — persist data to session
- `setModel()` / `getThinkingLevel()` / `setThinkingLevel()`
- `events: EventBus` — inter-extension communication

---

## Data Available Per Turn (for stats collection)

From `turn_end` event's `message: AgentMessage` (which is `AssistantMessage`):

| Field | Description |
|-------|-------------|
| `message.usage.input` | Input tokens |
| `message.usage.output` | Output tokens |
| `message.usage.cacheRead` | Cache read tokens |
| `message.usage.cacheWrite` | Cache write tokens |
| `message.usage.totalTokens` | Total tokens |
| `message.usage.cost.total` | Total cost ($) |
| `message.usage.cost.input` | Input cost ($) |
| `message.usage.cost.output` | Output cost ($) |
| `message.model` | Model ID string |
| `message.provider` | Provider string |
| `message.api` | API type string |
| `message.stopReason` | Why the turn ended |
| `message.timestamp` | Unix timestamp ms |
| `message.content` | Array of text/thinking/toolCall blocks |
| `toolResults` | Array of tool result messages |

From `ctx` (ExtensionContext):
| Field | Description |
|-------|-------------|
| `ctx.cwd` | Project path |
| `ctx.model.contextWindow` | Max context window |
| `ctx.model.cost` | Cost per million tokens |
| `ctx.getContextUsage()` | Current context usage (tokens, percent) |
| `ctx.sessionManager` | Session metadata |

From `tool_result` events:
| Field | Description |
|-------|-------------|
| `toolName` | Which tool was called (bash, read, edit, write, grep, find, ls, or custom) |
| `input` | Tool arguments |
| `details` | Tool-specific details (e.g., `EditToolDetails` has file paths) |

---

## Blueprint: Stats Collection Extension

```typescript
import type { ExtensionFactory } from "@mariozechner/pi-coding-agent";

const statsExtension: ExtensionFactory = (pi) => {
  let turnIndex = 0;
  let toolCalls: Array<{ name: string; args: any }> = [];
  let filesModified: Set<string> = new Set();
  let agentStartTime: number = 0;

  pi.on("agent_start", () => {
    turnIndex = 0;
    toolCalls = [];
    filesModified = new Set();
    agentStartTime = Date.now();
  });

  // Track tool calls
  pi.on("tool_result", (event) => {
    toolCalls.push({ name: event.toolName, args: event.input });
    
    // Track files touched
    if (event.toolName === "edit" || event.toolName === "write") {
      const filePath = (event.input as any).filePath || (event.input as any).file_path;
      if (filePath) filesModified.add(filePath);
    }
    if (event.toolName === "read") {
      const filePath = (event.input as any).filePath || (event.input as any).file_path;
      if (filePath) filesModified.add(filePath);
    }
  });

  // Log stats per turn
  pi.on("turn_end", async (event, ctx) => {
    turnIndex++;
    const msg = event.message;
    if (msg.role !== "assistant") return;

    const contextUsage = ctx.getContextUsage();

    const stats = {
      datetime: new Date().toISOString(),
      projectPath: ctx.cwd,
      model: msg.model,
      provider: msg.provider,
      api: msg.api,
      agentName: "pi-coding-agent", // or derive from config
      turnIndex,
      stopReason: msg.stopReason,
      
      // Token usage
      inputTokens: msg.usage.input,
      outputTokens: msg.usage.output,
      cacheReadTokens: msg.usage.cacheRead,
      cacheWriteTokens: msg.usage.cacheWrite,
      totalTokens: msg.usage.totalTokens,
      
      // Cost
      costTotal: msg.usage.cost.total,
      costInput: msg.usage.cost.input,
      costOutput: msg.usage.cost.output,
      costCacheRead: msg.usage.cost.cacheRead,
      costCacheWrite: msg.usage.cost.cacheWrite,
      
      // Context
      contextTokens: contextUsage?.tokens,
      contextWindow: contextUsage?.contextWindow,
      contextPercent: contextUsage?.percent,
      
      // Tool calls this turn
      toolCallsThisTurn: event.toolResults.map(r => ({
        name: r.toolName,
        isError: r.isError,
      })),
    };

    // Ship to MongoDB
    await sendToMongo(stats);
  });

  // Log session-level stats
  pi.on("agent_end", async (event, ctx) => {
    const sessionStats = {
      datetime: new Date().toISOString(),
      projectPath: ctx.cwd,
      totalTurns: turnIndex,
      totalToolCalls: toolCalls.length,
      filesModified: Array.from(filesModified),
      durationMs: Date.now() - agentStartTime,
    };
    await sendToMongo(sessionStats);
  });

  // Track compaction events
  pi.on("session_compact", (event) => {
    // Log compaction event — context was compressed
  });

  // Track model changes
  pi.on("model_select", (event) => {
    // Log model switches
  });
};

export default statsExtension;
```

---

## How Existing Extensions Hook In

All extensions use the same `ExtensionFactory` pattern:

1. **pi-subagents** — Would use `registerTool()` to add a subagent spawn tool, `turn_end`/`agent_end` to track subagent lifecycle
2. **pi-interactive-shell** — Uses `user_bash` event to intercept shell commands, `registerCommand()` for slash commands
3. **pi-messenger** — Uses `agent_end` to deliver responses, `input` to receive messages

Extensions are loaded via `jiti` (TypeScript JIT compiler), so they can be plain `.ts` files — no build step needed.

---

## Key Findings

1. **✅ Full extension API exists** — `@mariozechner/pi-coding-agent` has a comprehensive extension system
2. **✅ `turn_end` gives us everything** — `AssistantMessage` has full `Usage` (tokens + cost), model info, tool calls
3. **✅ `getContextUsage()` available** — Context window usage (tokens, percent, window size)
4. **✅ `tool_result` events** — Track every tool call with full input/output/error status
5. **✅ Files touched** — Available from tool_result events for edit/write/read tools
6. **✅ Compaction tracking** — `session_before_compact` / `session_compact` events
7. **✅ No build step** — Extensions are `.ts` files loaded via jiti
8. **✅ Inter-extension communication** — `EventBus` for custom events between extensions
9. **⚠️ Extension runs inside pi process** — MongoDB writes should be async/non-blocking
10. **⚠️ No dedicated "cost per session" accumulator** — We'd compute this ourselves from turn data

## Extension Placement

Place the extension at:
- `~/.pi/extensions/stats-collector.ts` (global — all projects)
- OR `<project>/.pi/extensions/stats-collector.ts` (project-specific)

## What We Can't Get (easily)

- **Agent name** — Not a first-class concept; would need to be configured per-project
- **Total session cost** — Must accumulate from individual turns
- **Subagent vs main agent distinction** — Would need to coordinate via EventBus or file
- **Pre-compaction full context** — Available via `session_before_compact` event's `preparation` field
