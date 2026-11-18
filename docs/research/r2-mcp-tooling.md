# R2 — MCP server availability for the Phase-1 ProjectDiscovery toolchain

**Ticket:** R2 (wayfinder research) · **Date:** 2026-11-18 · **Scope:** subfinder, dnsx, httpx, nuclei, interactsh
**Status:** public research on open-source tooling; no exploitation, no scanning performed.

## TL;DR

- **There is no official ProjectDiscovery MCP server.** The official MCP registry returns 0 results for "projectdiscovery", and the `projectdiscovery` GitHub org publishes no MCP repo. PD's AI story today is the ProjectDiscovery Cloud Platform and Nuclei AI templates, not an MCP integration.
- **Community coverage is partial and immature.** One community server (`intelligent-ears/pd-tools-mcp`) wraps most of the recon chain (subfinder, dnsx, httpx, nuclei + naabu, katana) but has **no license** and is effectively a one-shot, abandoned repo — not reusable. interactsh is **not** covered by it.
- **interactsh is the one tool with a clean, reusable option:** `tachote/mcp-interactsh` (MIT, Node).
- **Net recommendation: BUILD thin CLI wrappers** for subfinder/dnsx/httpx/nuclei behind a shared scaffold, and **BUY (reuse)** `tachote/mcp-interactsh` for interactsh (or at least model the interactsh server on it).

---

## Per-tool findings

### subfinder, dnsx, httpx, nuclei — only `pd-tools-mcp`, not reusable

The only server that exposes these as MCP tools is the community project **`intelligent-ears/pd-tools-mcp`** ("ProjectDiscovery MCP Server").

**Repo facts (GitHub API, fetched 2026-11-18):**

| Field | Value |
|---|---|
| Repo | `intelligent-ears/pd-tools-mcp` |
| Language | TypeScript (Node, stdio transport) |
| **License** | **none (null)** → all-rights-reserved, legally not reusable as-is |
| Stars | 8 |
| Created / last push | 2025-12-05 / 2025-12-06 (≈4 commits, no activity since) |

**Exposed tools** (from the README / glama schema): `subfinder`, `dnsx`, `naabu`, `httpx`, `katana`, `nuclei`, and a combined `bug_bounty_workflow`. **No interactsh tool.**

**Input schemas (as documented — outputs are *not* documented in the repo):**

| Tool | Inputs | How target/scope is passed |
|---|---|---|
| `subfinder` | `domain` (string, required), `silent` (bool, opt) | single `domain` arg |
| `dnsx` | `domains` (array, required), `recordType` (opt: A/AAAA/CNAME/…) | array of `domains` |
| `httpx` | `urls` (array, required), `followRedirects` (bool), `screenshot` (bool) | array of `urls` |
| `nuclei` | `targets` (array, required), `templates` (array, opt), `severity` (array, opt: critical…info) | array of `targets` |
| `bug_bounty_workflow` | `domain` (required) + toggles (portScan/crawl/vulnScan), `severityFilter`, rate-limit settings | single `domain`; returns a JSON report |

**Prerequisite:** the actual ProjectDiscovery CLIs must be installed via Go and present on `$PATH`; the server just shells out to them.

**Verdict for these four:** not buyable. No license is a hard blocker for a project repo, the output schema is undocumented, and the repo is unmaintained. The chained `bug_bounty_workflow` is also the wrong granularity for an orchestrated swarm that wants to drive each stage itself.

#### nuclei — additional nuclei-only candidates (still thin wrappers)

- `gc-nuclei-mcp` (npm package) — nuclei only; binary located via `NUCLEI_PATH`.
- `crazymarky/mcp_nuclei_server` — Python; binary via `NUCLEI_BIN_PATH`.
- `FuzzingLabs/mcp-security-hub` — MIT, Python, Dockerized collection of offensive tools incl. nuclei (the most license-clean option for nuclei).
- `addcontent/nuclei-mcp` — Go (mark3labs/mcp-go); install docs still contain a placeholder path, treat as unverified.

None is an authoritative/first-party server; all are thin shells over the `nuclei` binary with varying quality and maintenance.

### interactsh — `tachote/mcp-interactsh` is a clean reuse candidate

**Repo facts (GitHub API, 2026-11-18):** `tachote/mcp-interactsh`, JavaScript/Node (≥18), **MIT license**, 3 stars, last push 2025-10-15. Install: `npx -y mcp-interactsh` or clone + `npm install`.

**Exposed tools (4):**

| Tool | Input | Output / behavior |
|---|---|---|
| `create_interactsh_session` | — | returns correlation ID, secret key, PEM private key, **callback domain**, server URL, probe instructions |
| `list_interactsh_sessions` | — | sessions cached in the current MCP process |
| `poll_interactsh_session` | optional filters: `method`, `path_contains`, `query_contains`, `protocol`, `text_contains` (+ a session id) | new interactions, decrypted locally |
| `deregister_interactsh_session` | (session id) | tears the session down |

**How scope/target is passed:** the session returns a `callback domain`; a probe host is `<correlation_id(20 hex)><nonce(13)>.<domain>` (first label must be 33 chars). You inject that host into the target (e.g. via a nuclei/httpx payload), then `poll_interactsh_session`.

**Config:** `INTERACTSH_BASE_URL` (default `https://oast.pro`), `INTERACTSH_DOMAIN_SUFFIX`, optional `INTERACTSH_TOKEN`. Point `INTERACTSH_BASE_URL` at a self-hosted interactsh server for a private OAST (recommended for a controlled swarm rather than the public `oast.pro`).

Alternative: `x0base/mcp-security-toolkit` exposes `interactsh_register` / `interactsh_poll` as part of a broader toolkit, if a single multi-tool server is preferred.

---

## Buy vs build recommendation

| Tool | Ready MCP server today? | Recommendation | Rough build effort (thin CLI wrapper) |
|---|---|---|---|
| **subfinder** | Only via unlicensed/abandoned `pd-tools-mcp` | **BUILD** | ~0.5 day (single `domain` in, JSON lines out) |
| **dnsx** | same | **BUILD** | ~0.5 day |
| **httpx** | same | **BUILD** | ~0.5–1 day (more flags/output fields) |
| **nuclei** | Several thin community wrappers (gc-nuclei-mcp, FuzzingLabs, crazymarky) | **BUILD** (optionally vendor FuzzingLabs' as a reference) | ~1–1.5 days (template/severity/rate-limit flags, structured `-jsonl` parsing) |
| **interactsh** | **Yes — `tachote/mcp-interactsh` (MIT)** | **BUY / reuse** (point at self-hosted OAST) | ~1–2 days only if you must build it in-house |

**Rationale for BUILD on the recon four:** every existing server is a thin `exec()` over the same CLIs we already ship; building them ourselves behind one shared scaffold (arg-building + `-jsonl`/`-json` stdout parsing + a common result schema) is ~2–3 days total, gives us a documented output contract, per-stage control the chained `bug_bounty_workflow` doesn't offer, a license we own, and no dependency on an abandoned 8-star repo. Each PD CLI already emits structured JSON (`-json`/`-jsonl`), so wrapping is mechanical.

**Rationale for BUY on interactsh:** the session/poll/decrypt lifecycle is non-trivial crypto+protocol glue, `tachote/mcp-interactsh` already implements it cleanly under MIT, and it supports a self-hosted server via env var. Reuse it (vendor/pin the version) rather than reimplement.

---

## Sources (primary-first)

- Official MCP registry query (empty for projectdiscovery): https://registry.modelcontextprotocol.io/v0/servers?search=projectdiscovery
- ProjectDiscovery interactsh docs: https://docs.projectdiscovery.io/tools/interactsh — repo: https://github.com/projectdiscovery/interactsh
- `intelligent-ears/pd-tools-mcp`: https://github.com/intelligent-ears/pd-tools-mcp — schema listing: https://glama.ai/mcp/servers/intelligent-ears/pd-tools-mcp/schema (repo metadata via GitHub API `repos/intelligent-ears/pd-tools-mcp`, 2026-11-18)
- `tachote/mcp-interactsh`: https://github.com/tachote/mcp-interactsh — README: https://cdn.jsdelivr.net/gh/tachote/mcp-interactsh@main/README.md
- `x0base/mcp-security-toolkit` (interactsh_register/poll): https://glama.ai/mcp/servers/x0base/mcp-security-toolkit
- Nuclei wrappers: `gc-nuclei-mcp` https://unpkg.com/gc-nuclei-mcp@1.1.5/readme.md · `crazymarky/mcp_nuclei_server` https://lobehub.com/en/mcp/crazymarky-mcp_nuclei_server · `FuzzingLabs/mcp-security-hub` https://github.com/fuzzinglabs/mcp-security-hub · `addcontent/nuclei-mcp` https://cdn.jsdelivr.net/gh/addcontent/nuclei-mcp@main/README.md
