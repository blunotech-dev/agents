---
name: tool-execution-loop
description: Implements the agentic tool execution loop for Anthropic API artifacts — covers tool call detection, dispatch to handler, result injection, and multi-turn continuation until the model signals completion. Use this skill when building any AI-powered artifact that needs Claude to call tools (web search, MCP, custom functions) and loop until a final text response is produced.
category: "Ai Integration"
---

# Tool Execution Loop

## What This Skill Covers

Only the non-obvious parts of building a reliable tool loop. Assumes you already know how to make a basic `/v1/messages` call.

---

## Core Loop Shape

```js
async function runLoop(messages, tools) {
  while (true) {
    const res = await callClaude({ messages, tools });
    messages.push({ role: "assistant", content: res.content });

    if (res.stop_reason === "end_turn") return res.content;

    const toolResults = await dispatchAll(res.content);
    messages.push({ role: "user", content: toolResults });
  }
}
```

**Non-obvious:** `stop_reason === "tool_use"` means the model wants tool results injected. `stop_reason === "end_turn"` means it's done. Never check for absence of tool blocks — always check `stop_reason`.

---

## Tool Call Detection

A single response can contain **mixed blocks** — text + multiple tool calls interleaved. Collect all `tool_use` blocks regardless of position:

```js
const toolCalls = res.content.filter(b => b.type === "tool_use");
```

Do **not** assume tool calls are at the end or that there's only one. The model can emit N tool calls in one turn (parallel tool use).

---

## Dispatching: Parallel vs Serial

Run independent tool calls in parallel. They share no state — `Promise.all` is safe and faster:

```js
async function dispatchAll(contentBlocks) {
  const calls = contentBlocks.filter(b => b.type === "tool_use");
  const results = await Promise.all(calls.map(dispatch));
  return results; // array of tool_result blocks
}
```

**When to serialize instead:** only if tool B depends on tool A's output. In that case the model will naturally emit them in separate turns, not the same one — so parallel dispatch is almost always correct.

---

## Result Injection Shape

Tool results go back as a `user` turn. Each result block must echo the original `tool_use_id`:

```js
async function dispatch(toolCall) {
  const output = await handlers[toolCall.name](toolCall.input);
  return {
    type: "tool_result",
    tool_use_id: toolCall.id,        // must match exactly
    content: stringify(output),       // always stringify; never pass objects
  };
}
```

**Non-obvious pitfalls:**
- Missing `tool_use_id` → API error, not a silent failure
- Passing an object instead of a string to `content` → silently wrong outputs
- Omitting the `user` role wrapper around the results array → API error

Correct wrapper:
```js
messages.push({
  role: "user",
  content: toolResults   // array of tool_result blocks
});
```

---

## Error Handling Inside the Loop

Surface tool errors back to the model rather than throwing — it can recover or ask the user:

```js
async function dispatch(toolCall) {
  try {
    const output = await handlers[toolCall.name](toolCall.input);
    return { type: "tool_result", tool_use_id: toolCall.id, content: JSON.stringify(output) };
  } catch (err) {
    return {
      type: "tool_result",
      tool_use_id: toolCall.id,
      content: `Error: ${err.message}`,
      is_error: true,         // tells the model this was a failure
    };
  }
}
```

**Non-obvious:** if you throw instead of returning `is_error: true`, you break the conversation history and the model has no context about what failed.

---

## Loop Termination Guards

Prevent infinite loops from misbehaving tool schemas or unexpected model behavior:

```js
const MAX_TURNS = 10;
let turns = 0;

while (true) {
  if (++turns > MAX_TURNS) throw new Error("Loop exceeded max turns");
  // ...
}
```

Also guard against the model emitting `tool_use` blocks for handlers you haven't registered — throw loudly at dispatch time, not silently swallow:

```js
if (!handlers[toolCall.name]) {
  throw new Error(`No handler registered for tool: ${toolCall.name}`);
}
```

---

## Streaming Variant (non-obvious delta accumulation)

If using streaming (`stream: true`), tool call arguments arrive as fragmented `input_json_delta` events. You must accumulate per `index`:

```js
const toolInputs = {};
for await (const event of stream) {
  if (event.type === "content_block_start" && event.content_block.type === "tool_use") {
    toolInputs[event.index] = { id: event.content_block.id, name: event.content_block.name, rawInput: "" };
  }
  if (event.type === "content_block_delta" && event.delta.type === "input_json_delta") {
    toolInputs[event.index].rawInput += event.delta.partial_json;
  }
  if (event.type === "message_stop") break;
}
// After stream ends, parse accumulated inputs
const calls = Object.values(toolInputs).map(t => ({ ...t, input: JSON.parse(t.rawInput) }));
```

**Non-obvious:** `input_json_delta` is partial JSON — never try to parse mid-stream. Always wait for `message_stop` before parsing any tool input.

---

## Conversation History Growth

Each loop iteration appends 2 turns (assistant + user). For long-running loops this balloons context fast. Prune aggressively for intermediate tool results the model no longer needs:

```js
// Keep only the last N tool result turns, always keep the first user message
function pruneHistory(messages, keepLast = 3) {
  const toolResultTurns = messages.filter(m => 
    Array.isArray(m.content) && m.content.some(b => b.type === "tool_result")
  );
  const toRemove = toolResultTurns.slice(0, -keepLast);
  return messages.filter(m => !toRemove.includes(m));
}
```

---

## MCP Tools in the Loop

MCP tool calls arrive identically to native tools (`tool_use` blocks). The difference is in the API request — pass `mcp_servers` instead of (or alongside) `tools`. Dispatch logic is the same; you do not write handlers for MCP tools — the API calls the MCP server and returns a `tool_result` automatically.

**Non-obvious:** when mixing MCP and custom tools, you only need handlers for your custom tools. MCP results are injected by the API; you just continue the loop normally.

---

## Checklist

- [ ] Loop condition checks `stop_reason`, not content shape
- [ ] All `tool_use` blocks collected before dispatch (not just first)
- [ ] Parallel dispatch via `Promise.all` unless explicit dependency
- [ ] `tool_use_id` echoed exactly in every result block
- [ ] Tool errors returned as `is_error: true`, not thrown
- [ ] Max turn guard in place
- [ ] Streaming: input parsed only after `message_stop`
- [ ] History pruning for long loops