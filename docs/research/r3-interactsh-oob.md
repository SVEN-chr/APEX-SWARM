# R3 — interactsh out-of-band (OOB) correlation model

**Ticket:** R3 (wayfinder research) · **Purpose:** inform evidence-verification design for blind/OOB vulnerabilities.
**Scope:** How interactsh issues payloads, does its server↔client crypto, how clients poll, how nuclei ties a callback back to a template/request, and what an interaction record contains as persisted evidence.
**Sources:** interactsh + nuclei source code and ProjectDiscovery docs only (cited inline). Line-level facts were read from `main` (interactsh) and `dev` (nuclei) branches on 2026-11-18.

> Convention note: the repo had no existing `docs/research/` folder. Created one here to match the ticket's suggested path and the repo's `docs/<area>/` layout (cf. `docs/adr/`, `docs/agents/`).

---

## 1. Payload / correlation-ID issuance

An interactsh **payload** is a subdomain of the server domain, composed of two concatenated random parts:

```
<correlationID><nonce>.<server-host>
e.g.  c23b2la0kl1krjcrdj10cndmnioyyyyyn.oast.pro
```

- **Correlation ID** — default **20 chars** (`-cidl` / `-correlation-id-length`, min 3). It is the stable key that identifies a client/session; it is the prefix the server indexes registrations and interactions under.
- **Nonce** — default **13 chars** (`-cidn` / `-correlation-id-nonce-length`, min 3), appended per generated URL so each payload is unique even for one client. Default payload length is therefore **33 chars**.
- Server and client **must** use the same lengths or correlation fails ("It is important and required to use same length on both side").

Source: interactsh README (flags `-cidl`, `-cidn`; payload example) — https://github.com/projectdiscovery/interactsh/blob/main/README.md

### How the strings are generated (client source)
- Correlation ID (anonymous session): `newAnonymousCorrelationID()` uses `xid.NewWithTime(time.Now())` (keeps the 4-byte timestamp prefix) then **overwrites the trailing 8 bytes with `crypto/rand`**, and truncates to `CorrelationIdLength`. Comment: the overwrite avoids a "stable per-machine fingerprint" that a plain `xid` would leak.
- **Secret key**: `uuid.New().String()` (google/uuid) — per-client shared secret, never part of the hostname.
- **Per-URL nonce**: `URL()` returns `correlationID + randomData + "." + serverURL.Host`, where `randomData` is `CorrelationIdNonceLength` bytes of `crypto/rand` encoded with `zbase32.StdEncoding`.

Source: `pkg/client/client.go` — https://github.com/projectdiscovery/interactsh/blob/main/pkg/client/client.go

---

## 2. Server↔client registration & crypto (high level)

interactsh uses a **per-registration hybrid scheme**: RSA to bootstrap, AES-256-CTR to carry interaction data. Zero-logging/"AES encryption" claim in the docs resolves to the following concrete flow.

**Client side** (`pkg/client/client.go`):
1. `initializeRSAKeys()` → `rsa.GenerateKey(rand.Reader, 2048)` (RSA-2048 keypair, held only in client memory).
2. `encodePublicKey()` → `x509.MarshalPKIXPublicKey` → PEM block (`"RSA PUBLIC KEY"`) → base64 (`StdEncoding`).
3. **Register** `POST /register` with JSON body `RegisterRequest{ public-key, secret-key, correlation-id }`. (HTTP has an HTTP-fallback registration path; `-nf` disables it. README.)

**Server side** (`pkg/server/http_server.go` → `pkg/storage/storagedb.go`):
1. `registerHandler` decodes `RegisterRequest` (`public-key` / `secret-key` / `correlation-id`), rejects bad correlation ID or empty secret with **400**, then `Storage.SetIDPublicKey(...)`. Returns `{message:"registration successful", capabilities:...}`.
2. `SetIDPublicKey`:
   - Parses the key via `ParseB64RSAPublicKeyFromPEM` (base64-decode → PEM → PKIX public key).
   - Rejects a correlation ID already present in cache (no re-registration/overwrite).
   - **Generates a 32-byte AES key** once per registration: `aesKey := make([]byte,32); rand.Read(...)` → **AES-256**.
   - Wraps it: `rsa.EncryptOAEP(sha256.New(), rand.Reader, publicKey, aesKey, []byte(""))` (RSA-OAEP/SHA-256, empty label), base64-encoded into `AESKeyEncrypted`.
   - Stores `CorrelationData{ SecretKey, AESKey (raw 32 bytes), AESKeyEncrypted }` in a cache keyed by correlation ID, with eviction TTL (`-eviction`, default **30 days**; `ExpireAfterWrite` for fixed strategy else `ExpireAfterAccess`). **The RSA public key itself is not retained** — only the derived AES material.

So: the AES session key is minted by the **server**, encrypted to the client's public key, and the client later recovers it with its private key. The client's private key never leaves the client; the server never stores a reusable copy of the public key.

Sources:
- `pkg/server/http_server.go` (register/deregister/poll handlers) — https://github.com/projectdiscovery/interactsh/blob/main/pkg/server/http_server.go
- `pkg/storage/storagedb.go` (`SetIDPublicKey`, `GetInteractions`, AES key gen + RSA-OAEP wrap) — https://github.com/projectdiscovery/interactsh/blob/main/pkg/storage/storagedb.go
- `pkg/storage/util.go` (`AESEncrypt`, `ParseB64RSAPublicKeyFromPEM`) — https://github.com/projectdiscovery/interactsh/blob/main/pkg/storage/util.go

### Authentication caveat (relevant to evidence integrity)
`GetInteractions(correlationID, secret)` authorizes a poll by comparing the supplied secret to the stored one with **`strings.EqualFold` (case-insensitive, non-constant-time)** — **not** `subtle.ConstantTimeCompare`. Empty secret on either side is rejected. Worth noting if this correlation secret is ever treated as an authentication boundary for evidence. (`pkg/storage/storagedb.go`.)

---

## 3. How the client polls / retrieves interactions

- **Transport**: `getInteractions` sends `GET {server}/poll?id=<correlationID>&secret=<secretKey>` (note: query param is **`secret`**, not `secret-key`). If a server token is configured it is sent in the `Authorization` header. Poll cadence default **5s** (`-pi` / `-poll-interval`); keep-alive default **1m** (`-kai`); `-sf` session file lets a client resume. (README + `pkg/client/client.go`.)
- **Server response** `PollResponse` JSON tags:
  - `data` — array of base64 strings, each an AES-encrypted `Interaction` JSON.
  - `aes_key` — base64 of the RSA-OAEP-wrapped AES key (`AESKeyEncrypted`). *(Tag is `aes_key` with an underscore.)*
  - `extra` (`extra`) and `tlddata` (`tlddata,omitempty`) — extra/root-TLD interactions; **server comments note these are NOT encrypted.**
- **Encryption of each item** (`AESEncrypt(key, message)` in `util.go`): `aes.NewCipher(key)` + `cipher.NewCTR(block, iv)` with a **random IV prepended** to the ciphertext, whole thing base64 (`StdEncoding`). (Mode is **CTR**, despite a stray "CBC"-style comment elsewhere.)
- **Client decryption** (`decryptMessage`): base64-decode `aes_key` → `rsa.DecryptOAEP(sha256.New(), …, privKey, …)` to recover the AES key → for each `data` item: base64-decode → first `aes.BlockSize` bytes = IV → `cipher.NewCTR` decrypt → trim trailing `" \t\r\n"` → `json.Unmarshal` into `server.Interaction`.

Source: `pkg/client/client.go`, `pkg/storage/util.go`, `pkg/server/http_server.go` (links above).

---

## 4. How nuclei ties a callback to a template/request

Template author surface (docs): write `{{interactsh-url}}` in an **http** or **network** request; it is "replaced on runtime with unique interactsh URLs" and "Nuclei will handle correlation of the interaction to the template & the request it was generated from." Matcher/extractor `part` values:
- `interactsh_protocol` — `dns`, `http`, or `smtp` (standard matcher; DNS common, non-intrusive).
- `interactsh_request` — the request the interactsh server received.
- `interactsh_response` — the response the server sent.

Source: ProjectDiscovery "OOB Testing" — https://docs.projectdiscovery.io/templates/reference/oob-testing

Engine mechanics (`pkg/protocols/common/interactsh/interactsh.go`, nuclei `dev`):
1. **Mint & remember URL**: `NewURLWithData(data)` → `c.URL()` and stores the generated URL as a key in the `interactshURLs` cache (`SetWithExpire(url, marker, defaultInteractionDuration)`). `Replace`/`ReplaceWithMarker` swap `{{interactsh-url}}` markers in the template text for freshly minted URLs.
2. **Register the request against the payload**: when the HTTP/network request is actually sent, `RequestEvent(interactshURLs, data)` derives a **lookup key by stripping the server-host suffix (and trailing dot) off each URL** — i.e. the `<correlationID><nonce>` label — and stores the `*RequestData` in cache `c.requests` keyed by that id (`SetWithExpire(id, data, c.eviction)`). It also `data.Scope.track(id)`.
3. **Match a callback to its request**: the `poll()` callback looks up `c.requests.Get(interaction.UniqueID)` — **the cache is keyed by the interaction's `UniqueID`**, so `UniqueID` is the join column between a received callback and the originating request. (`FullId` appears only in debug output here.)
4. **Race handling**: a callback that arrives before its request is registered is buffered in `c.interactions` for `defaultInteractionDuration`; a later `RequestEvent` with the same key drains it.
5. **Produce a traceable finding**: `processInteractionForRequest` writes `interactsh_protocol / interactsh_request / interactsh_response / interactsh_ip` into `InternalEvent`, runs operators, and on a match sets **`event.Interaction = interaction`** (the full record rides along on the result) and `event.InteractshMatched`. Traceability rests on three linked things: the `Interaction` attached to the result event, the `RequestData` stored under the same id, and the `templateId:host` hash (`eventHash`) used for stop-at-first-match.

Source: `pkg/protocols/common/interactsh/interactsh.go` — https://github.com/projectdiscovery/nuclei/blob/dev/pkg/protocols/common/interactsh/interactsh.go

**Caveat for evidence design:** correlation is keyed on `UniqueID` (the correlation-ID label), held in an **in-memory cache with a TTL** (`defaultInteractionDuration` / `c.eviction`). A callback that arrives after eviction, or after the scan process exits, is no longer correlatable by nuclei — persistence must happen at capture time, not after. (Known edge: nuclei issues #1361 / #1844 on stop-at-first-match and correlation misses.)

---

## 5. Interaction record — fields available as persisted evidence

The authoritative schema is `server.Interaction` (`pkg/server/server.go`). Every decrypted callback unmarshals into this:

| Field | JSON tag | Evidence value |
|---|---|---|
| Protocol | `protocol` | `dns` / `http` / `https` / `smtp` / `smtps` / `ldap` / `ftp` — what channel fired |
| UniqueID | `unique-id` | correlation-ID label of the payload that was hit → **the join key** to the scan/request |
| FullId | `full-id` | full subdomain/path form of the payload that received the interaction |
| QType | `q-type,omitempty` | DNS query type (A/AAAA/MX/TXT/…) |
| RawRequest | `raw-request,omitempty` | **raw inbound request** the server received (full DNS/HTTP/SMTP payload) |
| RawResponse | `raw-response,omitempty` | raw response the server returned |
| SMTPFrom | `smtp-from,omitempty` | SMTP MAIL FROM, when protocol is SMTP |
| RemoteAddress | `remote-address` | **source IP** of the calling system (the victim/target reaching out) |
| Timestamp | `timestamp` | `time.Time` of the interaction |
| AsnInfo | `asninfo,omitempty` | ASN enrichment for the remote IP (populated with client `-asn`) |

Source: `pkg/server/server.go` (Interaction struct) — https://github.com/projectdiscovery/interactsh/blob/main/pkg/server/server.go

Client output: `-json` emits JSON Lines; `-v` prints full request/response; hosted-file fetches replace the body with a byte-count + sha256 placeholder (README).

---

## 6. Takeaways for evidence verification (blind vulns)

1. **Minimum persisted evidence tuple** per confirmed OOB hit: `unique-id` + `protocol` + `remote-address` + `timestamp` + `raw-request` (+ `q-type`/`smtp-from`/`asninfo` where present). This is enough to prove *a target-controlled callback of type X arrived at a payload uniquely minted for request Y at time T from IP Z*.
2. **The join key to the scan is `unique-id`** (interactsh correlation ID). Store it alongside nuclei's `templateId:host` so a finding is traceable both to the payload and to the template/request that planted it.
3. **Capture at poll time, not later** — correlation state (nuclei `c.requests`, server interaction cache) is TTL-bounded in memory; persist the `Interaction` the moment it's matched.
4. **Trust boundaries to document**: poll auth uses a non-constant-time, case-insensitive secret compare; `extra`/`tlddata` poll fields are transmitted unencrypted; the correlation ID carries a 4-byte timestamp prefix (xid) by design. None breaks evidence collection but all belong in a threat/assurance note for the verification layer.

---

### Primary sources
- interactsh README — https://github.com/projectdiscovery/interactsh/blob/main/README.md
- `pkg/client/client.go` — https://github.com/projectdiscovery/interactsh/blob/main/pkg/client/client.go
- `pkg/server/server.go` — https://github.com/projectdiscovery/interactsh/blob/main/pkg/server/server.go
- `pkg/server/http_server.go` — https://github.com/projectdiscovery/interactsh/blob/main/pkg/server/http_server.go
- `pkg/storage/storagedb.go` — https://github.com/projectdiscovery/interactsh/blob/main/pkg/storage/storagedb.go
- `pkg/storage/util.go` — https://github.com/projectdiscovery/interactsh/blob/main/pkg/storage/util.go
- nuclei `pkg/protocols/common/interactsh/interactsh.go` — https://github.com/projectdiscovery/nuclei/blob/dev/pkg/protocols/common/interactsh/interactsh.go
- ProjectDiscovery OOB Testing (nuclei) — https://docs.projectdiscovery.io/templates/reference/oob-testing
- interactsh overview — https://docs.projectdiscovery.io/tools/interactsh/overview
