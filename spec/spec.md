
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

The [Decentralized Web Node (DWN) specification](https://dwn-spec.pages.dev)
defines what a node does with a message. It does not say how a message reaches
a node. This specification does: it defines how a client finds the servers
that host a DID's node, and how it exchanges messages with them over HTTP and
WebSocket.

It covers:

- finding a DID's servers from its DID document;
- the JSON-RPC messages that carry DWN messages and replies;
- the HTTP binding, for requests and for record data of any size;
- the WebSocket binding, for subscriptions;
- a small read-only REST interface for public data;
- how a server describes itself, and how a tenant registers with one; and
- the *Local Node Profile*, for a server that runs on the user's own device.

## Status of This Document

DWN Transport Protocol is a _DRAFT_ specification under development by
[Enbox](https://enbox.org). It is a companion to the
[DWN specification](https://github.com/enboxorg/dwn-spec), and describes the
transport as implemented by the reference implementation,
[enboxorg/enbox](https://github.com/enboxorg/enbox).

The specification will be updated to incorporate feedback. We encourage
reviewers to submit
[GitHub Issues](https://github.com/enboxorg/dwn-transport-spec/issues) as the
means by which to communicate feedback and contributions.

The key words "****MUST****", "****MUST NOT****", "****REQUIRED****",
"****SHALL****", "****SHALL NOT****", "****SHOULD****", "****SHOULD NOT****",
"****RECOMMENDED****", "****MAY****", and "****OPTIONAL****" in this document
are to be interpreted as described in [[spec:RFC2119]].

Where this document gives a number as what "the reference server" or "the
reference client" does, the number is a default of the reference
implementation and not a requirement.

## Terminology

[[def:Decentralized Web Node, Decentralized Web Nodes, DWN, Node, Nodes]]
~ A personal data store and message relay, as defined in the
[DWN specification](https://github.com/enboxorg/dwn-spec).

[[def:Tenant, Tenants]]
~ The DID whose data a [[ref:Node]] holds. Every request names the tenant it is
addressed to.

[[def:DWN Server, DWN Servers, Server]]
~ A service that hosts the [[ref:Nodes]] of one or more [[ref:Tenants]] and
exposes them over the transports defined here.

[[def:DWN Client, DWN Clients, Client]]
~ Any software that sends requests to a [[ref:DWN Server]]: an application, a
wallet, a command-line tool, a sync client, or another server.

[[def:Service Endpoint, Service Endpoints]]
~ A URL, published in a DID document, at which a [[ref:DWN Server]] hosts the
[[ref:Node]] of that DID.

[[def:Subscription, Subscriptions]]
~ A stream of messages that a [[ref:Server]] pushes to a [[ref:Client]] over a
WebSocket connection, in answer to a `RecordsSubscribe` or `MessagesSubscribe`
message.

[[def:Progress Token]]
~ A position in a tenant's message feed, `{ streamId, epoch, position, messageCid? }`,
as defined in the DWN specification. A client treats it as opaque.

[[def:Local Node]]
~ A [[ref:DWN Server]] that runs on the user's own device and accepts
connections only from that device. See
[Local Node Profile](#local-node-profile).

[[def:Discovery File]]
~ The file `~/.enbox/dwn.json`, in which a [[ref:Local Node]] tells other
programs on the device where it is listening.

## Overview

| A client wants to | It uses |
|---|---|
| Find a DID's servers | The DID document's `DecentralizedWebNode` service ([Service Endpoint Discovery](#service-endpoint-discovery)) |
| Send a DWN message and get the reply | `dwn.processMessage` over HTTP, or over WebSocket for messages without data |
| Write or read record data | `dwn.processMessage` over HTTP; the data travels in the body |
| Deliver a message during synchronization | `dwn.applyReplicatedMessage` |
| Follow changes as they happen | `rpc.subscribe.dwn.processMessage` over WebSocket |
| Read public data with a plain `GET` | The [REST Convenience API](#rest-convenience-api) |
| Learn a server's limits and requirements | `GET /info` |

A server does not authenticate callers at the transport layer, except under
the [Local Node Profile](#local-node-profile). Authority comes from the
signatures on the DWN messages themselves, which the node verifies.

## Service Endpoint Discovery

### DID Document Service Entry

A DID names its servers in its DID document, in a service entry:

- `id` ****MUST**** be `<did>#dwn`.
- `type` ****MUST**** be `DecentralizedWebNode`.
- `serviceEndpoint` ****MUST**** be a URL string, or a non-empty array of URL
  strings. Several URLs name several servers that hold the same data.

::: example DID Document with a DWN Service Entry
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

A client finds the entry by its `id`, also accepting the relative forms `#dwn`
and `dwn`. Resolving a DID for its endpoints has one of four outcomes:

| Outcome | Meaning |
|---|---|
| `ready` | The entry is present and well formed. The result is its endpoint URLs, normalized |
| `service-missing` | The DID document has no such entry. This is not an error for a DID that needs no node, such as a delegate |
| `service-malformed` | The entry has the wrong `type`, or a `serviceEndpoint` that is not a URL string or a non-empty array of valid URL strings |
| `resolution-failed` | The DID itself could not be resolved |

### Endpoint URLs

- An endpoint URL ****MUST**** use `https:` or `http:`. A server reachable over a
  network ****SHOULD**** use `https:`.
- An endpoint URL ****MUST NOT**** have a query or a fragment.
- A client normalizes endpoint URLs by removing trailing `/` characters, and
  drops duplicates.
- Every path in this specification is relative to the normalized endpoint URL.
  The reference server serves from the root path only, so a deployment that
  publishes an endpoint with a path prefix has to strip the prefix before
  requests reach the server.

### Several Endpoints

When a DID lists several endpoints, each is a server holding the same DID's
data.

- A client ****SHOULD**** try another endpoint when a request to one fails,
  including when a read answers that the server does not hold a record's data
  (status `410`).
- A sync client keeps each endpoint in agreement with its own node separately,
  as the [DWN Sync Protocol](https://dwn-sync-spec.pages.dev) defines.

### Server-Side Forwarding and Record Delivery

A server ****MAY**** pass on messages it has just accepted, so that other
servers learn of them sooner than synchronization alone would deliver them.
The reference server has two such features, both off by default and both
best-effort. Neither replaces the
[DWN Sync Protocol](https://dwn-sync-spec.pages.dev), which is what guarantees
agreement.

1. **Forwarding to the tenant's other endpoints.** After accepting a
   `RecordsWrite` or `RecordsDelete` for a tenant, the server resolves the
   tenant's DID and sends the message, unmodified and with its data, to each
   of the tenant's other endpoints.
2. **Delivery to participants.** After accepting a `RecordsWrite` at a module
   path whose rule set has `$delivery: "direct"`, the server works out who
   takes part in the record: its `recipient`, the holders of the roles its
   rules name, and the authors or recipients of the ancestors its rules name.
   It sends the message and its data to each participant's endpoints.
   `$delivery: "subscribe"` is accepted in module definitions and has no
   server behavior.

Both send the original signed message with `dwn.processMessage`, so the
receiving node validates it exactly as it would from any other sender. A `409`
reply counts as success. The reference server retries a failed send twice,
after 1 and 5 seconds, with a 10-second timeout per request; caches the
endpoints it resolves for 300 seconds; and remembers what it has just sent for
60 seconds, so that two servers do not pass the same record back and forth.

When it resolves endpoints for these two features, a server is more lenient
than a client: it considers every service of type `DecentralizedWebNode`
whatever its `id`, and also accepts endpoints written as objects,
`{ "url": "<endpoint URL>", "dataRetention": "full" | "cache" }`, trying
`full` endpoints first.

A server does not fetch record data it lacks from other endpoints on a
reader's behalf. A read of a record whose data it does not hold answers `410`.

## JSON-RPC Messages

Every exchange between a [[ref:Client]] and a [[ref:Server]] is a
[JSON-RPC 2.0](https://www.jsonrpc.org/specification) request and response.
Batches are not supported.

### Requests

```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "method": "dwn.processMessage",
  "params": {
    "target": "did:example:alice",
    "message": { }
  }
}
```

- `jsonrpc` — ****MUST**** be `"2.0"`.
- `id` — a string, a number, or `null`. A client ****SHOULD**** use a fresh UUID for each request.
- `method` — one of the [methods](#methods) below.
- `params` — the method's parameters.
- `subscription` — `{ "id": "<subscription ID>" }`. Present on the three subscription methods only. The ID is chosen by the client and names the subscription on its connection.

### Responses

A response carries the `id` of its request, and either `result` or `error`.

```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "result": {
    "reply": { "status": { "code": 200, "detail": "OK" }, "entries": [ ] }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "error": { "code": -50429, "message": "Too many requests", "data": { "retryAfterSec": 5 } }
}
```

The two layers of outcome are separate. A JSON-RPC `error` means the server
did not hand the message to the node, or could not. Once the node has
processed the message, the response is a JSON-RPC success, and the node's own
verdict is the `status` inside `result.reply`. A reply with status `401`,
`409`, or `410` is therefore still a JSON-RPC success.

### Error Codes

| Code | Name | Sent when |
|---|---|---|
| `-32600` | `InvalidRequest` | A subscription is attempted over HTTP, or without a `subscription` object; or the tenant is over its quota (see below) |
| `-32601` | `MethodNotFound` | The method is unknown |
| `-32602` | `InvalidParams` | The parameters are missing or invalid; a subscription ID is already in use, or is unknown; or the request is not allowed on this transport |
| `-32603` | `InternalError` | The server failed unexpectedly |
| `-50400` | `BadRequest` | The HTTP framing is malformed, or a WebSocket frame is empty or not valid JSON |
| `-50403` | `Forbidden` | The caller may not use an option reserved for local callers |
| `-50429` | `TooManyRequests` | The tenant's rate limit is exceeded. `data.retryAfterSec` says when to retry |
| `-32700`, `-50401`, `-50409` | `ParseError`, `Unauthorized`, `Conflict` | Reserved. The reference server does not send them |
| `-32300` | `TransportError` | Never sent. A client reports it locally when a connection is lost with requests outstanding |

**Quota.** A server that limits how many messages or bytes a tenant may store
refuses a write over the limit with `-32600`, with `data.code` set to
`TenantMessageQuotaExceeded` or `TenantStorageQuotaExceeded`. The same code
appears at the start of `message`.

**Retrying.** A client treats `-32600`, `-32602`, `-50400`, `-50401`,
`-50403`, and `-50409` as final: repeating the request will not help. A quota
error is the exception, since it may clear later; a sync client retries it on
a schedule (see the [DWN Sync Protocol](https://dwn-sync-spec.pages.dev)).

### Methods

| Method | HTTP | WebSocket | Purpose |
|---|---|---|---|
| `dwn.processMessage` | Yes | In part | Process a DWN message |
| `dwn.applyReplicatedMessage` | Yes | In part | Admit a message during synchronization |
| `rpc.subscribe.dwn.processMessage` | No | Yes | Open a subscription |
| `rpc.subscribe.close` | No | Yes | Close a subscription |
| `rpc.ack` | No | Yes | Acknowledge subscription messages |
| `rpc.ping` | Yes | Yes | Check that a connection is alive |

A server ****MUST**** answer a subscription method sent over HTTP with
`-32600`.

#### `dwn.processMessage`

Processes one DWN message at the tenant's node.

- **Parameters:** `target`, the tenant's DID; and `message`, the DWN message.
- **Result:** `{ "reply": <the node's reply> }`.

Record data never travels inside this method's JSON:

- The data of a `RecordsWrite` is sent in the HTTP request body (see
  [Request Framing](#request-framing)). A server ****MUST**** therefore reject
  every `RecordsWrite` sent with this method over WebSocket, with `-32602`.
- The data of a `RecordsRead` or `MessagesRead` reply is sent in the HTTP
  response body (see [HTTP Responses](#http-responses)). Over WebSocket the
  reply arrives without its data, so a client ****SHOULD**** send reads over
  HTTP.

Data small enough to be inline, in `encodedData`, is part of the reply itself
and is unaffected.

#### `dwn.applyReplicatedMessage`

Admits one message that a sync client is copying from another node. The node
applies the same rules as for `dwn.processMessage`, but answers with a
structured outcome that tells the client what to do next. The outcomes, and
when a node reports each, are defined by the
[DWN Sync Protocol](https://dwn-sync-spec.pages.dev).

```json
{
  "target": "did:example:alice",
  "message": { },
  "encodedData": "<base64url>",
  "ancestryOnly": true,
  "includeMaterializationConfirmation": true
}
```

- `target` — ****REQUIRED****. The tenant's DID.
- `message` — ****REQUIRED****. The DWN message. It ****MUST NOT**** contain `encodedData`.
- `encodedData` — ****OPTIONAL****. The record's data as unpadded base64url, for a caller that cannot send a body, which is to say over WebSocket. Its decoded length ****MUST**** equal `descriptor.dataSize` and ****MUST NOT**** exceed the server's `maxFileSize`. Over HTTP the data normally travels in the body instead; if a request has both, the server uses `encodedData`.
- `ancestryOnly` — ****OPTIONAL****. If present it ****MUST**** be `true`. It states that the caller is deliberately sending an initial `RecordsWrite` without its data, so that the node can validate the record's descendants. The message ****MUST**** then be an initial `RecordsWrite`, and no data may accompany it.
- `includeMaterializationConfirmation` — ****OPTIONAL****. `true` asks the node to report whether it holds the record's data. Because the answer reveals what the node stores, a server ****MUST**** accept `true` only over HTTP, and only from a caller authenticated under the [Local Node Profile](#local-node-profile). `false` has no effect.

A request that breaks one of these rules is answered with `-32602`, except
that `includeMaterializationConfirmation: true` from a caller that is not a
local caller is answered with `-50403`.

Over WebSocket, a `RecordsWrite` whose `dataSize` is greater than zero is
accepted only with `encodedData` or with `ancestryOnly: true`.

- **Result:** `{ "result": <outcome> }`, where the outcome's `kind` is `Applied`, `Duplicate`, `Superseded`, `Incomplete`, `Deferred`, or `Invalid`. Note the property name: `result.result`, not `result.reply`.

```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "result": {
    "result": {
      "kind": "Applied",
      "position": { "streamId": "8f3a…", "epoch": "6d1e…", "position": "42", "messageCid": "bafyrei..." }
    }
  }
}
```

#### `rpc.subscribe.dwn.processMessage`

Opens a [[ref:Subscription]] on a WebSocket connection. See
[Subscriptions](#carrying-subscriptions).

- **Parameters:** `target`, and `message`, a `RecordsSubscribe` or `MessagesSubscribe` message.
- **`subscription`:** `{ "id": "<ID chosen by the client>" }`. A missing `subscription` is answered with `-32600`, and an ID already in use on the connection with `-32602`.
- **Result:** `{ "reply": <the node's reply> }`.

#### `rpc.subscribe.close`

Closes a subscription.

- **Parameters:** none.
- **`subscription`:** `{ "id": "<subscription ID>" }`. A missing `subscription` is answered with `-32600`, and an unknown ID with `-32602`.
- **Result:** `{ "reply": { "status": 200, "detail": "Accepted" } }`.

#### `rpc.ack`

Acknowledges messages received on a subscription. See
[Flow Control](#flow-control).

- **Parameters:** `{ "cursor": <progress token> }`, the `cursor` of the last message being acknowledged.
- **`subscription`:** `{ "id": "<subscription ID>" }`. A missing `subscription` or an invalid `cursor` is answered with `-32602`.
- **Result:** `{ "reply": { "status": 200, "detail": "OK" } }`. If the request had no `id`, the server answers with one of its own.

#### `rpc.ping`

Checks that the connection and the server are alive.

- **Parameters:** none.
- **Result:** `{ "ok": true }`.

## HTTP Transport

A client sends a JSON-RPC request as an HTTP `POST` to the
[[ref:Service Endpoint]] URL. The body holds the request and, when there is
any, the record's data, so that data of any size can be streamed without
being encoded into JSON.

(`GET` on the same URL returns a short plain-text notice. It is not part of
the protocol.)

### Request Framing

The request's `Content-Type` ****MUST**** be `application/vnd.enbox.dwn-rpc`,
with no parameters. A server compares it without regard to case.

The body has three parts:

```
+--------+---------------------+----------------------------+------------------+
| flags  | envelope length     | envelope                   | data             |
| 1 byte | 4 bytes, big-endian | JSON-RPC request, UTF-8    | to end of body   |
+--------+---------------------+----------------------------+------------------+
```

- **Flags.** `0x00` means the body ends with the envelope. `0x01` means the
  record's data follows the envelope. Any other value is invalid.
- **Envelope length.** The length of the envelope in bytes, as an unsigned
  32-bit big-endian integer. It ****MUST**** be at least 1 and at most
  1,048,576 (1 MiB).
- **Envelope.** The JSON-RPC request, as JSON in UTF-8.
- **Data.** Present only with flags `0x01`: the raw bytes of the record's data,
  running to the end of the body. Its length is not stated in the frame; it
  is `descriptor.dataSize` of the message.

With flags `0x00` a client ****MUST NOT**** send anything after the envelope.
A server stops reading at the end of the envelope, and ****MAY**** reject a
body in which more bytes follow.

A server answers a body that breaks any of these rules, including an envelope
that is not valid UTF-8 or not valid JSON, with HTTP `400` and JSON-RPC error
`-50400`.

::: example A `RecordsWrite` with Its Data
```http
POST / HTTP/1.1
Host: dwn.example.com
Content-Type: application/vnd.enbox.dwn-rpc

<0x01><4-byte envelope length>{"jsonrpc":"2.0","id":"abc-123","method":"dwn.processMessage","params":{"target":"did:example:alice","message":{...}}}<record data>
```
:::

### HTTP Responses

**A reply with a data stream.** When the node's reply carries record data (a
`RecordsRead`, or a `MessagesRead` of a record with data):

- the response body is the data, with `Content-Type: application/octet-stream`; and
- the JSON-RPC response is in the `dwn-response` header, as JSON, without the data.

::: example A `RecordsRead` Response
```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
dwn-response: {"jsonrpc":"2.0","id":"abc-789","result":{"reply":{"status":{"code":200,"detail":"OK"},"entry":{"recordsWrite":{...}}}}}

<record data>
```
:::

**Any other reply.** The response body is the JSON-RPC response, with
`Content-Type: application/json`.

### HTTP Status Codes

| HTTP status | When | Body |
|---|---|---|
| `200` | The server produced a JSON-RPC success, whatever status the node's reply carries | As above |
| `400` | The framing or the envelope is malformed | JSON-RPC error `-50400` |
| `429` | The tenant's rate limit is exceeded | JSON-RPC error `-50429`, with a `Retry-After` header |
| `429` | The caller's own rate limit, by IP address, is exceeded | `{"error":"Rate limit exceeded"}`, with a `Retry-After` header |
| `500` | Any other JSON-RPC error | The JSON-RPC error response |
| `401`, `403` | Only under the [Local Node Profile](#local-node-profile) | See there |

A client therefore reads the JSON-RPC response on `200`, `400`, `429`, and
`500` alike, and looks for the node's verdict in `result.reply.status` only
when the response is a success.

### CORS

A public server answers every route, except administrative ones, with:

| Header | Value |
|---|---|
| `Access-Control-Allow-Origin` | `*` |
| `Access-Control-Allow-Methods` | `GET, POST, OPTIONS` |
| `Access-Control-Allow-Headers` | `*` |
| `Access-Control-Expose-Headers` | `dwn-response` |
| `Access-Control-Max-Age` | `86400` |

and answers `OPTIONS` with `204`. A [[ref:Local Node]] is stricter; see
[Local Node Profile](#local-node-profile).

## WebSocket Transport

A WebSocket connection carries the same JSON-RPC messages. It exists for
[[ref:Subscriptions]], which need the server to send messages of its own
accord. A client may also use an open connection for small requests that
carry no data stream.

### Connecting

A client opens a WebSocket at the [[ref:Service Endpoint]] URL, with the
scheme changed to `wss:` (or `ws:` for `http:`). A client ****SHOULD**** first
check that the server's `GET /info` reports `webSocketSupport: true`.

### Frames

Each WebSocket message is one JSON-RPC request or response, as JSON text.

- A server accepts frames up to `ceil(maxFileSize / 3) * 4 + 65,536` bytes:
  room for the largest record it accepts, base64url-encoded, plus the rest of
  the request.
- A server answers an empty frame, or one that is not valid JSON, with error
  `-50400` and an `id` of its own choosing, since it could not read the
  request's.

### Carrying Subscriptions

The DWN specification defines what a subscription delivers: `event`, `eose`,
and `error` messages, in feed order. This section defines how they travel.

#### Opening

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
      "authorization": { }
    }
  },
  "subscription": { "id": "sub-1" }
}
```

The response carries the node's reply to the subscribe message. If its status
is not `200`, no subscription was opened.

Two identifiers are in play, and only one matters here. `subscription.id` in
the *request* is chosen by the client; it names the subscription on this
connection, and is what `rpc.ack` and `rpc.subscribe.close` refer to. The
`subscription.id` inside the node's *reply* is the node's own identifier for
the subscription, and the transport does not use it.

When the subscribe message has a `cursor`, stored events can arrive before
the response to the request does. A client ****MUST**** be ready for
subscription messages as soon as it has sent the request.

#### Subscription Messages

The server pushes each subscription message as a JSON-RPC response whose `id`
is the subscription ID and whose `result.subscription` is the message:

```json
{
  "jsonrpc": "2.0",
  "id": "sub-1",
  "result": {
    "subscription": {
      "type": "event",
      "cursor": { "streamId": "8f3a…", "epoch": "6d1e…", "position": "44", "messageCid": "bafyrei..." },
      "event": { "message": { } }
    }
  }
}
```

A client tells these from ordinary responses by their `id`. It
****MUST NOT**** reuse a subscription ID as a request `id` on the same
connection.

#### Closing

```json
{
  "jsonrpc": "2.0",
  "id": "req-2",
  "method": "rpc.subscribe.close",
  "subscription": { "id": "sub-1" }
}
```

The server stops delivering and releases the subscription. Closing the
connection closes every subscription on it.

### Flow Control

A server limits how many messages a subscription may have outstanding, so
that a slow client cannot make it buffer without bound.

- At most `maxInFlight` messages may be sent and not yet acknowledged. The
  server advertises `maxInFlight` in `GET /info`; the reference server's is 32.
- Further messages wait in a buffer. The reference server holds up to 1000.
- When the buffer is full, the server stops reading from the feed for that
  subscription. If no acknowledgement frees space within 30 seconds, it sends
  an `error` message with code `SubscriptionBufferTimeout` and closes the
  subscription. If the buffer is overrun regardless, it sends
  `SubscriptionBufferOverflow` and closes the subscription.

A client acknowledges with `rpc.ack`, passing the `cursor` of a message it
received:

```json
{
  "jsonrpc": "2.0",
  "id": "req-3",
  "method": "rpc.ack",
  "params": {
    "cursor": { "streamId": "8f3a…", "epoch": "6d1e…", "position": "44", "messageCid": "bafyrei..." }
  },
  "subscription": { "id": "sub-1" }
}
```

- Both `event` and `eose` messages count, and both ****MUST**** be acknowledged.
- An acknowledgement names one outstanding message by its `cursor`, and covers
  that message and every message sent before it. The server matches on
  `position`, and on `messageCid` when the `cursor` has one. It ignores an
  acknowledgement that names no outstanding message, or whose `streamId` or
  `epoch` differ from those of the oldest outstanding message.
- A client ****MUST**** acknowledge messages in the order they arrived, and
  ****MUST NOT**** acknowledge a message before it has finished handling it.
- An acknowledgement only regulates delivery. It is not a record of progress.
  A client that needs to resume later keeps its own position, and a sync
  client advances its checkpoint only once a message has been admitted to its
  own node.

### Liveness and Reconnection

- **Server.** A server sends a WebSocket `ping` frame every 30 seconds, and
  closes a connection that has not answered with `pong` by the next one.
- **Client.** A client ****SHOULD**** send `rpc.ping` periodically, and treat a
  missing answer as a dead connection. The reference client pings every 30
  seconds and waits 10 seconds for the answer. In a browser, it also pings
  once, with a 5-second wait, whenever the device comes back online or the
  page becomes visible, so that a connection that died while the device slept
  is noticed at once.
- **Reconnecting.** When a connection is lost, every subscription on it is
  gone. A client reconnects, and opens its subscriptions again. The reference
  client waits between attempts with jittered exponential backoff from 1 to
  30 seconds, and does not try while the device is offline.

A subscription reopened with the `cursor` of the last message the client
handled resumes from there. A sync client does not rely on that: it reopens
its subscriptions without a cursor and covers the gap by querying the feed
(see the [DWN Sync Protocol](https://dwn-sync-spec.pages.dev)).

A server keeps no record of a client between connections. Everything it knows
about a subscription lives and dies with the connection.

## REST Convenience API

A server offers a few `GET` routes for data that anyone may read. Each builds
an unsigned DWN message and returns what the node answers, so they reach only
published records and published modules.

Where a route would otherwise reveal that something exists but is not public,
it answers `404`, exactly as if it did not exist.

### Read a Record

```
GET /:did/records/:recordId
GET /:did/read/records/:recordId
```

| Node's reply | HTTP response |
|---|---|
| `200`, with data | `200`. The body is the record's data, with `Content-Type` set to the record's `dataFormat`. The `dwn-response` header holds the `RecordsRead` reply as JSON |
| `401` | `404`, with an empty body |
| Any other status | That status, with the reply as a JSON body |

Unlike the header of the same name on the JSON-RPC route, this
`dwn-response` header holds the bare reply, not a JSON-RPC response. A client
****MUST**** ignore any `data` property in it.

### Read a Record by Module Path

```
GET /:did/read/modules/:module/:modulePath
```

- `:module` is the module URI, base64url-encoded.
- `:modulePath` is a module path, and may contain `/`.
- Query parameters of the form `filter.<property>` add to the filter, as for
  [Query Records](#query-records).

The server finds the most recently published record that matches, and answers
as for [Read a Record](#read-a-record). It answers `404` if none matches, and
`400` if the module path is empty.

### Read a Module Definition

```
GET /:did/read/modules/:module
```

Answers `200` with the module's `ModulesConfigure` message as JSON, or `404`
if the module is not installed or not published.

### List Module Definitions

```
GET /:did/query/modules
```

Answers `200` with a JSON array of the `ModulesConfigure` messages of the
tenant's published modules.

### Query Records

```
GET /:did/query
```

Runs a `RecordsQuery` built from the query parameters. A parameter name is a
path into the query, with `.` between levels:

| Parameter | Meaning |
|---|---|
| `filter.module` | Module URI |
| `filter.modulePath` | Module path |
| `filter.schema` | Schema URI |
| `filter.dataFormat` | Media type |
| `filter.recordId` | Record ID |
| `filter.contextId` | Context ID |
| `filter.tags.<name>` | Value of a string tag |
| `dateSort` | `createdAscending`, `createdDescending`, `publishedAscending`, `publishedDescending`, `updatedAscending`, or `updatedDescending` |

Every value is passed as a string. Properties that are not strings, such as
`pagination.limit`, therefore cannot be set this way; a client that needs
them uses `dwn.processMessage`.

::: example Query Records
```
GET /did:example:alice/query?filter.module=https://social.example/modules/feed&filter.modulePath=post&dateSort=publishedDescending
```
:::

The response is `200` with the `RecordsQuery` reply as JSON, whatever status
the reply itself carries; or `400` with `{"error":"Bad Request"}` if the
parameters do not make a valid query.

## Server Information

### `GET /info`

Returns, as JSON, what a client needs to know before it uses a server.

```json
{
  "server": "@enbox/dwn-server",
  "version": "0.1.48",
  "sdkVersion": "0.4.32",
  "url": "https://dwn.example.com",
  "webSocketSupport": true,
  "maxFileSize": 104857600,
  "maxInFlight": 32,
  "registrationRequirements": ["proof-of-work-sha256-v0", "terms-of-service"]
}
```

| Property | Type | Meaning |
|---|---|---|
| `server` | string | ****REQUIRED****. The name of the server software |
| `version` | string | The server's version |
| `sdkVersion` | string | The version of the DWN implementation it runs |
| `url` | string | ****REQUIRED****. The server's own base URL |
| `webSocketSupport` | boolean | ****REQUIRED****. Whether the server accepts WebSocket connections |
| `maxFileSize` | number | ****REQUIRED****. The largest record data the server accepts, in bytes. The reference server's default is 104,857,600 (100 MiB) |
| `maxInFlight` | number | How many subscription messages may be unacknowledged at once (see [Flow Control](#flow-control)). Absent if the server applies no limit |
| `registrationRequirements` | string[] | ****REQUIRED****. What a tenant must do to register: any of `proof-of-work-sha256-v0`, `terms-of-service`, and `provider-auth-v0`. Empty if the server is open to all |
| `providerAuth` | object | Present with `provider-auth-v0`: `{ authorizeUrl, tokenUrl, refreshUrl?, managementUrl? }` (see [Provider Authentication](#provider-authentication)) |
| `localNode` | boolean | `true` for a [[ref:Local Node]] |
| `localPairing` | object | Present with `localNode`: `{ pairUrl, pollUrlTemplate }` (see [Pairing](#pairing)) |

The reference client caches this document for 15 minutes.

### `GET /health`

Returns `200` with `{"ok": true}`.

## Tenant Registration

A server that hosts many tenants ****MAY**** require a DID to register before
it accepts messages for it. `registrationRequirements` in `GET /info` says
what registration takes. A registration route for a requirement the server
does not have answers `404`.

A tenant registers by [submitting](#submitting-a-registration) its DID with
one credential: a proof of work, or a token from an account provider.

A message for a DID that is not an active tenant is not processed. The reply
to `dwn.processMessage` has status `401`, and the outcome of
`dwn.applyReplicatedMessage` is `Deferred`, with the reason `tenant-inactive`.

### Proof of Work

```
GET /registration/proof-of-work
```

```json
{
  "challengeNonce": "<64 hex characters>",
  "maximumAllowedHashValue": "<64 hex characters>"
}
```

The client searches for a `responseNonce`, 64 hex characters of its own
choosing, such that

```
SHA-256( challengeNonce || responseNonce || registrationData )
```

read as a 256-bit unsigned integer, is no greater than
`maximumAllowedHashValue`. The three inputs are concatenated as text:
`challengeNonce` and `responseNonce` as the hex strings they are, and
`registrationData` as the exact JSON text of the `registrationData` object
that the client will submit.

Because the hash covers `registrationData`, a solution cannot be reused for
another DID. It also means the server hashes the JSON as *it* re-serializes
what it received, with the properties in the order they arrived and no
whitespace. A client ****MUST**** therefore send, and hash, `registrationData`
in exactly that form: `{"did":"…","termsOfServiceHash":"…"}`.

The server changes `challengeNonce` periodically and accepts the previous one
for a while. It adjusts `maximumAllowedHashValue` to the rate at which
solutions arrive. It refuses a `responseNonce` it has seen recently. The
reference server changes the nonce every 10 minutes, and adjusts the
difficulty every 10 seconds toward 10 solutions a minute.

### Terms of Service

```
GET /registration/terms-of-service
```

Returns the terms as plain text. To accept them, a client sends the SHA-256
hash of that text, in lowercase hex, as `registrationData.termsOfServiceHash`.

A tenant is active only while the hash it accepted is the hash of the
server's current terms. When the terms change, the tenant has to register
again. Until it does, the server does not process messages for it.

### Provider Authentication

With `provider-auth-v0`, an account provider vouches for the user instead of
a proof of work. `providerAuth` in `GET /info` gives the provider's URLs.

1. The client sends the user to `authorizeUrl`, with the query parameters
   `redirect_uri` and a random `state`.
2. When the user has signed in, the provider returns an authorization `code`
   and the same `state`, normally by redirecting to `redirect_uri`.
3. The client exchanges the code at `tokenUrl`:
   ```json
   { "grantType": "authorization_code", "code": "<code>", "redirectUri": "<redirect_uri>" }
   ```
   and receives:
   ```json
   { "registrationToken": "<token>", "refreshToken": "<token>", "expiresIn": 3600, "tokenType": "bearer" }
   ```
   `refreshToken` and `expiresIn` are optional.
4. If the provider has a `refreshUrl`, the client can get a new token there
   with `{ "grantType": "refresh_token", "refreshToken": "<token>" }`.

How a server checks a `registrationToken` is its own business. The reference
server can verify it as a JWT, and can also act as a simple provider itself
for development, at `/provider-auth/authorize`, `/provider-auth/token`, and
`/provider-auth/refresh`. Its `authorize` route answers with the JSON object
`{ "code", "state", "redirectUri" }` instead of redirecting.

### Submitting a Registration

```
POST /registration
Content-Type: application/json
```

```json
{
  "registrationData": {
    "did": "did:example:alice",
    "termsOfServiceHash": "<64 hex characters>"
  },
  "proofOfWork": {
    "challengeNonce": "<64 hex characters>",
    "responseNonce": "<64 hex characters>"
  }
}
```

or, with a provider's token in place of `proofOfWork`:

```json
{
  "registrationData": { "did": "did:example:alice", "termsOfServiceHash": "<64 hex characters>" },
  "providerAuth": { "registrationToken": "<token>" }
}
```

- If the request has `providerAuth.registrationToken`, the server uses it and
  ignores `proofOfWork`. A request with neither is refused.
- When the server has terms of service, the registration ****MUST**** carry
  their hash, whichever credential is used. A registration by proof of work
  without it is refused. A registration by provider token without it is
  recorded, but the tenant is not active (see [Terms of Service](#terms-of-service)).

| Response | Body |
|---|---|
| `200` | `{ "success": true }` |
| `400` | `{ "code": "<error code>", "message": "<error code>: <detail>" }` |
| `500` | `{ "success": false }` |

## Local Node Profile

A [[ref:Local Node]] is a server that runs on the user's own device, for the
applications on that device. It speaks the same protocol as any other server,
with two additions: it accepts connections only from the device itself, and
it requires every caller to present a token.

The second matters because any web page the user visits can try to reach a
port on `localhost`. A local node therefore gives a token to a web origin only
once the user has approved that origin.

A local node reports `localNode: true` in `GET /info`.

### Loopback Only

- A local node ****MUST**** listen on a loopback address only, and
  ****MUST**** refuse to start otherwise. A loopback host is `localhost`,
  `::1`, or an address in `127.0.0.0/8`.
- It ****MUST**** reject, with `403`, any request or WebSocket upgrade whose
  `Host` header does not name a loopback host. This defeats DNS rebinding, in
  which a hostile page makes its own domain resolve to `127.0.0.1`.

### Authentication

A token is 32 random bytes, base64url-encoded. A server ****MUST**** compare
tokens in constant time.

A caller presents its token as `Authorization: Bearer <token>`. A browser
cannot set that header when it opens a WebSocket, so on a WebSocket upgrade
the token may instead be the query parameter `localNodeToken`.

There are two kinds of token:

- **A token for a web origin**, obtained by [pairing](#pairing). It is valid
  only on requests whose `Origin` header is exactly the origin it was issued
  to.
- **A token for programs on the device**, such as command-line tools. The
  local node writes it to the [[ref:Discovery File]], which only the user can
  read. It is valid only on requests with *no* `Origin` header.

Every route requires a valid token, with these exceptions: `OPTIONS`
requests, `GET /info`, `POST /local/pair`, and `GET /local/pair/:requestId`.
A request without a valid token is answered with `401` and
`{"error":"Unauthorized"}`. WebSocket upgrades always require one.

CORS is narrowed to match. `Access-Control-Allow-Origin` is never `*`: it
repeats the request's `Origin`, with `Vary: Origin`, only for `/info`, for
`/local/pair`, for a pairing request that origin started, and for origins
that are already paired. `Access-Control-Allow-Headers` is
`authorization, content-type`.

### Pairing

Pairing is how a web application gets a token.

**1. Ask.** The application sends `POST /local/pair`. The body is ignored;
the server reads the `Origin` header, which the browser sets and a page
cannot forge.

```json
{ "requestId": "<uuid>", "status": "pending" }
```

A request without an `http:` or `https:` `Origin` is refused with `400`. A
second request from an origin that already has one pending returns the same
`requestId`. Beyond that, an origin may start at most 5 requests in 60
seconds; more are refused with `429` and `Retry-After`. A request expires
after 5 minutes.

**2. The user decides.** The local node asks the user, in its own interface,
whether to trust the origin. A local node may be configured with origins it
approves without asking.

**3. Poll.** The application polls `GET /local/pair/:requestId`, from the same
origin. A poll from any other origin is answered with `404`.

```json
{ "status": "approved", "origin": "https://app.example", "token": "<token>" }
```

`status` is `pending`, `approved`, `denied`, or `expired`. The response to
`POST /local/pair` always says `pending`, even for an origin approved without
asking, so the application always polls. The `token` is returned **once**, on
the first poll that finds the request approved; later polls omit it. A
request that has been decided can still be polled for 5 minutes.

**4. Check.** An application that already has a token can test it with
`GET /local/status`, which answers
`{ "localNode": true, "origin": "<origin>", "paired": true }`, or `401` if
the token is no longer valid.

A token for an origin stays valid until the user revokes it on the device.
There is no route for revoking one. When a token is revoked, the local node
****MUST**** close every WebSocket connection that was opened with it.

### Finding a Local Node

**The discovery file.** A local node writes `~/.enbox/dwn.json` when it
starts:

```json
{
  "endpoint": "http://127.0.0.1:55500",
  "pid": 12345,
  "capabilities": ["http", "ws"],
  "localNodeToken": "<token>"
}
```

| Property | Type | Meaning |
|---|---|---|
| `endpoint` | string | ****REQUIRED****. The URL the local node listens on |
| `pid` | number | ****REQUIRED****. Its process ID. A reader discards a file that is malformed, or whose process is no longer running |
| `capabilities` | string[] | The transports it offers: `http`, `ws` |
| `localNodeToken` | string | The token for programs on the device |

The file ****MUST**** be readable by the user only. The reference local node
listens on the first free port of `55500` to `55509`, then `3000`, and issues
a new `localNodeToken` each time it starts.

**Programs on the device** read the discovery file, and confirm what they
found by checking that `GET /info` at that endpoint names the server they
expect (the reference client expects `@enbox/dwn-server`). If a token stops
working, they read the file again.

**A web application** cannot read the file. At startup it ****MUST NOT****
go looking for a local node by trying ports, because that makes the browser
ask the user for permission to reach the local network. It may only check a
pairing it already has, with `GET /info` and `GET /local/status`. When the
user explicitly asks to connect to a local node, the application may try
`127.0.0.1` on the ports above.

**A desktop application** that runs a local node can register the `dwn://`
URL scheme. Opening `dwn://connect?callback=<encoded URL>` makes it redirect
the browser to
`<callback>#<base64url of {"endpoint":"http://127.0.0.1:<port>"}>`. The page
that receives this ****MUST**** accept the endpoint only if its host is
`127.0.0.1`, `::1`, `localhost`, or a name ending in `.localhost`.

### Local Callers and Synchronization

A sync client that runs beside a local node may ask it, when delivering a
message, whether it actually holds the record's data
(`includeMaterializationConfirmation`; see
[`dwn.applyReplicatedMessage`](#dwnapplyreplicatedmessage)). Only a caller
authenticated under this profile may ask.

The reference local node accepts record data up to 1 GiB, and has
[forwarding and delivery](#server-side-forwarding-and-record-delivery) turned
on.

## Limits

### Rate Limits

| Limit | Applies to | Reference default | When exceeded |
|---|---|---|---|
| Per IP address | HTTP requests, not WebSocket upgrades | 30 a second, bursts of 50 | HTTP `429`, `Retry-After`, `{"error":"Rate limit exceeded"}` |
| Per tenant | Messages for one tenant, on either transport | 20 a second, bursts of 50 | JSON-RPC `-50429` with `data.retryAfterSec`; over HTTP also status `429` and `Retry-After` |
| Per origin | `POST /local/pair` | 5 in 60 seconds | HTTP `429`, `Retry-After` |

### Sizes

- **A JSON-RPC request over HTTP:** at most 1 MiB, not counting record data.
- **Record data:** at most the server's `maxFileSize`. Data sent with
  `dwn.applyReplicatedMessage` is never read past `descriptor.dataSize`.
- **A WebSocket frame:** see [Frames](#frames).
- **Quota:** a server may also limit how much a tenant stores in total (see
  [Error Codes](#error-codes)).

## Security Considerations

- **Transport security.** A server reachable over a network ****SHOULD****
  use TLS (`https:` and `wss:`). A [[ref:Local Node]] may use `http:` and
  `ws:`, since its traffic never leaves the device.
- **The transport is not trusted.** A node verifies every message it is
  given: its signatures, its CIDs, and its authority. Nothing in this
  specification lets a caller skip that, and a server gains no authority over
  a tenant's data by carrying it.
- **Parsing.** The `dwn-response` header carries JSON from a server, and REST
  query parameters carry property paths from a caller. Both are untrusted
  input. A server ****MUST**** ignore query parameters whose path contains
  `__proto__`, `constructor`, or `prototype`.
- **Error detail.** A server ****SHOULD NOT**** reveal internal detail in
  error messages. A client ****SHOULD**** keep tokens out of its logs,
  including a `localNodeToken` in a WebSocket URL.
- **What an observer sees.** TLS hides the content of messages, but not
  which server a client talks to, when, or how much. Record-level encryption,
  defined in the DWN specification, protects data from the server itself.

## Reference Client Behavior

This section is not normative. It records what the reference client does, as
a starting point for other clients.

- **Timeouts.** An HTTP request times out after 30 seconds, or 300 seconds
  for a `dwn.applyReplicatedMessage` whose data is larger than 1 MiB. A
  WebSocket connection attempt times out after 3 seconds, and a request sent
  over WebSocket after 30.
- **Retries.** Over HTTP, network failures and statuses `408`, `429`, `500`,
  `502`, `503`, and `504` are retried up to 3 times, with jittered
  exponential backoff from 500 ms to 10 s. A `Retry-After` header is honored
  up to 10 s. A `500` that carries a JSON-RPC error the client treats as
  final (see [Error Codes](#error-codes)) is not retried, and neither is a
  request whose body is a stream that cannot be sent twice.
- **Choosing a transport.** Requests go over HTTP by default. When a
  WebSocket connection to the server is already open, a request that carries
  no data stream and whose JSON is at most 64 KiB is sent over it instead:
  queries, deletes, module operations, and replicated messages whose data is
  absent or small enough to send as `encodedData`. Reads of record data, and
  `RecordsWrite` messages sent with `dwn.processMessage`, always use HTTP.

## References

[DWN Specification](https://dwn-spec.pages.dev)
~ Messages, replies, subscriptions, and progress tokens.

[DWN Sync Protocol](https://dwn-sync-spec.pages.dev)
~ Replicated admission and its outcomes, and how a sync client uses this transport.

[Enbox Connect Protocol](https://web5-connect-spec.pages.dev)
~ The relay routes that a server may host under `/connect`.

[JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
~ JSON-RPC Working Group. March 2010.

[[spec:RFC2119]]
~ [Key words for use in RFCs to Indicate Requirement Levels](https://datatracker.ietf.org/doc/html/rfc2119). S. Bradner. March 1997.

[[spec:RFC4648]]
~ [The Base16, Base32, and Base64 Data Encodings](https://datatracker.ietf.org/doc/html/rfc4648). S. Josefsson. October 2006.

[[spec:RFC6455]]
~ [The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455). I. Fette, A. Melnikov. December 2011.

[[spec:RFC8259]]
~ [The JavaScript Object Notation (JSON) Data Interchange Format](https://datatracker.ietf.org/doc/html/rfc8259). T. Bray. December 2017.

[RFC 9110 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
~ R. Fielding, M. Nottingham, J. Reschke. June 2022.
