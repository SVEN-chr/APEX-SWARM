---
status: accepted
---

# Claude is called through the native Anthropic SDK, behind a provider-neutral contract

The agent loop talks to a small provider-neutral contract (`Provider.Complete`), and the only Phase 1 implementation, AnthropicProvider, calls Claude through the official `anthropic-sdk-go`. Claude must never be routed through an OpenAI-compatible endpoint or shim, even after an OpenAI-compatible provider is added for other models. The features the loop depends on exist only on the native API: explicit and automatic prompt-cache breakpoints, signed reasoning blocks that must be replayed verbatim, effort-based reasoning control, refusal stop reasons with a category, and separate cache-read and cache-write usage counts.

## Considered options

- **One OpenAI-compatible client for every model.** Rejected: it drops cache control, reasoning-block replay, and refusal details, which silently raises cost and breaks tool use with reasoning.
- **Call the Anthropic SDK directly from the loop, with no contract.** Rejected: Phase 2 adds cost-based routing across providers, and the loop would have to be rewritten around SDK types.
- **Neutral contract plus native SDK per provider.** Chosen.

## Consequences

- The contract carries concepts an OpenAI-style API lacks: an opaque `Reasoning` block, a `Refusal` stop reason, and cache-read and cache-write token counts. Other providers leave them empty.
- Conversation history is append-only. Rewriting it invalidates the reasoning blocks and the prompt cache.
- Model and reasoning effort are fixed for a whole Run, because the prompt cache is scoped to both.
- Each new provider is a new implementation of the contract, not a base-URL change.

Decided in [T2 · LLM 契约 + AnthropicProvider + 缓存断点](https://github.com/SVEN-chr/APEX-SWARM/issues/3).
