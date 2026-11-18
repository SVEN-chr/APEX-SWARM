# R1 — Anthropic Go SDK capabilities (for a provider-neutral LLM contract)

Verifies `github.com/anthropics/anthropic-sdk-go` against primary sources. Scope: prompt caching, tool use + streaming, response `Usage`, extended thinking. All Go identifiers below were read from the SDK's generated source (`message.go`) and its `examples/`; all wire/semantic rules from the official Anthropic docs.

**Sources**
- SDK source: `message.go` — <https://github.com/anthropics/anthropic-sdk-go/blob/main/message.go>
- SDK examples: `examples/tools/main.go`, `examples/message-streaming/main.go` (dir: <https://github.com/anthropics/anthropic-sdk-go/tree/main/examples>)
- SDK README: <https://github.com/anthropics/anthropic-sdk-go/blob/main/README.md>
- Prompt caching: <https://platform.claude.com/docs/en/build-with-claude/prompt-caching>
- Extended thinking: <https://platform.claude.com/docs/en/build-with-claude/extended-thinking>

Date verified: 2026-11-18.

---

## 1. Prompt caching (`cache_control` breakpoints)

**Wire shape.** A breakpoint is a `cache_control` object attached to a block: `"cache_control": {"type": "ephemeral"}`, optionally `{"type": "ephemeral", "ttl": "1h"}` (default TTL 5m; `ephemeral` is the only cache type). [caching docs]

**Go representation.** `anthropic.CacheControlEphemeralParam{ TTL: ... }` with `Type` defaulting to `ephemeral`. `TTL` is `CacheControlEphemeralTTL`, values `"5m"`/`"1h"`. It appears as a `CacheControl CacheControlEphemeralParam` field (tag `cache_control,omitzero`) on ~20+ request block param types — e.g. `ToolParam`, `TextBlockParam`, `ImageBlockParam`, `DocumentBlockParam`, `ToolUseBlockParam`, `ToolResultBlockParam`. You set the breakpoint by populating that field on the specific block you want to end the cached prefix. [message.go lines 3906 (`CacheControlEphemeralParam`), many `CacheControl` fields e.g. 469/2694/3511/…]

**Where it attaches.** On individual content blocks in three request sections, cached as the prefix `tools` → `system` → `messages` in that order:
- `tools` — tool definitions
- `system` — blocks in the system array
- `messages[].content` — text/image/document in user turns; text, `tool_use`, and `tool_result` blocks in both user and assistant turns

Not directly cacheable: thinking blocks (cached only as part of a prior assistant turn), sub-content like citations, empty text blocks. [caching docs]

**How many breakpoints.** **Max 4 explicit breakpoints per request.** Automatic caching (a single top-level `cache_control` on the request body, which the system slides to the last cacheable block) consumes one of the 4 slots; requesting it when 4 explicit breakpoints already exist returns 400. Automatic caching is unavailable on Amazon Bedrock. A breakpoint caches everything up to and including the marked block; hashing is cumulative, so any change at/before a breakpoint invalidates it and everything after; matching requires a 100%-identical prefix. Minimum cacheable length is model-dependent (512–4096 tokens); shorter prompts silently skip caching. [caching docs]

---

## 2. Tool use (Go)

From `examples/tools/main.go`:

- **Define** a tool as `anthropic.ToolParam{ Name, Description: anthropic.String(...), InputSchema: anthropic.ToolInputSchemaParam{ Properties: map[string]any{...} } }`. The input schema is JSON-Schema-shaped (`Properties` map with `type`/`description`/`enum`); the example hand-writes it (no codegen helper used there).
- **Pass** tools by wrapping each as `anthropic.ToolUnionParam{OfTool: &tool}` and putting the `[]anthropic.ToolUnionParam` into `MessageNewParams.Tools` (alongside `Model`, `MaxTokens`, `Messages`).
- **Read tool calls** by iterating `message.Content` and switching `block.AsAny().(type)` with `case anthropic.ToolUseBlock:`. Use `block.Name` (which tool), `block.ID` (correlate the result), and the raw input JSON (`block.JSON.Input.Raw()` → `json.Unmarshal` into a per-tool struct). (The SDK also exposes typed accessors such as `block.AsToolUse()`.)
- **Feed results back**: append the assistant turn with `message.ToParam()`, then build each result with `anthropic.NewToolResultBlock(block.ID, resultString, isError)` (third arg is the error flag), collect into `[]anthropic.ContentBlockParamUnion`, and send one user message via `anthropic.NewUserMessage(results...)`. Loop until a response contains no `tool_use` blocks.

(The SDK also ships higher-level helpers — `tool-runner`, `mcp-tool-runner`, structured outputs — but the manual loop above is the portable contract.)

## 2b. Streaming (Go)

From `examples/message-streaming/main.go`:

```go
stream := client.Messages.NewStreaming(ctx, params) // returns a streaming handle
for stream.Next() {                 // false when the stream ends
    event := stream.Current()
    switch e := event.AsAny().(type) {
    case anthropic.MessageDeltaEvent:        // e.Delta.StopSequence, and final usage
    case anthropic.ContentBlockDeltaEvent:
        switch d := e.Delta.AsAny().(type) {
        case anthropic.TextDelta:            // d.Text — incremental chunk
        }
    }
}
if stream.Err() != nil { /* errors surface here, after the loop */ }
```

Event variants include `MessageStartEvent`, `ContentBlockStartEvent`, `ContentBlockDeltaEvent` (nested deltas: `TextDelta`, `InputJSONDelta`, `ThinkingDelta`, `SignatureDelta`), `ContentBlockStopEvent`, `MessageDeltaEvent`, `MessageStopEvent`. To reconstruct the full message while streaming, declare `var msg anthropic.Message` and call `msg.Accumulate(event)` on each event in the loop. Final `usage` (as `MessageDeltaUsage`) arrives on the terminal `message_delta` event.

---

## 3. Response `Usage` fields (exact Go names)

Top-level `anthropic.Usage` (`message.go` line 16446). All token counts are `int64`:

| Go field | JSON | Meaning |
|---|---|---|
| `CacheCreationInputTokens` | `cache_creation_input_tokens` | Input tokens written to cache by this request (billed cache-write rate). |
| `CacheReadInputTokens` | `cache_read_input_tokens` | Input tokens read from cache (billed cache-read rate). |
| `InputTokens` | `input_tokens` | Non-cached input tokens (content after the last cache breakpoint). |
| `OutputTokens` | `output_tokens` | Generated output tokens; the inclusive, authoritative billing total. |
| `CacheCreation` | `cache_creation` | Struct `CacheCreation{ Ephemeral5mInputTokens, Ephemeral1hInputTokens int64 }` — write tokens split by TTL; sums to `CacheCreationInputTokens`. |
| `OutputTokensDetails` | `output_tokens_details` | Struct with `ThinkingTokens int64` (`thinking_tokens`): output tokens spent on internal reasoning, a read-only subset of `OutputTokens`. |
| `ServerToolUse` | `server_tool_use` | `ServerToolUsage` (server-side tool request counts, e.g. web search). |
| `InferenceGeo` | `inference_geo` | Region where inference ran. |

Total input processed = `CacheReadInputTokens + CacheCreationInputTokens + InputTokens`. If both cache counts are 0, nothing was cached (commonly: below min length). The streaming variant `MessageDeltaUsage` carries the same core four counters plus `OutputTokensDetails`/`ServerToolUse`, but **not** the `CacheCreation` TTL breakdown. [message.go 16446; `CacheCreation` 3949; `OutputTokensDetails` 10898]

---

## 4. Extended thinking (LEGACY — read before designing)

**Status.** Manual extended thinking (`thinking.type: "enabled"` + `budget_tokens`) is **deprecated**. Models 4.7+ (e.g. Opus 5.5, Sonnet 5/5.5, Haiku 5.5) **reject it with 400**. The replacement is **adaptive thinking**: `thinking: {type: "adaptive"}` with reasoning depth controlled by `output_config.effort` rather than a token budget. [extended-thinking docs]

**Enable (manual, Go).**
```go
MessageNewParams{ Thinking: anthropic.ThinkingConfigParamOfEnabled(10000), ... }
```
`ThinkingConfigParamUnion` has `OfEnabled` (`ThinkingConfigEnabledParam{ BudgetTokens int64 }`) and `OfAdaptive` (`ThinkingConfigAdaptiveParam{ Display: ThinkingConfigAdaptiveDisplay }`, values `summarized`/`omitted`). `budget_tokens` must be ≥1024 and `< max_tokens` (exception: interleaved thinking). [message.go 13237/13281/13160/13290]

**Returned.** A `anthropic.ThinkingBlock` content block with `.Thinking` (summarized reasoning text) and `.Signature` (opaque verification token). When using tools with thinking, thinking blocks **must be passed back verbatim with `Signature` intact**, and in manual mode the final assistant turn must start with a thinking block. Streaming emits `ThinkingDelta`/`SignatureDelta`. Cost is observable via `usage.output_tokens_details.thinking_tokens` (final `message_delta` only when streaming). [message.go 13105; extended-thinking docs]

---

## Design implications for a provider-neutral contract

- **Caching**: model a breakpoint as "mark this block as a cache boundary" plus an optional TTL enum; cap at 4 and preserve `tools→system→messages` ordering. Anthropic requires an exact-prefix match, so the contract must keep block order/content stable across turns.
- **Usage**: expose at least `inputTokens`, `outputTokens`, `cacheReadTokens`, `cacheWriteTokens`, `reasoningTokens`; note total-input = read + write + input. The TTL split and `server_tool_use` are Anthropic-specific extras.
- **Tool loop**: the portable shape is id-correlated `tool_use` → `tool_result(isError)` with an echo of the assistant turn; keep an `isError` flag.
- **Thinking**: treat it as two modes — legacy budgeted (`enabled`) vs. adaptive (`effort`). Prefer an abstract "reasoning effort" knob over a raw token budget, since budgeted mode is 400 on current models. Carry an opaque, pass-through `signature` for reasoning blocks (required for tool use).
