# Glossary

Domain vocabulary for APEX SWARM. Implementation-free — concepts, not structs.

## Target
The single authorized seed (a domain or IP) a run starts from. Must lie within Scope. The Target is an *input*; it is not an Asset.

## Scope
The authorized boundary of a run: an allow-list and a deny-list of matchers (domain, IP, or CIDR). A domain matcher implicitly covers its subdomains. Something is *in scope* when it matches an allow entry and no deny entry. Scope is enforced in two places — the tool layer and the executor. Scope governs only what a run directs at the Target's side; the run's own infrastructure (such as the OOB callback host) is not subject to it. Human approval can never override Scope.

## Asset
Something discovered during recon (a subdomain, host, or live web service) that falls within scope. Distinct from the Target, which is the seed that led to it.

## Finding
A potential issue reported by a scanner (nuclei) against an Asset. A Finding is *suspected* until verified.

## Verified finding
A Finding confirmed by an out-of-band (OOB) interaction: a scanner payload triggered a callback to our interactsh and was correlated back to the Finding. Only verified findings count as hard evidence.

## Evidence
The persisted record proving a Verified finding: the correlated OOB interaction (protocol, source IP, timestamp, raw request, correlation id).

## Run
One end-to-end execution against one Target, under one Scope and one Budget: recon → scan → verify → persist.

## Budget
The hard guardrails on a Run's cost: a token ceiling (primary) and a wall-clock timeout (secondary). Hitting either stops the run.

## Risk tier
The classification of a single tool invocation by its potential impact on what it touches. *Passive*: sends nothing to the Target's own hosts. *Active low-risk*: sends benign probes to in-scope hosts. *High-risk*: could change state or cause harm. The tier belongs to the invocation, not the tool — the same tool can land in different tiers depending on what it is asked to do. A high-risk invocation never runs without human approval.
