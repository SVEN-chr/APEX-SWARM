---
status: accepted
---

# The executor's tool registry is the only source of tool definitions; MCP is used as transport, not discovery

Every tool the agent can call is registered on the executor side with a descriptor: the tool definition shown to the model, its base Risk tier, its timeout, which input fields are directed at the Target, and which result fields may be returned. The model is shown the definitions from this registry, never the ones an MCP server advertises. MCP carries the calls and nothing else. We chose this because an MCP server cannot be trusted to classify its own risk or to keep its advertised definitions stable, and because Scope enforcement in the executor needs to know which inputs point at the Target without trusting the tool.

## Considered options

- **Expose whatever each MCP server advertises.** Rejected: a third-party server (interactsh is one) declares no Risk tier, can change its tool descriptions between versions, and can add tools we never reviewed. Advertised definitions that change would also break the prompt cache, which requires the tool list to be identical across calls.
- **Take definitions from the server and attach our metadata by tool name.** Rejected: the model would still read text we do not control, and a tool with no metadata needs a default tier, which is a guess.
- **Registry as the single source, checked against the servers at startup.** Chosen.

## Consequences

- Fail-closed in both directions. A registered tool that its server does not provide stops startup. A tool a server provides that is not registered is never exposed.
- Adding a tool always means writing a descriptor and choosing its Risk tier. Pointing the executor at a new MCP server adds nothing on its own.
- Upgrading a third-party server can break startup when its tools change. Its version is pinned, and each upgrade is a reviewed change.
- The executor can enforce Scope without trusting the tool, because the descriptor says where the Target-directed inputs are. This is what makes the two Scope checks independent.
- Results from third-party tools pass through the descriptor's field allow-list, so secrets a server returns (interactsh session keys) never reach the model or the step log.

Decided in [T3 · MCP 工具适配契约 + 风险分级 + scope 两处强制](https://github.com/SVEN-chr/APEX-SWARM/issues/4).
