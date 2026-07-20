# Agent Runtime Debug Ability

Last updated: 2026-07-20

This note documents how to debug Agent runtime call context in this repository. It covers the standalone `Agent` runtime from `@cline/agents` and the `ClineCore` / hub path that wraps it.

## What To Inspect

Use two complementary debug surfaces:

1. Runtime events for the timeline.
2. Runtime hooks and tool `execute(input, context)` for exact call context.

For standalone `Agent`, subscribe before calling `run()` so early events are not missed:

```ts
const unsubscribe = agent.subscribe((event) => {
  console.log("runtime event", event.type, {
    iteration: event.snapshot.iteration,
    runId: event.snapshot.runId,
    pendingToolCalls: event.snapshot.pendingToolCalls,
  })
})

const result = await agent.run("Debug this task")
unsubscribe()
```

For async logging or instrumentation, use `hooks.onEvent`:

```ts
const agent = new Agent({
  ...config,
  hooks: {
    onEvent: async (event) => {
      await writeDebugLog({
        type: event.type,
        runId: event.snapshot.runId,
        iteration: event.snapshot.iteration,
      })
    },
  },
})
```

## Tool Call Context

The tool context passed to `tool.execute(input, context)` includes:

```ts
{
  sessionId,
  agentId,
  conversationId,
  runId,
  iteration,
  toolCallId,
  signal,
  metadata,
  snapshot,
  emitUpdate,
}
```

Minimal tool-side debug pattern:

```ts
const debugTool = createTool({
  name: "debug_context",
  description: "Print the current runtime call context.",
  inputSchema: z.object({ label: z.string().optional() }),
  execute: async (input, context) => {
    console.dir(
      {
        input,
        sessionId: context.sessionId,
        agentId: context.agentId,
        conversationId: context.conversationId,
        runId: context.runId,
        iteration: context.iteration,
        toolCallId: context.toolCallId,
        metadata: context.metadata,
        snapshot: {
          status: context.snapshot?.status,
          iteration: context.snapshot?.iteration,
          pendingToolCalls: context.snapshot?.pendingToolCalls,
          messageCount: context.snapshot?.messages.length,
          usage: context.snapshot?.usage,
          lastError: context.snapshot?.lastError,
        },
      },
      { depth: 5 },
    )

    context.emitUpdate?.({
      type: "debug",
      label: input.label,
      toolCallId: context.toolCallId,
    })

    return { ok: true }
  },
})
```

Long-running tools should also watch the abort signal:

```ts
if (context.signal?.aborted) {
  return { aborted: true }
}
```

## Hook-Based Debugging

Use hooks when you need to observe or alter model requests, tool inputs, tool policies, or tool results.

```ts
const debugHooks = {
  beforeModel(ctx) {
    console.log("beforeModel", {
      iteration: ctx.snapshot.iteration,
      runId: ctx.snapshot.runId,
      messages: ctx.request.messages.length,
      tools: ctx.request.tools?.map((tool) => tool.name),
      options: ctx.request.options,
    })
  },

  afterModel(ctx) {
    console.log("afterModel", {
      iteration: ctx.snapshot.iteration,
      finishReason: ctx.finishReason,
      messageId: ctx.assistantMessage.id,
    })
  },

  beforeTool(ctx) {
    console.log("beforeTool", {
      iteration: ctx.snapshot.iteration,
      tool: ctx.tool.name,
      toolCallId: ctx.toolCall.toolCallId,
      input: ctx.input,
    })
  },

  afterTool(ctx) {
    console.log("afterTool", {
      tool: ctx.tool.name,
      toolCallId: ctx.toolCall.toolCallId,
      durationMs: ctx.durationMs,
      result: ctx.result,
    })
  },

  onEvent(event) {
    console.log("event", event.type, {
      runId: event.snapshot.runId,
      iteration: event.snapshot.iteration,
      status: event.snapshot.status,
    })
  },
}
```

You can also use `beforeTool` to normalize or replace bad input:

```ts
beforeTool(ctx) {
  if (ctx.tool.name === "example_tool" && typeof ctx.input === "string") {
    return { input: { value: ctx.input } }
  }
}
```

Or block a tool call during debugging:

```ts
beforeTool(ctx) {
  if (ctx.tool.name === "dangerous_tool") {
    return {
      skip: true,
      reason: "Blocked by debug hook",
    }
  }
}
```

## Event Types Worth Watching

Standalone `Agent` emits `AgentRuntimeEvent` values:

```ts
run-started
message-added
turn-started
assistant-text-delta
assistant-reasoning-delta
assistant-message
tool-started
tool-updated
tool-finished
usage-updated
turn-finished
status-notice
run-finished
run-failed
```

Common event debug filter:

```ts
agent.subscribe((event) => {
  switch (event.type) {
    case "tool-started":
    case "tool-updated":
    case "tool-finished":
    case "run-failed":
    case "run-finished":
      console.dir(event, { depth: 5 })
      break
  }
})
```

## Source Breakpoints

Set breakpoints or temporary logs at these locations:

| Purpose | File |
| --- | --- |
| Tool context interface | `sdk/packages/shared/src/agent.ts` |
| Runtime snapshot interface | `sdk/packages/shared/src/agent.ts` |
| `beforeTool` hook invocation | `sdk/packages/agents/src/agent-runtime.ts` |
| Actual `tool.execute(input, context)` call | `sdk/packages/agents/src/agent-runtime.ts` |
| `tool-started`, `tool-updated`, `tool-finished` events | `sdk/packages/agents/src/agent-runtime.ts` |
| ClineCore hook merging | `sdk/packages/core/src/runtime/orchestration/session-runtime-orchestrator.ts` |
| ClineCore runtime event translation | `sdk/packages/core/src/runtime/orchestration/session-runtime-orchestrator.ts` |
| Hub tool context parsing | `sdk/packages/core/src/hub/runtime-host/hub-runtime-host.ts` |

The most important methods in `sdk/packages/agents/src/agent-runtime.ts` are:

```txt
AgentRuntime.snapshot()
AgentRuntime.execute()
AgentRuntime.executeToolCalls()
AgentRuntime.prepareToolExecution()
AgentRuntime.executePreparedTool()
```

The critical flow is:

```txt
execute()
  -> model request
  -> assistant message with tool calls
  -> executeToolCalls()
  -> prepareToolExecution()
  -> beforeTool hooks
  -> policy / approval
  -> executePreparedTool()
  -> tool.execute(input, context)
  -> afterTool hooks
  -> tool-finished event
```

## ClineCore And Hub Notes

If you debug through `ClineCore`, subscribe with `cline.subscribe()` instead of `agent.subscribe()`:

```ts
cline.subscribe((event) => {
  if (event.type === "chunk" && event.payload.type === "text") {
    process.stdout.write(event.payload.text)
  }

  if (event.type === "agent_event") {
    console.dir(event.payload.event, { depth: 5 })
  }

  if (event.type === "ended") {
    console.log("session ended", event.payload.finishReason)
  }
})
```

`ClineCore` translates runtime events into session events. Do not mix these event names with standalone `Agent` event names:

| Standalone Agent | ClineCore |
| --- | --- |
| `assistant-text-delta` | `chunk` with `payload.type === "text"` |
| `tool-started` | projected through `agent_event` / hook events |
| `run-finished` | `ended` |

Hub caveat: cross-process hub tool execution narrows the tool context. In `hub-runtime-host.ts`, `parseToolContext()` currently keeps only:

```ts
{
  agentId,
  conversationId,
  iteration,
  metadata,
  signal,
}
```

That means `runId`, `toolCallId`, and `snapshot` may be missing inside hub-contributed or client-contributed tools. If those fields matter, log them before the hub boundary or extend the hub context serialization.

## Quick Checklist

1. Register `agent.subscribe()` before `run()`.
2. Add `hooks.beforeModel` to see the final model request.
3. Add `hooks.beforeTool` to inspect and optionally rewrite tool input.
4. Add logging inside `tool.execute(input, context)` for actual call context.
5. Use `context.emitUpdate()` to stream progress from long-running tools.
6. Add `hooks.afterTool` to inspect output and duration.
7. Use `agent.snapshot()` after completion to inspect final messages, usage, pending tool calls, and last error.
8. If using `ClineCore` or hub, verify whether the tool executes locally or across the hub boundary.

## Common Failure Modes

| Symptom | What to check |
| --- | --- |
| Missing early events | Subscribe before `run()` starts. |
| Tool context has no `snapshot` | The tool may be hub-contributed across process boundaries. |
| Tool never completes | Check `signal.aborted`, pending promises, and whether the model is repeatedly calling tools. |
| Run stops with mistake limit | Return structured error data from tools instead of throwing. |
| Tool input shape is wrong | Inspect `beforeTool`, schema descriptions, and parsed `toolCall.input`. |
| ClineCore event names look different | You are looking at translated `CoreSessionEvent`, not `AgentRuntimeEvent`. |

