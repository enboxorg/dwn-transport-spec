
DWN Transport Protocol
==================

**Specification Status:** Draft

**Latest Draft:**
  https://dwn-transport-spec.pages.dev
<!-- -->

**Editors:**
~ [Liran Cohen](https://www.linkedin.com/in/itsliran/) (Enbox)

**Participate:**
~ [GitHub repo](https://github.com/enboxorg/dwn-transport-spec)
~ [File a bug](https://github.com/enboxorg/dwn-transport-spec/issues)
~ [Commit history](https://github.com/enboxorg/dwn-transport-spec/commits/main)

------------------------------------

## Abstract

The [Decentralized Web Node (DWN) specification](https://dwn-spec.pages.dev) defines core message interfaces, module definitions, authorization, and encryption for personal data storage nodes — while leaving network transport, synchronization, and wallet connection protocols to companion specifications.

This specification defines the network and loopback transport layer that clients, servers, wallets, and agents ****MUST**** implement to interoperate when communicating with [[ref:Decentralized Web Nodes]].

Specifically, this specification covers:

- **Service endpoint discovery** from DID documents, plus optional server-side forwarding and `$delivery` routing
- **JSON-RPC 2.0 wire format** for `dwn.processMessage`, `dwn.applyReplicatedMessage`, subscriptions, flow control, and application-level keepalive (`rpc.ping`)
- **HTTP transport** using single-body binary framing (`application/vnd.enbox.dwn-rpc`)
- **WebSocket transport** for real-time subscriptions, sliding-window backpressure, pooled RPC routing, and dual-layer liveness
- **REST convenience API** for simple, unauthenticated resource access
- **Server information and capability discovery** (`/info`, `/health`)
- **Tenant registration** (Proof-of-Work, Terms of Service, and `provider-auth-v0`)
- **Sync transport binding** over HTTP/WebSocket for the companion [DWN Sync Protocol](https://dwn-sync-spec.pages.dev)
- **Local Transport** (opt-in Local Node Profile, loopback pairing, origin-bound tokens, and passive discovery)

## Status of This Document

DWN Transport Protocol is a _DRAFT_ specification under development by
[Enbox](https://enbox.org). It is a companion specification to the
[DWN specification](https://github.com/enboxorg/dwn-spec) and describes the
transport layer implemented in [enboxorg/enbox](https://github.com/enboxorg/enbox)
(`@enbox/dwn-server`, `@enbox/dwn-clients`, `@enbox/local-node`, `@enbox/agent`,
`@enbox/auth`, and `@enbox/dids`).

Earlier draft sections covering per-message payment (`-50402`, `dwn-payment`,
L402/Cashu/x402), Sparse Merkle Tree `MessagesSync`, DID challenge-response
local auth, Bonjour/mDNS, iOS activation, Android AIDL, and browser extension
bridges are not part of the current implementation and have been removed from
this draft.

The specification will be updated to incorporate feedback. We encourage reviewers
to submit [GitHub Issues](https://github.com/enboxorg/dwn-transport-spec/issues)
as the means by which to communicate feedback and contributions.

## Terminology

[[def:Decentralized Web Node, Decentralized Web Nodes, DWN, Node]]
~ A decentralized personal and application data storage and message relay node,
as defined in the [DWN specification](https://github.com/enboxorg/dwn-spec).

[[def:DWN Server, DWN Servers]]
~ A network-accessible or loopback-bound service that hosts one or more
[[ref:Decentralized Web Node]] instances and exposes them via the HTTP and
WebSocket transports defined in this specification. A single DWN Server may
serve multiple tenants.

[[def:DWN Client, DWN Clients]]
~ Any software that sends DWN messages to a [[ref:DWN Server]] over the
transports defined in this specification. Clients include user agents, browser
applications, CLI tools, other DWN Servers performing forwarding/delivery, and
sync agents.

[[def:Tenant]]
~ The DID that owns a [[ref:Decentralized Web Node]] instance. All messages sent
to a DWN Server are scoped to a target tenant.

[[def:Service Endpoint, Service Endpoints]]
~ A URL published in a DID document that identifies the network location of a
[[ref:DWN Server]] hosting a [[ref:Decentralized Web Node]] for that DID.

[[def:JSON-RPC Envelope]]
~ A JSON object conforming to the
[JSON-RPC 2.0 specification](https://www.jsonrpc.org/specification) that wraps
a DWN message or control request for transport over HTTP or WebSocket.

[[def:Subscription]]
~ A long-lived stream over WebSocket where the server pushes real-time events to
the client in response to a `RecordsSubscribe` or `MessagesSubscribe` message.
Messages are delivered as [[ref:SubscriptionMessage]] objects. See
[Subscription Events](#subscription-events).

[[def:SubscriptionMessage]]
~ A 3-way discriminated union delivered over a [[ref:Subscription]]. Each
message has a `type` field: `"event"` (carrying a progress token, payload, and
optional feed metadata), `"eose"` (End-of-Stored-Events marker completing
catch-up replay), or `"error"` (terminal subscription error closing the stream).

[[def:Progress Token]]
~ A replay position object assigned by the server's durable replication feed /
EventLog with the shape `{ streamId, epoch, position, messageCid? }`.
`position` is a monotonic decimal string within `(streamId, epoch)`.
`messageCid` is present when the token points to a specific delivered feed row,
and is omitted for high-water cursors that produced no matching entry and for
position-zero empty-log anchor tokens (`position: "0"`). Clients ****MUST NOT****
synthesize or transform token fields. Position ordering comparisons
****MUST**** be numeric (BigInt-safe), not lexicographic. Progress tokens are
source-local resume artifacts and ****MUST NOT**** be reused across different
remote providers or different logical streams.

[[def:Stream ID]]
~ A stable identifier for the logical event stream domain issuing progress
tokens (the first 8 bytes of `SHA-256(UTF-8(tenantDid))` as 16-character
lowercase hex). It ****MUST**** remain stable across restarts, horizontal
scaling, and failover, and ****MUST NOT**** include process-specific identifiers.

[[def:Stream Epoch]]
~ A generation identifier for a [[ref:Stream ID]]. Epoch changes when replay
continuity cannot be guaranteed. Tokens from older epochs are invalid. Epoch
values are equality-checked only and ****SHOULD**** be generated as UUID v4
values.

[[def:Local Transport]]
~ A loopback-bound HTTP and WebSocket transport profile (`Local Node Profile`)
for on-device communication between a [[ref:DWN Client]] (browser app, CLI, or
desktop app) and a local [[ref:DWN Server]] process, protected by loopback
binding, `Host` header validation, origin-bound pairing tokens, and file-backed
CLI bearer tokens.

[[def:Discovery File]]
~ A JSON file written by a local [[ref:DWN Server]] process at
`~/.enbox/dwn.json` (with `0o600` permissions) advertising its loopback
`endpoint`, `pid`, `capabilities`, and optional `localNodeToken`.

## Service Endpoint Discovery

### DID Document Service Entry

A [[ref:DWN Client]] discovers the network location of a DID's [[ref:Decentralized Web Node]]
by resolving the DID and inspecting the DID document for a service entry of type
`DecentralizedWebNode`.

- The `type` property ****MUST**** be `"DecentralizedWebNode"`.
- When publishing a service entry, implementations ****MUST**** set `id` to
  `<did>#dwn` (or `#dwn` / `dwn`). Canonical DID endpoint resolution
  (`@enbox/dids`) locates the DWN service exclusively by `id` matching
  `<did>#dwn`, `#dwn`, or `dwn` (returning `service-missing` for any other `id`).
- The `serviceEndpoint` property ****MUST**** be a single URL `string` or a
  non-empty array of URL `string` values. Multiple endpoints indicate nodes that
  synchronize to the same state.

::: example DID Document with DWN Service Entry
```json
{
  "id": "did:example:alice",
  "service": [{
    "id": "did:example:alice#dwn",
    "type": "DecentralizedWebNode",
    "serviceEndpoint": [
      "https://dwn.example.com",
      "https://dwn-backup.example.com"
    ]
  }]
}
```
:::

::: note
Server-side forwarding (`DeliveryService`) matches all services with
`type === "DecentralizedWebNode"` regardless of `id`, and additionally tolerates
endpoint map objects `{ "url": "<endpoint-url>", "dataRetention"?: "full" | "cache" }`
either directly as `serviceEndpoint` or inside `serviceEndpoint` arrays so full
endpoints can be prioritized ahead of cache endpoints during forwarding.
Canonical client resolution (`@enbox/dids`) requires string URLs and reports
non-string endpoint entries as `service-malformed`. Recursive
DID-as-service-endpoint resolution is not implemented.
:::

### Endpoint URL Requirements

- [[ref:Service Endpoint]] URLs ****MUST**** use the `https:` or `http:` scheme
  (production remote endpoints ****SHOULD**** use `https:`).
- URLs ****MUST NOT**** include query parameters (`?`) or fragment identifiers
  (`#`).
- Trailing `/` characters are stripped during normalization, and duplicate
  normalized URLs are deduplicated.
- The normalized URL represents the base path for all transport endpoints
  defined in this specification. Note that `dwn-server` strictly matches
  `path === "/"` for `POST` JSON-RPC requests; deployments advertising a path
  prefix require a reverse proxy to strip the prefix before forwarding to
  `dwn-server`.

### Multi-Endpoint Resolution

When a DID document contains multiple [[ref:Service Endpoint]] URLs:

- Clients ****SHOULD**** implement failover by attempting alternative endpoints
  when a request fails.
- Sync agents maintain independent replication links per
  `(tenant DID, remote endpoint, projection ID, authorization epoch)` across
  endpoints so all configured replicas converge.
### Server-Side Forwarding and Record Delivery

`dwn-server` provides an optional fire-and-forget post-processing hook (`DeliveryService`) supporting two latency optimizations over HTTP `dwn.processMessage` (triggered on `202` replies from `dwn.processMessage` and non-`204` `Applied` replies from `dwn.applyReplicatedMessage`):

1. **Peer Endpoint Forwarding (`forwardingEnabled`)**: After processing (`202 Accepted`) a `RecordsWrite` or `RecordsDelete` for a tenant, the server resolves the tenant's other DWN service endpoints (cached with 300s TTL, excluding its own base URL, prioritizing full endpoints ahead of `cache` endpoints) and forwards the **original unmodified signed message** (plus stored data stream when `dataSize > 0`) via HTTP `dwn.processMessage`.
2. **Direct Record Delivery (`deliveryEnabled`, `$delivery: "direct"`)**: After processing (`202`) a `RecordsWrite` at a module path declaring `$delivery: "direct"`, the server resolves participant DIDs (explicit `recipient`, role holders, and ancestor actors) and forwards the original signed message (and data) to their DWN endpoints. (`$delivery: "subscribe"` is accepted in module schemas as a hint; server-side subscribe delivery is not implemented.)

Both forwarding and delivery use bounded retries (up to 2 retries at `1s` and `5s`, `10s` per-request timeout), treat `409` responses as idempotent success, maintain a short-lived mode-prefixed deduplication cache (`fwd:${tenant}:${id}` / `dlv:${targetDid}:${id}`, default 60s TTL, keyed on top-level `recordId`) to avoid loops, and never replace authoritative synchronization defined in the [DWN Sync Protocol](https://dwn-sync-spec.pages.dev). Transparent server-side read proxying to peer endpoints is not implemented (missing local record data returns `410`).

## JSON-RPC 2.0 Wire Format

All DWN messages sent between [[ref:DWN Clients]] and [[ref:DWN Servers]] are
wrapped in [JSON-RPC 2.0](https://www.jsonrpc.org/specification) envelopes.

### Request Object

```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "method": "dwn.processMessage",
  "params": {
    "target": "did:example:alice",
    "message": { ... }
  }
}
```

- `jsonrpc` — ****MUST**** be `"2.0"`.
- `id` — ****MAY**** be a `string`, `number`, or `null` (clients ****SHOULD****
  use a UUID v4 on request-response calls).
- `method` — ****MUST**** be one of the [registered methods](#methods).
- `params` — Method-specific parameters object (omitted for `rpc.ping`; for
  `rpc.subscribe.close`, `JsonRpcSocket` sends `{}` and servers ignore `params`).
- `subscription` — Required on `rpc.subscribe.dwn.processMessage`,
  `rpc.subscribe.close`, and `rpc.ack`: `{ "id": "<subscription-id>" }`.

### Response Object

#### Success Response

For `dwn.processMessage` and `rpc.subscribe.dwn.processMessage`:
```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "result": {
    "reply": {
      "status": { "code": 200, "detail": "OK" },
      "entries": [ ... ]
    }
  }
}
```

For `dwn.applyReplicatedMessage` (note the key is `result.result`, not `result.reply`):
```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "result": {
    "result": {
      "kind": "Applied",
      "position": {
        "streamId": "<stream-id>",
        "epoch": "<stream-epoch>",
        "position": "42",
        "messageCid": "bafy..."
      }
    }
  }
}
```

::: note
When a `dwn.processMessage` reply includes a data stream (`RecordsRead` or
`MessagesRead` with data) over HTTP, the `entry.data` field is **deleted** from
the JSON envelope and transmitted as the HTTP response body alongside the
`dwn-response` header (see [Response Format](#response-format)).
:::

#### Error Response

```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "error": {
    "code": -50429,
    "message": "Too many requests",
    "data": { "retryAfterSec": 5 }
  }
}
```

### Error Codes

Implementations use the following `JsonRpcErrorCodes`:

| Code | Name | Description |
|---|---|---|
| `-32700` | `ParseError` | Invalid JSON |
| `-32600` | `InvalidRequest` | Invalid JSON-RPC request, missing `subscription` object on `rpc.subscribe.dwn.processMessage` or `rpc.subscribe.close`, or tenant quota exceeded (`data: { code: "TenantMessageQuotaExceeded" \| "TenantStorageQuotaExceeded" }`) |
| `-32601` | `MethodNotFound` | Unknown JSON-RPC method |
| `-32602` | `InvalidParams` | Invalid or missing method parameters, missing `subscription` object on `rpc.ack`, duplicate/unknown subscription ID, disallowed WS `RecordsWrite`, invalid `encodedData`, or `includeMaterializationConfirmation: true` over WebSocket |
| `-32603` | `InternalError` | Unexpected internal server error |
| `-32300` | `TransportError` | Client/transport-level failure |
| `-50400` | `BadRequest` | Malformed HTTP RPC framing, missing/invalid JSON envelope, or invalid WebSocket JSON frame |
| `-50401` | `Unauthorized` | Authentication or authorization failure |
| `-50403` | `Forbidden` | Operation not permitted (e.g., `includeMaterializationConfirmation: true` without local-node authentication) |
| `-50409` | `Conflict` | State conflict |
| `-50429` | `TooManyRequests` | Per-tenant rate limit exceeded; includes `data: { retryAfterSec: number }` |

::: note
`ProgressGap` (`410`) is a **DWN reply** status (`result.reply.status.code === 410`
with `result.reply.error.code === "ProgressGap"`) returned inside a JSON-RPC
**success** response (or mid-stream as a `SubscriptionMessage` with
`type: "error"`), not a JSON-RPC error code.
:::

**Client Terminal vs. Retryable Classification (`DwnRpcError`):**
Clients classify `-32600`, `-32602`, `-50400`, `-50401`, `-50403`, and `-50409`
as terminal **except** when `code === -32600` carries a structured quota
exceeded code (`TenantMessageQuotaExceeded` or `TenantStorageQuotaExceeded`) —
or when `error.message` includes either quota error code substring — which sync
agents handle as retryable quota blocks.

### Methods

| Method | Transport | Description |
|---|---|---|
| `dwn.processMessage` | HTTP, WebSocket | Process a single DWN message |
| `dwn.applyReplicatedMessage` | HTTP, WebSocket | Admit a replicated message and return structured `ReplicationApplyResult` |
| `rpc.subscribe.dwn.processMessage` | WebSocket only | Open a `RecordsSubscribe` or `MessagesSubscribe` subscription |
| `rpc.subscribe.close` | WebSocket only | Close an active subscription |
| `rpc.ack` | WebSocket only | Acknowledge receipt of subscription messages (sliding-window flow control) |
| `rpc.ping` | WebSocket (router accepts any) | Application-level liveness ping |

#### `dwn.processMessage`

Processes a single DWN message against the target tenant's node.

**Parameters:**
- `target` — Tenant DID.
- `message` — DWN message object.

**Result:** `{ "reply": <DwnMessageReply> }`.

#### `dwn.applyReplicatedMessage`

Admits a replicated message through `Dwn.applyReplicatedMessage` and returns a
structured `ReplicationApplyResult`.

**Parameters:**
```json
{
  "target": "did:example:alice",
  "message": { ... },
  "encodedData": "<optional-unpadded-base64url-for-ws>",
  "ancestryOnly": true,
  "includeMaterializationConfirmation": true
}
```
- `target` — ****MUST**** be the tenant DID.
- `message` — ****MUST**** be the DWN message object. Inline `message.encodedData` ****MUST NOT**** be present (rejected with `-32602`); callers carrying inline data over WebSocket ****MUST**** use `params.encodedData`, and HTTP callers carrying data ****MUST**** stream it in the framed HTTP body. (If a caller supplies both `params.encodedData` and an HTTP binary stream, `dwn-server` uses `params.encodedData` and cancels the HTTP stream.)
- `encodedData` — ****MAY**** be present (typically over WebSocket). Strict unpadded base64url whose decoded byte length ****MUST**** equal `descriptor.dataSize` and ****MUST NOT**** exceed `maxRecordDataSize` (else `-32602`).
- `ancestryOnly` — ****MAY**** be `true` (only boolean `true` is valid; `false` is rejected with `-32602`). Requires a data-less initial `RecordsWrite` (`RecordsWrite.isInitialWrite(message)` with a payload descriptor and no inbound data stream or `encodedData`). Permitted over both HTTP and WebSocket.
- `includeMaterializationConfirmation` — ****MAY**** be a boolean. When `false`, it has no effect and is accepted on any transport. When `true`, it ****MUST**** be sent over HTTP (`transport === "http"`, else `-32602`) AND authenticated via the Local Node Profile (`isLocalNodeAuthenticated === true`, else `-50403` `Forbidden`); it requests `Duplicate.materialized: true` or `Superseded.currentWriteMaterialized: true` when local record data is positively observed. (Note: `Duplicate.materialized: true` is also returned without this flag whenever an exact duplicate `RecordsWrite` is replayed *with* inbound data; when `includeMaterializationConfirmation !== true`, proven-superseded writes short-circuit before quota enforcement and return `{ kind: "Superseded" }`.)

**Result:** `{ "result": <ReplicationApplyResult> }` where `result.kind` is one of:
- `"Applied"` (`{ kind: "Applied", ancestryOnly?: true, position?: ProgressToken }`)
- `"Duplicate"` (`{ kind: "Duplicate", materialized?: true }`)
- `"Superseded"` (`{ kind: "Superseded", currentWriteMaterialized?: true }`)
- `"Incomplete"` (`{ kind: "Incomplete", missing: DependencyRef[] }` where `DependencyRef.type` is `"Module" | "InitialWrite" | "Parent" | "Ancestor" | "Role" | "Grant" | "EncryptionControl" | "RecordData"`)
- `"Invalid"` (`{ kind: "Invalid", reason: string }`)
- `"Deferred"` (`{ kind: "Deferred", reason: "record-data-unavailable" | "resolver-unavailable" | "storage" | "tenant-inactive" }`)

#### `rpc.subscribe.dwn.processMessage`

Opens a real-time subscription on a WebSocket connection.

**Parameters:** `target` (tenant DID), `message` (`RecordsSubscribe` or `MessagesSubscribe`).
**Subscription:** `{ "id": "<client-assigned-id>" }` (duplicate IDs on the same connection return `-32602`).
**Result:** `{ "reply": <SubscribeReply> }` (including `entries` + pagination `cursor` for `RecordsSubscribe` snapshot mode, or `head` + `fingerprint` / `roleRecordId` for `MessagesSubscribe`).

#### `rpc.subscribe.close`

Closes an active subscription on the current WebSocket connection.

**Parameters:** None (clients such as `JsonRpcSocket` may send `{}`; servers ignore `params`).
**Subscription:** `{ "id": "<subscription-id>" }` (missing `subscription` returns `-32600`; unknown `id` returns `-32602`).
**Result:** `{ "reply": { "status": 200, "detail": "Accepted" } }`.

#### `rpc.ack`

Acknowledges receipt and processing of subscription messages for sliding-window
flow control.

**Parameters:** `{ "cursor": <ProgressToken> }` (`messageCid` may be omitted or a non-empty string).
**Subscription:** `{ "id": "<subscription-id>" }` (missing `subscription` returns `-32602`).
**Result:** The server responds with `{ "reply": { "status": 200, "detail": "OK" } }` (inventing a UUID `id` if the request omitted `id`).

#### `rpc.ping`

Application-level keepalive and on-demand connection health probe.

**Request:** `{ "jsonrpc": "2.0", "id": "<id>", "method": "rpc.ping" }` (no `params`).
**Result:** `{ "ok": true }`.

## HTTP Transport

The HTTP transport is used for request-response DWN message processing and
replicated apply, including arbitrarily large streaming binary payloads.

### Endpoint

All JSON-RPC HTTP requests are sent via HTTP `POST` to the base
[[ref:Service Endpoint]] URL (`POST /`). A `GET /` request returns a `200 OK`
`text/plain` server banner.

### Request Format

DWN HTTP requests use a **single binary-framed request body** with media type
`application/vnd.enbox.dwn-rpc`. The legacy `dwn-request` HTTP header is **not**
used and **not** accepted.

#### Content-Type Requirement

- `Content-Type` ****MUST**** be exact `application/vnd.enbox.dwn-rpc`
  (case-insensitive after trimming whitespace; media-type `;` parameters are
  rejected with HTTP `400` / `-50400`).

#### Binary Body Wire Layout

Every request body has the following exact binary layout:

```
+---------------+---------------------------+-------------------------------+-------------------------+
| Byte 0: flags | Bytes 1..4: uint32be len  | Bytes 5..(5+len-1): envelope  | Bytes (5+len)..EOF      |
| (0x00 | 0x01) | (1 <= len <= 1,048,576)   | UTF-8 JSON-RPC request bytes  | Optional raw data stream|
+---------------+---------------------------+-------------------------------+-------------------------+
```

- **Byte 0 (`flags`)**:
  - `0x00` — Envelope only; no trailing payload bytes should follow (any trailing bytes buffered in the same stream chunk as the envelope are rejected with HTTP `400` / `-50400`; the flag is authoritative and `http-dwn-rpc-framing.ts` cancels the reader without awaiting EOF once the envelope is read).
  - `0x01` (`FRAME_DATA_FOLLOWS_FLAG`) — Raw binary record data stream follows immediately after the JSON envelope.
  - Any other bit set is rejected with HTTP `400` / `-50400`.
- **Bytes 1..4 (`envelopeLength`)**: Unsigned 32-bit big-endian integer (`uint32be`) giving the exact byte length of the JSON-RPC envelope. ****MUST**** satisfy `1 <= envelopeLength <= 1_048_576` (**1 MiB**, `HTTP_DWN_RPC_MAX_ENVELOPE_BYTES`; `envelopeLength === 0` fails JSON parsing and likewise returns HTTP `400` / `-50400`).
- **Bytes 5..(5 + `envelopeLength` - 1)**: Strict UTF-8 encoded JSON-serialized `JsonRpcRequest` (`TextDecoder('utf-8', { fatal: true })`).
- **Bytes (5 + `envelopeLength`)..EOF**: When `flags === 0x01`, the raw binary data stream (for `RecordsWrite` or data-bearing `dwn.applyReplicatedMessage`). Clients sending a streaming body set Fetch `duplex: "half"`.

::: example Framed HTTP Request (Envelope + Binary Data)
```http
POST / HTTP/1.1
Host: dwn.example.com
Content-Type: application/vnd.enbox.dwn-rpc

<0x01><uint32be envelope length><UTF-8 {"jsonrpc":"2.0","id":"abc-123","method":"dwn.processMessage","params":{"target":"did:example:alice","message":{...}}}><raw binary record data>
```
:::

#### Transport Authentication Header

When connecting to a Local Node Profile endpoint (or any endpoint configured
with a bearer token), clients attach `Authorization: Bearer <token>` on HTTP
requests (`rpc-auth.ts`).

### Response Format

#### Response with Data Stream

When the DWN reply includes a data stream (`RecordsRead` or `MessagesRead` with data):

- The `dwn-response` HTTP header ****MUST**** contain `JSON.stringify(jsonRpcResponse)` with `entry.data` removed.
- The HTTP response body ****MUST**** contain the raw binary data stream.
- The `Content-Type` header ****MUST**** be `application/octet-stream`.

::: example RecordsRead HTTP Response with Data
```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
dwn-response: {"jsonrpc":"2.0","id":"abc-789","result":{"reply":{"status":{"code":200,"detail":"OK"},"entry":{"recordsWrite":{"recordId":"bafy...","contextId":"bafy...","descriptor":{"interface":"Records","method":"Write","dataFormat":"image/png",...},"authorization":{...}}}}}}

<binary record data>
```
:::

#### Response without Data Stream

For all other replies (queries, writes, deletes, module operations, replicated apply, etc.):

- The HTTP response body ****MUST**** contain `JSON.stringify(jsonRpcResponse)`.
- The `Content-Type` header ****MUST**** be `application/json`.

#### Error Responses

- **Framing / `Content-Type` / UTF-8 / invalid JSON envelope errors**: HTTP `400` with a JSON-RPC error response carrying code **`-50400`** (`BadRequest`). (Note: in current `dwn-server`, a valid non-object JSON value such as `null` passes JSON parsing and fails during routing with plain-text HTTP `500`.)
- **Per-IP rate limit exceeded**: HTTP `429 Too Many Requests` with `Retry-After` header and plain JSON body `{"error":"Rate limit exceeded"}`.
- **Per-tenant rate limit exceeded**: HTTP `429 Too Many Requests` with `Retry-After` header and JSON-RPC error code **`-50429`** (`TooManyRequests`) and `data: { retryAfterSec }`.
- **Local-node profile rejections**: Non-loopback `Host` header returns HTTP `403 Forbidden`; missing/invalid bearer token or mismatched `Origin` returns HTTP `401` with `{"error":"Unauthorized"}`.
- **All other JSON-RPC errors** returned by the JSON-RPC router (`-32600`, `-32601`, `-32602`, `-32603`, `-50403`, quota errors, and WebSocket-only subscription control methods sent over HTTP which return `-32600`): HTTP **`500`** with the JSON-RPC error envelope in the body.
- **DWN-level outcomes** (`200`, `202`, `204`, `400`, `401`, `404`, `409`, `410`, `500`, `501`): returned inside HTTP `200 OK` with the DWN status in `result.reply.status.code` (or `result.result` for `dwn.applyReplicatedMessage`).

#### Client Retry and Timeout Behavior

`HttpDwnRpcClient` applies default request timeouts of **30 seconds** (extended to **300 seconds** for `dwn.applyReplicatedMessage` whenever `message.descriptor.dataSize > 1_048_576`, including bodyless `ancestryOnly` writes) and retries transient network errors (`TypeError`) and HTTP `408`, `429`, `500`, `502`, `503`, `504` up to 3 times using jittered exponential backoff (`500ms` to `10s`, respecting `Retry-After` capped at `10s`), skipping retries on HTTP `500` responses containing a terminal JSON-RPC error and never retrying non-replayable `ReadableStream` bodies. `DwnServerInfoCacheMemory` caches `GET /info` responses for **15 minutes** by default.

### CORS

On public (non-local-node) deployments, servers return the following CORS
headers on all routes except `/admin*` and `/metrics` (including DWN `/`, REST,
`/info`, `/health`, `/registration*`, `/provider-auth/*`, `/local/*`, `404`
responses, and `/connect/*` relay routes defined in the
[Enbox Connect Protocol specification](https://github.com/enboxorg/web5-connect-spec)):

| Header | Value |
|---|---|
| `Access-Control-Allow-Origin` | `*` |
| `Access-Control-Allow-Methods` | `GET, POST, OPTIONS` |
| `Access-Control-Allow-Headers` | `*` |
| `Access-Control-Expose-Headers` | `dwn-response` |
| `Access-Control-Max-Age` | `86400` |

`OPTIONS` preflight requests return HTTP `204 No Content`.

Under the **Local Node Profile**, wildcard `*` is **never** used:
`Access-Control-Allow-Origin` reflects the request `Origin` with `Vary: Origin`
only for `/info`, `/local/pair`, matching-origin `/local/pair/:requestId`, or
already-paired origins, and `Access-Control-Allow-Headers` is narrowed to
`authorization, content-type`.

### Method Restrictions

All non-subscribe DWN methods (`dwn.processMessage` and
`dwn.applyReplicatedMessage`) are supported over HTTP. HTTP is required for
streaming large `RecordsWrite` data, `RecordsRead`/`MessagesRead` data streams,
and `dwn.applyReplicatedMessage` with `includeMaterializationConfirmation: true`.
(Note: `rpc.subscribe.dwn.processMessage` with a `Subscribe` message, as well as
`rpc.subscribe.close` and `rpc.ack`, return JSON-RPC `-32600` over HTTP; however,
`rpc.subscribe.dwn.processMessage` with a non-subscribe message is executed by
the router over HTTP.)

## WebSocket Transport

The WebSocket transport provides real-time subscriptions (`RecordsSubscribe`,
`MessagesSubscribe`), sliding-window backpressure, dual-layer liveness, and
pooled low-latency routing for eligible control-plane and small/ancestry-only
replicated-apply RPCs.

### Connection Establishment

Clients upgrade at the base [[ref:Service Endpoint]] URL (`GET /` with
`Upgrade: websocket`; note that `dwn-server` accepts WebSocket upgrades on any
`GET` path and runs upgrade handling prior to per-IP HTTP rate limiting).
Remote endpoints ****SHOULD**** use `wss://`; loopback local-node endpoints may
use `ws://`. Clients should check `ServerInfo.webSocketSupport` before opening
a WebSocket (`JsonRpcSocket` uses a default **3-second** `connectTimeout`; when
`webSocketSupport === false`, `dwn-server` does not attach a `SocketConnection`
to upgraded sockets).

Under the **Local Node Profile**, WebSocket upgrades are never public and
authenticate via `Authorization: Bearer <token>` OR query parameter
`?localNodeToken=<token>` (alongside strict `Origin` binding). Clients redact
query parameters and credentials from connection error messages.

### Message Framing and Payload Limits

Each WebSocket message carries a single JSON-serialized `JsonRpcRequest`
(client → server) or `JsonRpcResponse` (server → client). Batching is not
supported.

- **Server maximum frame size**: `maxWsJsonRpcPayloadBytes(maxRecordDataSize) = Math.ceil(maxRecordDataSize / 3) * 4 + 65_536` (for the default `100 MiB` `maxRecordDataSize`, ~`133.4 MiB`, accommodating base64url `dwn.applyReplicatedMessage` payloads).
- **Client pooled-socket opportunistic routing cap**: `WS_JSON_RPC_ENVELOPE_BYTES = 65_536` (**64 KiB total UTF-8 JSON-RPC frame**, including both envelope metadata and base64url-encoded `params.encodedData` — roughly 47–48 KiB of raw record bytes).
- **Invalid JSON / empty frame**: Returns a JSON-RPC error with code **`-50400`** (`BadRequest`) and a server-generated UUID `id`.

### Request-Response Correlation

Requests and responses are correlated by JSON-RPC `id`. Subscription pushes
carry the client-assigned `subscription.id` as the JSON-RPC response `id`.
(Implementation note: in current `dwn-server`, `enforceQuota` sets the error
response `id` to `message.recordId ?? crypto.randomUUID()` rather than the
incoming JSON-RPC `id`, so quota errors over WebSocket do not correlate with
pending request IDs.)

### Subscriptions

WebSocket is the ****only**** transport supporting `RecordsSubscribe` and
`MessagesSubscribe`.

#### Opening a Subscription

The client sends `rpc.subscribe.dwn.processMessage` with `subscription: { id }`:

```json
{
  "jsonrpc": "2.0",
  "id": "req-1",
  "method": "rpc.subscribe.dwn.processMessage",
  "params": {
    "target": "did:example:alice",
    "message": {
      "descriptor": {
        "interface": "Records",
        "method": "Subscribe",
        "messageTimestamp": "2026-06-01T00:00:00.000000Z",
        "filter": { "module": "https://social.example/modules/feed" }
      },
      "authorization": { ... }
    }
  },
  "subscription": { "id": "sub-1" }
}
```

- The server reserves `subscription.id` before async validation/replay (duplicate ID returns `-32602`).
- In cursor mode, replay frames may arrive before the opening JSON-RPC response resolves; clients refresh their 30-second subscribe timeout as replay frames arrive.
- **Snapshot & anchor replies**:
  - `RecordsSubscribe` (no `cursor`): reply includes `entries` and optional pagination `cursor`; no EOSE is sent.
  - `MessagesSubscribe` (owner/delegate): reply includes `head` (current log head, or empty-log anchor `{ streamId, epoch, position: "0" }` omitting `messageCid`) and optional canonical-scope `fingerprint`, captured after the live listener is registered. Role-authorized `MessagesSubscribe` replies omit `head`/`fingerprint` and include `roleRecordId`.
- **`ProgressGap` (`410`)**: If `cursor` is no longer replayable, the DWN returns a JSON-RPC success response whose `result.reply` has `status: { code: 410, detail: "Progress token gap" }` and `error: { code: "ProgressGap", requested, oldestAvailable, latestAvailable, reason }` where `reason` is `"token_too_old" | "token_too_new" | "epoch_mismatch" | "stream_mismatch" | "message_mismatch"`.

#### Subscription Events

Pushed frames set `id` to `subscription.id` and `result.subscription` to a
3-way [[ref:SubscriptionMessage]] union:

##### Event Messages (`type: "event"`)

```json
{
  "jsonrpc": "2.0",
  "id": "sub-1",
  "result": {
    "subscription": {
      "type": "event",
      "cursor": {
        "streamId": "<stream-id>",
        "epoch": "<stream-epoch>",
        "position": "<monotonic-position>",
        "messageCid": "<message-cid>"
      },
      "seq": "<monotonic-position>",
      "messageCid": "<message-cid>",
      "isLatestBaseState": true,
      "module": "https://social.example/modules/feed",
      "encodedData": "<optional-base64url>",
      "event": {
        "message": { ... },
        "initialWrite": { ... }
      }
    }
  }
}
```

| Field | Type | Description |
|---|---|---|
| `type` | `"event"` | Event discriminator |
| `cursor` | `ProgressToken` | Replay token used for `rpc.ack` flow control and reconnection |
| `seq` | `string` (optional) | Source feed decimal sequence position |
| `messageCid` | `string` (optional) | Stored message CID |
| `isLatestBaseState` | `boolean` (optional) | Whether the message was indexed as latest base state |
| `module` | `string` (optional) | Indexed module URI |
| `encodedData` | `string` (optional) | Inline base64url data (`<= 30,000` bytes; detached from `event.message` and `event.initialWrite`, omitted in `metadataOnly` mode) |
| `event` | `object` | `{ message, initialWrite? }` |

##### EOSE (`type: "eose"`)

Delivered in cursor mode after stored replay reaches the frozen catch-up
high-water token (`bounds.latest`, or input `cursor` if the log is empty):

```json
{
  "jsonrpc": "2.0",
  "id": "sub-1",
  "result": {
    "subscription": {
      "type": "eose",
      "cursor": {
        "streamId": "<stream-id>",
        "epoch": "<stream-epoch>",
        "position": "<monotonic-position>",
        "messageCid": "<optional-message-cid>"
      }
    }
  }
}
```

##### Terminal Error Messages (`type: "error"`)

Sent when the server terminates a subscription mid-stream (e.g., delivery-time
grant/role revocation, mid-stream `ProgressGap`, `SubscriptionBufferOverflow`,
or `SubscriptionBufferTimeout`):

```json
{
  "jsonrpc": "2.0",
  "id": "sub-1",
  "result": {
    "subscription": {
      "type": "error",
      "cursor": {
        "streamId": "<stream-id>",
        "epoch": "<stream-epoch>",
        "position": "<monotonic-position>",
        "messageCid": "<optional-message-cid>"
      },
      "error": {
        "code": "SubscriptionBufferTimeout",
        "detail": "subscription ACKs did not free buffer capacity before the deadline"
      }
    }
  }
}
```

Clients ****MUST NOT**** treat `cursor` on an error message as an applied checkpoint.

#### Closing a Subscription

```json
{
  "jsonrpc": "2.0",
  "id": "req-2",
  "method": "rpc.subscribe.close",
  "subscription": { "id": "sub-1" }
}
```

Server stops delivery, cleans up resources, and replies
`{ "reply": { "status": 200, "detail": "Accepted" } }`. Closing the WebSocket
automatically closes all active subscriptions on that connection.

#### Flow Control (Backpressure)

The server maintains a per-subscription sliding window (`FlowController`) with:
- **`maxInFlight`**: default `32` (advertised in `/info`).
- **`MAX_BUFFER_SIZE`**: `1000` buffered messages beyond `maxInFlight`.
- **`MAX_BUFFER_WAIT_MS`**: `30_000` ms (30 seconds). When the buffer is full,
  the producer awaits capacity for up to 30 seconds; on timeout the server emits
  terminal `{ type: "error", cursor, error: { code: "SubscriptionBufferTimeout", detail } }`
  and closes the subscription. Pushing past `1000` without awaiting emits
  `SubscriptionBufferOverflow` and closes the subscription.

Both `"event"` and `"eose"` messages consume window slots and ****MUST**** be
acknowledged via `rpc.ack`:

```json
{
  "jsonrpc": "2.0",
  "method": "rpc.ack",
  "params": {
    "cursor": {
      "streamId": "<stream-id>",
      "epoch": "<stream-epoch>",
      "position": "<monotonic-position>",
      "messageCid": "<optional-message-cid>"
    }
  },
  "subscription": { "id": "sub-1" }
}
```

- `ack(cursor)` verifies `streamId` and `epoch` against the oldest unacked entry
  and cumulatively acknowledges all unacked entries up to `cursor.position`
  (matching `messageCid` when present). Unknown/stale tokens are ignored.
- Clients chain `rpc.ack` strictly in arrival order, gating both `lastCursor`
  advancement and `rpc.ack` emission on the consumer handler promise completing
  (halting on `SubscriptionHandlerTerminalError`).
- `rpc.ack` is transport-layer flow control only; durable sync checkpoints are
  persisted separately by clients/agents after local admission.

### Heartbeat and Wake Liveness

Connections use **dual-layer liveness**:

1. **Server RFC 6455 `ping` / `pong`**: The server sends WebSocket `ping` frames
   every **30 seconds** and terminates connections that fail to respond with
   `pong` before the next interval.
2. **Client `rpc.ping` heartbeat**: Clients send application-level `rpc.ping`
   every **30 seconds** with a **10-second** response timeout.
3. **Wake-triggered on-demand probe (`checkHealth`)**: On browser `online` and
   `visibilitychange` (`visible`) events, `WebSocketDwnRpcClient` sends an
   out-of-band `rpc.ping` with a **5-second** timeout to immediately detect
   silently dead connections after sleep/backgrounding and fast-forward
   reconnection.
4. **Reconnection and resubscription**: Clients reconnect with jittered
   exponential backoff (`1s` to `30s`), pause reconnect attempts while offline,
   emit local transport lifecycle notifications (`disconnected`, `reconnecting`,
   `reconnected`), and re-open generic subscriptions via
   `resubscribeFactory(lastCursor)` (emitting a client-synthesized terminal
   `SubscriptionError` with `code: "SubscriptionRecoveryFailed"` if automatic
   resubscription fails).

### Method Restrictions and Pooled WebSocket Routing

| Operation | HTTP | WebSocket | Notes |
|---|---|---|---|
| `dwn.processMessage` — `RecordsWrite` | Yes | **No** | Server rejects ALL `RecordsWrite` via `dwn.processMessage` over WS with `-32602` (even `dataSize === 0`). |
| `dwn.processMessage` — `RecordsRead` / `MessagesRead` | Yes | Avoid | Server does not stream binary data over WS (cancels stream and omits data); clients route Reads over HTTP. |
| `dwn.processMessage` — `RecordsQuery`, `RecordsCount`, `RecordsDelete`, `ModulesConfigure`, `ModulesQuery`, `MessagesQuery` | Yes | Yes | Pooled WS client opportunistically routes requests whose total UTF-8 JSON-RPC frame is `<= 64 KiB` without `signal`/`timeoutMs` over an already-connected socket. |
| `dwn.applyReplicatedMessage` (with streaming data) | Yes | Conditional | Over WS, supported for `dataSize === 0`, `ancestryOnly: true`, or inline `params.encodedData` within frame limits. Opportunistic pooled WS routing accepts only `undefined`, `Blob`, `ArrayBuffer`, or `ArrayBufferView` payloads whose full UTF-8 JSON-RPC frame (envelope + base64url data) is `<= 64 KiB` (`ReadableStream` and `string` fall back to HTTP). |
| `dwn.applyReplicatedMessage` (`includeMaterializationConfirmation: true`) | Yes (local-node auth only) | **No** | Rejected over WS with `-32602`; requires local-node auth over HTTP (`-50403` otherwise). (`false` is accepted on either transport.) |
| `rpc.subscribe.dwn.processMessage` (`RecordsSubscribe`, `MessagesSubscribe`) | **No** | Yes | WebSocket only (returns `-32600` for Subscribe messages over HTTP). |
| `rpc.subscribe.close`, `rpc.ack`, `rpc.ping` | No (`rpc.ping` accepted by router) | Yes | WebSocket control methods (`close` and `ack` return `-32600` over HTTP). |

## REST Convenience API

[[ref:DWN Servers]] expose read-only REST endpoints that internally construct
anonymous (unsigned) DWN messages for publicly readable records and published
modules. Read and module-query endpoints map `401` responses to empty HTTP `404`
to avoid existence oracles (whereas `GET /:did/query` unconditionally returns
HTTP `200` with the JSON `RecordsQueryReply`). Prototype-pollution query keys
(`__proto__`, `constructor`, and `prototype`) are silently ignored (stripped)
by REST query parsers. `dwn-server` does not emit `Cache-Control` or `ETag`
headers on REST responses.

### Record Read

```
GET /:did/read/records/:id
GET /:did/records/:id
```

- **200 with data**: Body is the raw record bytes; `Content-Type` is
  `descriptor.dataFormat`; `dwn-response` header contains `JSON.stringify(reply)`
  (a raw `RecordsReadReply`, not a JSON-RPC envelope; in current `dwn-server`,
  `entry.data` is not deleted prior to serialization and appears as `"data":{}`).
- **200 without data**: Returns HTTP `400`.
- **401**: Returns empty HTTP `404`.
- **404**: Returns HTTP `404` with the JSON `RecordsReadReply` body.
- **Other status codes** (e.g., `410`): Returns that HTTP status with JSON reply.

### Module Record Read

```
GET /:did/read/modules/:module/*
```

- `:module` — base64url-encoded module URI.
- `*` — non-empty `modulePath` (empty path returns HTTP `400`).
- Optional dot-notation query parameters (e.g., `filter.schema`, `filter.dataFormat`, `filter.tags.*`).
- Queries matching records with `pagination.limit = 1` and `dateSort = publishedDescending`, then returns the `RecordsRead` response (same response format as [Record Read](#record-read)).

### Module Definition Read

```
GET /:did/read/modules/:module
```

- `:module` — base64url-encoded module URI.
- Issues anonymous `ModulesQuery` with `filter: { module }`.
- **200**: Body is `entries[0]` (the full `ModulesConfigure` message JSON).
- **404**: Module not installed or not published (`401` mapped to empty `404`).

### Module Query

```
GET /:did/query/modules
```

- Issues anonymous `ModulesQuery` with no filter.
- **200**: Body is JSON array `entries` of all published `ModulesConfigure` messages (`401` mapped to empty `404`).

### Record Query

```
GET /:did/query
```

Queries publicly readable records using dot-notation query parameters mapped
directly as string leaf values into `RecordsQuery` options:

| Parameter | Description |
|---|---|
| `filter.module` | Normalized module URI |
| `filter.modulePath` | Module path |
| `filter.schema` | Schema URI |
| `filter.dataFormat` | Media type |
| `filter.recordId` | Specific record ID |
| `filter.contextId` | Context ID subtree |
| `filter.tags.<key>` | String tag filter value |
| `dateSort` | `createdAscending`, `createdDescending`, `publishedAscending`, `publishedDescending`, `updatedAscending`, `updatedDescending` |

::: note
Because `dwn-server`'s REST query parser assigns query parameter values as raw
strings without numeric type coercion (and DWN JSON Schema validation does not
coerce types), passing `pagination.limit=<n>` fails schema validation (`type: "number"`)
and returns HTTP `400`. Similarly, `pagination.cursor` is a structured
`{ messageCid, value }` object rather than a scalar string.
:::

::: example Record Query
```
GET /did:example:alice/query?filter.module=https://social.example/modules/feed&filter.modulePath=post&dateSort=publishedDescending
```
:::

- **200**: JSON-serialized `RecordsQueryReply` (returned with HTTP `200 OK`
  regardless of the inner `reply.status.code`, including `401`).
- **400**: `{ "error": "Bad Request" }` on malformed parameters that fail
  `RecordsQuery.create` schema validation.

## Server Information

### Info Endpoint

```
GET /info
```

**Response (`200 OK`, `application/json` — multi-tenant remote server example):**

```json
{
  "maxFileSize": 104857600,
  "maxInFlight": 32,
  "registrationRequirements": [
    "proof-of-work-sha256-v0",
    "terms-of-service",
    "provider-auth-v0"
  ],
  "server": "@enbox/dwn-server",
  "sdkVersion": "0.4.32",
  "url": "https://dwn.example.com",
  "version": "0.1.48",
  "webSocketSupport": true,
  "providerAuth": {
    "authorizeUrl": "https://provider.example.com/authorize",
    "tokenUrl": "https://provider.example.com/token",
    "refreshUrl": "https://provider.example.com/refresh",
    "managementUrl": "https://provider.example.com/account"
  }
}
```

**Response (`200 OK`, `application/json` — Local Node Profile example):**

```json
{
  "maxFileSize": 1073741824,
  "maxInFlight": 32,
  "registrationRequirements": [],
  "server": "@enbox/dwn-server",
  "sdkVersion": "0.4.32",
  "url": "http://127.0.0.1:55500",
  "version": "0.1.48",
  "webSocketSupport": true,
  "localNode": true,
  "localPairing": {
    "pairUrl": "http://127.0.0.1:55500/local/pair",
    "pollUrlTemplate": "http://127.0.0.1:55500/local/pair/{requestId}"
  }
}
```

#### ServerInfo Object

| Property | Type | Required | Description |
|---|---|---|---|
| `server` | `string` | Yes | Server implementation identifier (e.g., `"@enbox/dwn-server"`) |
| `version` | `string` | Optional | Server package version (semver, omitted if unavailable) |
| `sdkVersion` | `string` | Optional | DWN SDK version (semver, omitted if unavailable) |
| `url` | `string` | Yes | Canonical base URL of the server |
| `webSocketSupport` | `boolean` | Yes | Whether WebSocket connections are enabled |
| `maxFileSize` | `number` | Yes | Advertised maximum record data size in bytes (default `104857600` = 100 MiB on `dwn-server`; `1073741824` = 1 GiB on `local-node`) |
| `maxInFlight` | `number` | Optional | Maximum unacknowledged events per subscription (default `32`; when absent, server does not advertise backpressure) |
| `registrationRequirements` | `string[]` | Yes | Active requirements (`"proof-of-work-sha256-v0"`, `"terms-of-service"`, `"provider-auth-v0"`; empty if open) |
| `localNode` | `boolean` | Optional | `true` when running under the opt-in Local Node Profile |
| `localPairing` | `object` | Optional | Present when `localNode === true`: `{ pairUrl, pollUrlTemplate }` derived from `url` |
| `providerAuth` | `object` | Conditional | Present whenever `"provider-auth-v0"` is in `registrationRequirements`: `{ authorizeUrl, tokenUrl, refreshUrl?, managementUrl? }` |

### Health Endpoint

```
GET /health
```

Returns `200 OK`, `application/json`, `{"ok": true}`. (Protected by bearer auth when `localNodeProfileEnabled === true`.)

## Tenant Registration

Multi-tenant [[ref:DWN Servers]] ****MAY**** require tenant registration before
accepting DWN messages for a DID, advertising active requirements in
`ServerInfo.registrationRequirements`. Disabled registration routes (e.g.,
`/registration/proof-of-work` when PoW is disabled, `/registration/terms-of-service`
when ToS is unset, or `/registration` when no registration store is configured)
return HTTP `404 Not Found`.

### Proof-of-Work Challenge

```
GET /registration/proof-of-work
```

**Response (`200 OK`, `application/json`):**

```json
{
  "challengeNonce": "A1B2C3D4E5F6...64_HEX_CHARS...",
  "maximumAllowedHashValue": "000000FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF"
}
```

| Property | Type | Description |
|---|---|---|
| `challengeNonce` | `string` | 64-character hex server challenge nonce rotating every 10 minutes. Accepts previous & current nonces (uppercase random hex); when `challengeSeed` (`DWN_REGISTRATION_PROOF_OF_WORK_SEED`) is configured, nonces are deterministic lowercase SHA-256 hex and a `nextChallengeNonce` window is also accepted. |
| `maximumAllowedHashValue` | `string` | 64-character hex upper bound adjusted adaptively (target 10 solves/minute, reevaluated every 10 seconds) |

#### Solving the Challenge

The client ****MUST**** find a **64-character hex** `responseNonce` (32 random
bytes; non-reusable while present in the last-minute set, pruned on the 10-second
difficulty timer for an effective 60–70s replay window) bound to the exact
serialized `registrationData` such that:

```
BigInt("0x" + SHA-256_hex(challengeNonce + responseNonce + JSON.stringify(registrationData)))
  <= BigInt("0x" + maximumAllowedHashValue)
```

Binding `JSON.stringify(registrationData)` prevents an attacker from pre-mining
solutions independent of the target DID and terms-of-service hash. Because
standard `JSON.stringify` (without JCS canonicalization) is used on both client
and server, non-JavaScript clients ****MUST**** preserve the key order parsed
from the submitted JSON (`did`, then `termsOfServiceHash`, with no whitespace).

### Terms of Service

```
GET /registration/terms-of-service
```

Returns the terms of service as `text/plain`. The client computes the lowercase
hex SHA-256 hash of the text and passes it in `registrationData.termsOfServiceHash`.

### Provider Authentication Registration (`provider-auth-v0`)

When `provider-auth-v0` is advertised along with `ServerInfo.providerAuth`:

1. Client directs the user to `{providerAuth.authorizeUrl}` with query parameters
   `redirect_uri` and random `state`.
2. After user authentication/authorization, the provider returns an authorization
   `code` and matching `state` (note: external providers typically redirect via
   `302`; `dwn-server`'s built-in `OpenAuthHandler` at
   `GET /provider-auth/authorize` returns HTTP `200` JSON `{ code, state?, redirectUri }`,
   or `503` if pending codes reach 10,000).
3. Client exchanges the code via `POST {providerAuth.tokenUrl}`:
   ```json
   {
     "grantType": "authorization_code",
     "code": "<authorization_code>",
     "redirectUri": "<redirect_uri>"
   }
   ```
   Response (`200 OK`):
   ```json
   {
     "registrationToken": "<opaque_token>",
     "refreshToken": "<optional_refresh_token>",
     "expiresIn": 3600,
     "tokenType": "bearer"
   }
   ```
   (`refreshToken` and `expiresIn` are optional; `OpenAuthHandler` does not
   require `grantType`, though `DwnRegistrar` includes it.)
4. If `providerAuth.refreshUrl` is advertised, client may refresh via
   `POST {providerAuth.refreshUrl}` with `{ "grantType": "refresh_token", "refreshToken": "<token>" }`.
5. `dwn-server` includes a built-in `JwtProviderAuthPlugin` (HS256 secret or
   remote JWKS verification, mapping `sub` → `accountId`) and built-in
   `OpenAuthHandler` routes (`/provider-auth/authorize`, `/provider-auth/token`,
   `/provider-auth/refresh`).

### Registration Submission

```
POST /registration
Content-Type: application/json
```

```json
{
  "registrationData": {
    "did": "did:example:alice",
    "termsOfServiceHash": "e3b0c44298fc1c149afbf4c8996fb924..."
  },
  "proofOfWork": {
    "challengeNonce": "<64-char-hex>",
    "responseNonce": "<64-char-hex>"
  },
  "providerAuth": {
    "registrationToken": "<opaque-token>"
  }
}
```

- **Precedence**: `providerAuth` and `proofOfWork` are evaluated as mutually
  exclusive alternatives: if `request.providerAuth?.registrationToken !== undefined`,
  provider-auth registration runs; otherwise if `request.proofOfWork === undefined`,
  the server rejects with HTTP `400` (`RegistrationRequestMissingCredentials`);
  otherwise proof-of-work registration runs.
- **Terms of Service**: Whenever the server has a Terms of Service configured,
  clients ****SHOULD**** include `registrationData.termsOfServiceHash` even when
  registering via `providerAuth` so active-tenant checks match the configured
  ToS hash. (Note: `@enbox/auth`'s `registerPair` currently omits
  `termsOfServiceHash` on provider-auth registrations.)
- **Responses**:
  - **200**: `{ "success": true }`
  - **400**: `{ "code": "<DwnServerErrorCode>", "message": "<code: detail>" }`
  - **500**: `{ "success": false }` (malformed JSON bodies currently fail before
    the JSON error handler and return plain-text HTTP `500`)

## Sync Transport Binding

Full synchronization semantics — including replication feed sequences, XOR
fingerprints, `ReplicationApplyResult` / `DependencyRef` closures, sync scopes,
followed contexts, reconciliation flows, fault tolerance, and `sync-next` — are
defined in the companion [DWN Sync Protocol](https://dwn-sync-spec.pages.dev).

At the transport layer, sync agents bind to this specification as follows:

- **Durable feed queries and reads**: Sent via `dwn.processMessage` over HTTP
  (`MessagesQuery`, `MessagesRead`, `RecordsRead` with
  `includeReplicationSupport: true`), or opportunistically over pooled WebSockets
  for eligible control-plane queries `<= 64 KiB`.
- **Replicated admission (`dwn.applyReplicatedMessage`)**:
  - Streaming/large writes use HTTP `application/vnd.enbox.dwn-rpc` binary
    framing (`300s` default client timeout when `descriptor.dataSize > 1 MiB`).
  - Zero-data writes, `ancestryOnly: true` writes, and small inline
    `params.encodedData` writes whose complete UTF-8 JSON-RPC frame is
    `<= 64 KiB` may be opportunistically routed over an already-connected pooled
    WebSocket.
  - `includeMaterializationConfirmation: true` ****MUST**** use HTTP
    authenticated via the Local Node Profile.
  - Remote quota rejections return JSON-RPC `-32600` with structured
    `TenantMessageQuotaExceeded` / `TenantStorageQuotaExceeded` data.
- **Cursorless live subscriptions**: Sync agents open and reconnect live
  `MessagesSubscribe` streams **cursorless** over WebSocket (gating `rpc.ack`
  on local admission completion) and trigger durable pull/push reconciliation
  passes on reconnect rather than relying on cursor-mode WS catch-up replay.
- **Stateless servers**: Servers maintain only connection-local flow-control
  state in memory and never persist per-peer sync checkpoints.

## Local Transport (Local Node Profile)

### Overview

`dwn-server` and `@enbox/local-node` provide an opt-in **Local Node Profile**
(`localNodeProfileEnabled: true` / `DWN_LOCAL_NODE_PROFILE=true`) for on-device
HTTP and WebSocket communication between browser apps, CLI tools, or desktop
apps and a local DWN process.

All local traffic reuses the standard JSON-RPC 2.0 wire format, binary HTTP
framing (`application/vnd.enbox.dwn-rpc`), and WebSocket subscriptions, layered
with loopback isolation and origin-bound pairing authentication. `@enbox/local-node`
inherits default `dwn-server` rate limits as well as `forwardingEnabled: true`
and `deliveryEnabled: true`.

### Loopback Isolation and Anti-DNS-Rebinding

When `localNodeProfileEnabled` is enabled:
- **Bind enforcement**: The server defaults `hostname` to `127.0.0.1` and
  refuses to start unless bound to a loopback address (`localhost`, `::1`, or
  `127.0.0.0/8`, stripping an optional trailing `.` and rejecting `*.localhost`).
- **`Host` header validation**: Every incoming HTTP request and WebSocket
  upgrade ****MUST**** carry a loopback `Host` hostname; non-loopback `Host`
  headers are rejected with HTTP **`403 Forbidden`** to defeat DNS-rebinding
  attacks.
- **Strict CORS**: Wildcard `*` is disabled; `Access-Control-Allow-Origin`
  reflects `Origin` (with `Vary: Origin`) only for `/info`, `/local/pair`,
  matching-origin `/local/pair/:requestId`, or already-paired origins.
- **Public vs. protected routes**: Only `OPTIONS *`, `GET /info`,
  `POST /local/pair`, and `GET /local/pair/:requestId` are unauthenticated.
  All other routes — including WebSocket upgrades, `/`, `/local/status`,
  `/health`, REST routes, `/registration*`, `/provider-auth/*`, `/connect/*`,
  `/admin*`, and `/metrics` — require valid local-node authentication (HTTP
  `401` `{"error":"Unauthorized"}` otherwise).

### Pairing and Session Authentication

Tokens are 32 random bytes encoded as base64url and verified with constant-time
comparison (`timingSafeEqual`):

- **Browser origins** authenticate via `Authorization: Bearer <token>` (HTTP &
  WS) or `?localNodeToken=<token>` (WS upgrade), and the session is bound to the
  exact `http(s)` `Origin` header (`origin === session.origin`).
- **CLI / native processes** authenticate using the origin-less `localNodeToken`
  from the [[ref:Discovery File]], and ****MUST NOT**** send an `Origin` header
  (`origin === undefined`).

#### Pairing Endpoints

1. **`POST /local/pair`**:
   - Requires a valid `http:` or `https:` `Origin` header (`400` otherwise);
     request body is ignored (clients such as `@enbox/auth` send `{}`).
   - Coalesces duplicate pending requests for the same origin **before**
     rate-limiting; new requests are rate-limited to **5 attempts per 60 seconds**
     per origin (`429` + `Retry-After`).
   - Auto-approves synchronously if the origin is in `localNodeAllowedOrigins`,
     and expires after **5 minutes** (`300_000` ms).
   - Response (`200 OK`): `{ "requestId": "<uuid>", "status": "pending" }`
     (returned with `"pending"` even when auto-approved; clients ****MUST****
     poll `GET /local/pair/:requestId` to retrieve the issued token).
2. **`GET /local/pair/:requestId`**:
   - Empty `:requestId` returns HTTP `400` (`{"error":"Pairing request ID is required."}`).
   - Requires an `Origin` header matching the initiating origin (`404` otherwise).
   - Response (`200 OK`):
     `{ "status": "pending" | "approved" | "denied" | "expired", "origin": "<origin>", "token"?: "<32-byte-base64url>" }`.
   - On `"approved"`, **`token` is returned ONCE** (the first successful poll
     marks `tokenReturned = true`; subsequent polls omit `token`).
   - Terminal requests are retained for 5 minutes before pruning.
3. **`GET /local/status`** (authenticated):
   - Response (`200 OK`): `{ "localNode": true, "origin"?: "<origin>", "paired": true }`.
   - Used for passive revalidation of stored browser pairings.

#### Persistence, Revocation, and Local-Only Capabilities

- `@enbox/local-node` persists approved browser sessions at
  `~/.enbox/local-node-sessions.json` (`0o600` on POSIX, `{ version: 1, sessions: [{ createdAt, origin, token }] }`)
  and creates an ephemeral origin-less `localNodeToken` for CLI/native apps.
- Session revocation is host-local programmatic only (`DwnServer.revokeLocalNodePairingToken`
  / `LocalNode.revokePairingToken`; no HTTP `/local/revoke` route exists).
  Revoking a session token deletes it and **immediately closes all live
  WebSocket connections** authenticated by that token (using default close code
  `1000` without a custom reason).
- `dwn.applyReplicatedMessage` permits `includeMaterializationConfirmation: true`
  only over HTTP on authenticated Local Node Profile connections.

### Local Discovery

#### Discovery File (`~/.enbox/dwn.json`)

`@enbox/local-node` listens on the first available port in `55500..55509, 3000`
and writes `~/.enbox/dwn.json` (`0o600` permissions on POSIX platforms):

```json
{
  "endpoint": "http://127.0.0.1:55500",
  "pid": 12345,
  "capabilities": ["http", "ws"],
  "localNodeToken": "<32-byte-base64url-cli-token>"
}
```

| Property | Type | Required | Description |
|---|---|---|---|
| `endpoint` | `string` | Yes | Loopback URL where the local server is listening (non-empty string in `isValidRecord`) |
| `pid` | `number` | Yes | OS process ID of the local server (required by `isValidRecord`; stale or corrupt files are deleted on read) |
| `capabilities` | `string[]` | No | Advertised transport capabilities (e.g., `["http", "ws"]`) |
| `localNodeToken` | `string` | No | Origin-less bearer token for CLI/native callers |

#### Passive Startup Discovery and User-Gesture Probing

- **CLI / native**: `LocalDwnDiscovery` reads `~/.enbox/dwn.json` and verifies
  `GET /info` reports `server === "@enbox/dwn-server"` (only `@enbox/local-node`
  single-instance detection and browser discovery additionally require
  `localNode === true`).
- **Browser startup**: Strictly **passive** — never sweeps localhost ports
  automatically (avoiding browser Private Network Access permission prompts).
  Revalidates persisted pairings via `GET /info` (`localNode === true`) +
  `GET /local/status`.
- **Explicit user-gesture probing (`probeLocalDwn`)**: First validates any
  persisted pairing; otherwise checks environment support (returning
  `{ status: "unsupported", reason }` for insecure contexts, Safari, or
  `"no-fetch"` when `fetch` is unavailable) and probes `127.0.0.1` ports
  `55500..55509, 3000` only on explicit user action.
- **Desktop scheme (`dwn://connect`)**: Desktop launchers (such as
  `electrobun-dwn`, which runs `DwnServer` without enabling the Local Node
  Profile or writing `localNodeToken`) handle
  `dwn://connect?callback=<encoded-url>` by redirecting to
  `<callback>#<base64url({"endpoint":"http://127.0.0.1:..."})>`, validated by
  `dwn-discovery-payload.ts` to loopback hostnames (`127.0.0.1`, `::1`,
  `localhost`, or `*.localhost`).

## Security Considerations

### TLS Requirements

Remote [[ref:DWN Servers]] ****SHOULD**** be deployed behind TLS (`https://` and
`wss://`). Loopback [[ref:Local Transport]] endpoints (`127.0.0.1`, `::1`,
`localhost`) ****MAY**** use plaintext `http://` and `ws://`.

### Rate Limiting and Tenant Quotas

`dwn-server` enforces token-bucket rate limits at three levels:
1. **Per-IP HTTP**: default `30 req/s`, burst `50` → HTTP `429`, `Retry-After`,
   `{"error":"Rate limit exceeded"}` (WebSocket upgrades bypass per-IP HTTP rate
   limiting).
2. **Per-tenant DID (HTTP & WebSocket)**: default `20 req/s`, burst `50` →
   JSON-RPC `-50429` (`TooManyRequests`, `data: { retryAfterSec }`), plus HTTP
   `429` + `Retry-After` on HTTP.
3. **Per-origin local pairing (`POST /local/pair`)**: `5` requests per `60s` →
   HTTP `429` + `Retry-After`.

Per-tenant message-count and storage quotas (active when `adminStore` is
configured via `adminToken`) return JSON-RPC `-32600` (`InvalidRequest`) with
structured `data: { code: "TenantMessageQuotaExceeded" | "TenantStorageQuotaExceeded" }`.

### Message Size Limits

- **HTTP RPC JSON envelope**: strictly capped at **1 MiB**
  (`HTTP_DWN_RPC_MAX_ENVELOPE_BYTES = 1_048_576`) by `http-dwn-rpc-framing.ts`.
- **Record data (`maxFileSize` / `maxRecordDataSize`)**: default **100 MiB**
  (`104_857_600`) on `dwn-server` (`1 GiB` on `local-node`), advertised in
  `GET /info` and enforced on WebSocket `maxPayloadLength` and
  `dwn.applyReplicatedMessage` inline `params.encodedData` (`-32602`). Note that
  current `dwn-server` does not explicitly enforce `maxRecordDataSize` on HTTP
  `POST /` record streams (relying on the HTTP runtime body limit), and inbound
  HTTP stream capping to `descriptor.dataSize` is applied only in
  `dwn.applyReplicatedMessage` (not `dwn.processMessage`).

### WebSocket, Header, and Input Hardening

- Servers enforce RFC 6455 `ping` timeouts (30s), `maxPayloadLength`, and
  per-subscription flow-control bounds (`maxInFlight: 32`, `MAX_BUFFER_SIZE: 1000`,
  `MAX_BUFFER_WAIT_MS: 30s`).
- `dwn-response` headers carry JSON envelopes and ****MUST**** be parsed safely
  by clients; REST query parsers silently ignore prototype-pollution keys
  (`__proto__`, `constructor`, `prototype`).
- Internal server errors are redacted to generic messages on the wire, and
  WebSocket clients strip query strings/credentials (`?localNodeToken=...`) from
  error logs.
- Local-node servers enforce loopback bind, `Host` header validation (`403`),
  constant-time token comparison, origin binding, `0o600` credential file
  permissions (POSIX), single-use pairing token delivery, and immediate
  WebSocket closure upon token revocation.

### Transport-Layer Privacy

::: note
TLS protects wire payloads in transit, but endpoint URLs (DNS/SNI), client IP
addresses, and request timing/sizes remain observable at the transport layer.
Record-level encryption (defined in the DWN specification) protects data at rest
and in transit independently of transport encryption. Deployments requiring
stronger transit anonymity may wrap DWN JSON-RPC messages in onion-routed or
relay-mediated transport layers while preserving end-to-end DWN message
authorization and CID verification at the destination node.
:::

## Normative References

[DWN Specification](https://dwn-spec.pages.dev)
~ Enbox Decentralized Web Node Specification.

[DWN Sync Protocol Specification](https://dwn-sync-spec.pages.dev)
~ Enbox Decentralized Web Node Sync Protocol Specification.

[Enbox Connect Protocol Specification](https://web5-connect-spec.pages.dev)
~ Enbox Connect Protocol (v2) Specification.

[JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
~ JSON-RPC Working Group. March 2010.

[[spec:RFC4648]]
~ [The Base16, Base32, and Base64 Data Encodings](https://datatracker.ietf.org/doc/html/rfc4648). S. Josefsson. October 2006.

[[spec:RFC6455]]
~ [The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455). I. Fette, A. Melnikov. December 2011.

[[spec:RFC8259]]
~ [The JavaScript Object Notation (JSON) Data Interchange Format](https://datatracker.ietf.org/doc/html/rfc8259). T. Bray. December 2017.

[[spec:RFC2119]]
~ [Key words for use in RFCs to Indicate Requirement Levels](https://datatracker.ietf.org/doc/html/rfc2119). S. Bradner. March 1997.

[RFC 9110 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
~ R. Fielding, M. Nottingham, J. Reschke. June 2022.
