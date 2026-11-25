# Batched tool calls

## Goal

The model can request multiple independent tools in one response. The runtime returns one result for each request. A routine code review aims to finish in 2 to 4 model turns.

## Wire format

Use one `<tool_calls>` element with ordered `<tool_call>` children. Each child can have a unique `id`. The runtime gives calls without an `id` a generated result ID. The runtime returns one `<tool_results>` element. Its `<tool_result>` children follow request order. Each result includes the request ID.

Single calls keep the existing `<tool_call>` and `<tool_result>` format. The parser accepts repeated direct `<tool_call>` elements for compatibility. It rejects mixed direct and wrapped calls. It also rejects duplicate explicit IDs.

## Runtime order

1. Parse and validate each call.
2. Apply policy to each valid call.
3. Resolve all required approvals before any tool starts.
4. Process calls in request order. Run `beforeToolCall`, execute the approved tool, save its result, then run `afterToolCall`.
5. Send all results in one message.

One denied or invalid call does not stop another valid call. Approval errors stop the batch before execution. The runtime saves each completed result before it starts the next call. A resumed run can return saved results without repeating a completed side effect.

## Hook interface

`AgentRuntime` accepts an optional `hooks` object:

```js
{
  beforeToolCall: async ({ toolCall, decision, skipped }) => {},
  afterToolCall: async ({ toolCall, result }) => {},
}
```

The runtime attempts both hooks once for each batch member, including denied and invalid calls. `skipped` is true when the call already has a result. Hooks receive independent snapshots. They do not approve, deny, or change the executed call or saved result. A Hook error or snapshot error creates a `tool.hook_failed` event. The runtime continues when that event is saved and reported. A session write error or event callback error can still stop the batch.

## Review target

The prompt asks the model to batch independent reads in its first review turn. It sets a target of 2 to 4 model turns for a routine code review. The target is advisory. Reviews can use more turns when they need more evidence. The tool round warning remains advisory and does not end a run.
