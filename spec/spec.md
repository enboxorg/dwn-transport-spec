
DWN Transport Protocol
==================

**Specification Status:** Draft

**Latest Draft:**
  TBD
<!-- -->

**Editors:**
~ [Liran Cohen](https://www.linkedin.com/in/itsliran/) (Enbox)

**Participate:**
~ [GitHub repo](https://github.com/enboxorg/dwn-transport-spec)
~ [File a bug](https://github.com/enboxorg/dwn-transport-spec/issues)
~ [Commit history](https://github.com/enboxorg/dwn-transport-spec/commits/main)

------------------------------------

## Abstract

The [Decentralized Web Node (DWN) specification](https://github.com/enboxorg/dwn-spec) defines message interfaces, authorization, encryption, and synchronization for personal data storage nodes — but intentionally omits how those messages are exchanged between parties over the network.

This specification fills that gap. It defines the transport protocols, wire formats, and server behaviors that clients, servers, wallets, and agents ****MUST**** implement to interoperate when communicating with [[ref:Decentralized Web Nodes]].

Specifically, this specification covers:

- **Service endpoint discovery** from DID documents
- **JSON-RPC 2.0 wire format** for encoding DWN messages over the network
- **HTTP transport** for request-response message processing
- **WebSocket transport** for real-time subscriptions and event streaming
- **REST convenience API** for simple, unauthenticated resource access
- **Server information and capability discovery**
- **Tenant registration** for multi-tenant DWN servers
- **Per-message payment** for usage-based DWN server monetization
- **Sync orchestration** between DWN nodes

## Status of This Document

DWN Transport Protocol is a _DRAFT_ specification under development by
[Enbox](https://enbox.org). It is a companion specification to the
[DWN specification](https://github.com/enboxorg/dwn-spec) and describes the
transport layer referenced in that specification's Technical Stack.

The specification will be updated to incorporate feedback. We encourage reviewers
to submit [GitHub Issues](https://github.com/enboxorg/dwn-transport-spec/issues)
as the means by which to communicate feedback and contributions.

## Terminology

[[def:Decentralized Web Node, DWN, Node]]
~ A decentralized personal and application data storage and message relay node,
as defined in the [DWN specification](https://github.com/enboxorg/dwn-spec).

[[def:DWN Server]]
~ A network-accessible service that hosts one or more [[ref:Decentralized Web Node]]
instances and exposes them via the HTTP and WebSocket transports defined in this
specification. A single DWN Server may serve multiple tenants.

[[def:DWN Client]]
~ Any software that sends DWN messages to a [[ref:DWN Server]] over the transports
defined in this specification. Clients include user agents, mobile applications,
other DWN Servers performing sync, and automated services.

[[def:Tenant]]
~ The DID that owns a [[ref:Decentralized Web Node]] instance. All messages sent
to a DWN Server are scoped to a target tenant.

[[def:Service Endpoint]]
~ A URL published in a DID document that identifies the network location of a
[[ref:DWN Server]] hosting a [[ref:Decentralized Web Node]] for that DID.

[[def:JSON-RPC Envelope]]
~ A JSON object conforming to the [JSON-RPC 2.0 specification](https://www.jsonrpc.org/specification)
that wraps a DWN message for transport over HTTP or WebSocket.

[[def:Subscription]]
~ A long-lived connection over WebSocket where the server pushes real-time events
to the client in response to a `RecordsSubscribe` or `MessagesSubscribe` message.
Events are delivered as `SubscriptionMessage` objects — a discriminated union of
event payloads (with cursors) and End-of-Stored-Events (EOSE) markers. See
[Subscription Events](#ws-subscription-events).

[[def:SubscriptionMessage]]
~ A discriminated union delivered over a [[ref:Subscription]]. Each message has a
`type` field: `"event"` for a regular event carrying a cursor and payload, or
`"eose"` for an End-of-Stored-Events marker signaling that catch-up replay is
complete.

[[def:Cursor]]
~ An opaque string assigned by the server's EventLog implementation to each event.
Clients persist cursors and pass them back when reconnecting to resume from the
last seen position. Cursor format is implementation-defined and ****MUST NOT**** be
parsed or constructed by clients.

[[def:Payment Scheme]]
~ A named protocol for fulfilling a payment obligation. Each scheme defines its own
`PaymentRequirements` extensions and `PaymentProof` format. Scheme identifiers use
URN format (`urn:enbox:payment:<scheme>:<version>`) for global uniqueness.

[[def:Payment Handler]]
~ A client-side component that can detect a supported [[ref:Payment Scheme]] in a
`PaymentRequirements` object and produce a corresponding `PaymentProof`.

## Service Endpoint Discovery {#service-endpoint-discovery}

### DID Document Service Entry {#did-document-service-entry}

A [[ref:DWN Client]] discovers the network location of a DID's [[ref:Decentralized Web Node]]
by resolving the DID and inspecting the DID document for a service entry of type
`DecentralizedWebNode`.

The service entry ****MUST**** conform to the following requirements:

- The `id` property ****MUST**** have a fragment value of `#dwn`.
- The `type` property ****MUST**** be `"DecentralizedWebNode"`.
- The `serviceEndpoint` property ****MUST**** be one of:
  - A `string` value containing a single HTTPS URL.
  - A map (JSON object) with a `url` property containing an HTTPS URL, and optional properties described below.
  - An `array` of `string` values and/or maps as described above. Multiple endpoints indicate nodes that synchronize to the same state.

When a `serviceEndpoint` entry is a map, the following properties are defined:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `url` | `string` | Yes | The HTTPS URL of the DWN endpoint. |
| `dataRetention` | `string` | No | `"cache"` when the node operates as a best-effort relay/cache. |

When `dataRetention` is absent, or when the entry is a bare `string`, the endpoint is a full node. A bare string `"https://dwn.example.com"` is equivalent to `{ "url": "https://dwn.example.com" }`.

::: example DID Document with DWN Service Entry
```json
{
  "id": "did:example:alice",
  "service": [{
    "id": "#dwn",
    "type": "DecentralizedWebNode",
    "serviceEndpoint": [
      "https://dwn.example.com",
      { "url": "https://relay.example.com", "dataRetention": "cache" }
    ]
  }]
}
```
:::

### Endpoint URL Requirements {#endpoint-url-requirements}

- [[ref:Service Endpoint]] URLs ****MUST**** use the `https` scheme.
- URLs ****MUST NOT**** include a trailing slash.
- URLs ****MUST NOT**** include query parameters or fragment identifiers.
- The URL represents the base path for all transport endpoints defined in this specification.

### Multi-Endpoint Resolution {#multi-endpoint-resolution}

When a DID document contains multiple [[ref:Service Endpoint]] URLs, a [[ref:DWN Client]]
****SHOULD**** implement failover by attempting alternative endpoints when a request fails.

Endpoints with no `dataRetention` property (including bare string entries) are full nodes and
****SHOULD**** be treated as equivalent replicas. The client ****MAY**** select any full
endpoint for a given request.

Endpoints with `"dataRetention": "cache"` are best-effort relays. A client ****MAY**** send
write operations to any endpoint (including cache endpoints). For read operations, clients
****SHOULD**** prefer full endpoints when available, as cache endpoints may not retain all
record data.

For write operations (`RecordsWrite`, `RecordsDelete`, `ProtocolsConfigure`), a client
****SHOULD**** send the message to at least one endpoint. The sync protocol ensures
that all replicas converge to the same state.

For sync operations, a client ****SHOULD**** choose a single endpoint to sync with.
Since all endpoints serve the same tenant's DWN, syncing with one endpoint is sufficient.

## JSON-RPC 2.0 Wire Format {#json-rpc-wire-format}

All DWN messages sent between [[ref:DWN Clients]] and [[ref:DWN Servers]] are wrapped
in [JSON-RPC 2.0](https://www.jsonrpc.org/specification) envelopes. This provides a
uniform request-response framing, error handling, and method dispatch.

### Request Object {#request-object}

A JSON-RPC request ****MUST**** conform to the following structure:

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
- `id` — ****MUST**** be a `string`, `number`, or `null`. Used to correlate requests
  with responses. Clients ****SHOULD**** use a UUID v4.
- `method` — ****MUST**** be one of the [registered methods](#methods).
- `params` — ****MUST**** be an object containing:
  - `target` — The DID of the [[ref:Tenant]] whose DWN should process the message.
  - `message` — The DWN message object as defined in the
    [DWN specification](https://github.com/enboxorg/dwn-spec).

For subscription methods, the request ****MUST**** also include:

- `subscription` — An object containing:
  - `id` — A client-assigned identifier for the subscription. ****MUST**** be a
    `string`, `number`, or `null`.

### Response Object {#response-object}

#### Success Response

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

- `id` — ****MUST**** match the `id` from the corresponding request.
- `result` — An object containing:
  - `reply` — The DWN reply object as defined in the
    [DWN specification](https://github.com/enboxorg/dwn-spec). The exact shape
    depends on the interface and method (see the DWN spec's
    [Response Objects](https://github.com/enboxorg/dwn-spec) section).

::: note
When the DWN reply includes a data stream (e.g., `RecordsRead`), the `entry.data`
field is **removed** from the JSON response and transmitted separately as described
in [HTTP Response Format](#http-response-format). This applies only to the HTTP transport.
:::

#### Error Response

```json
{
  "jsonrpc": "2.0",
  "id": "b5e5c5a0-3e2a-4c1d-8f7e-1a2b3c4d5e6f",
  "error": {
    "code": -32603,
    "message": "Internal error"
  }
}
```

- `error` — An object containing:
  - `code` — An integer error code. See [Error Codes](#error-codes).
  - `message` — A human-readable error description.
  - `data` — ****MAY**** be present with additional error context.

### Error Codes {#error-codes}

Implementations ****MUST**** use the following error codes:

| Code | Name | Description |
|---|---|---|
| `-32700` | Parse Error | Invalid JSON received by the server |
| `-32600` | Invalid Request | The JSON is not a valid JSON-RPC request |
| `-32601` | Method Not Found | The requested method does not exist |
| `-32602` | Invalid Params | Invalid method parameters |
| `-32603` | Internal Error | Internal server error |
| `-32300` | Transport Error | Transport-level failure (e.g., stream interrupted) |
| `-50400` | Bad Request | The DWN message was malformed or failed validation |
| `-50401` | Unauthorized | Authentication or authorization failed |
| `-50402` | Payment Required | The server requires payment to process this message. See [Per-Message Payment](#per-message-payment). |
| `-50403` | Forbidden | The operation is not permitted for the given tenant |
| `-50409` | Conflict | The message conflicts with existing state (e.g., duplicate) |

Error codes in the `-32xxx` range are standard JSON-RPC errors. Codes in the
`-50xxx` range are DWN-specific extensions that mirror HTTP status semantics.

### Methods {#methods}

| Method | Transport | Description |
|---|---|---|
| `dwn.processMessage` | HTTP, WebSocket | Process any DWN message |
| `rpc.subscribe.dwn.processMessage` | WebSocket only | Open a subscription for a DWN subscribe message |
| `rpc.subscribe.close` | WebSocket only | Close an active subscription |
| `rpc.ack` | WebSocket only | Acknowledge receipt of subscription events (flow control) |

#### `dwn.processMessage` {#method-process-message}

Processes a single DWN message against the target tenant's node. This is the primary
method for all DWN operations: writes, reads, queries, deletes, protocol configuration,
and sync.

**Parameters:**

- `target` — The DID of the tenant.
- `message` — The DWN message object.

**Result:**

- `reply` — The DWN reply object. The shape varies by interface and method.

#### `rpc.subscribe.dwn.processMessage` {#method-subscribe}

Opens a real-time subscription on a WebSocket connection. The server processes the
DWN subscribe message (e.g., `RecordsSubscribe` or `MessagesSubscribe`) and begins
pushing matching events to the client.

**Parameters:**

- `target` — The DID of the tenant.
- `message` — A DWN subscribe message (`RecordsSubscribe` or `MessagesSubscribe`).

**Subscription:**

- `id` — The client-assigned subscription identifier.

**Result:**

- `reply` — The DWN subscribe reply. If successful, subsequent events are pushed
  as individual JSON-RPC responses with the subscription `id` as the response `id`
  and `result.subscription` containing a [[ref:SubscriptionMessage]] — either an
  event payload with a [[ref:Cursor]], or an EOSE marker. See
  [Subscription Events](#ws-subscription-events).

#### `rpc.subscribe.close` {#method-subscribe-close}

Closes an active subscription on the current WebSocket connection.

**Parameters:** None.

**Subscription:**

- `id` — The identifier of the subscription to close.

**Result:**

- `reply` — `{ "status": { "code": 200, "detail": "OK" } }` on success.
- If the subscription ID is not found, the server ****MUST**** return a
  JSON-RPC error with code `-50400`.


## HTTP Transport {#http-transport}

The HTTP transport is used for request-response DWN message processing. It supports
all DWN interface methods, including data-bearing operations like `RecordsWrite` and
`RecordsRead`.

### Endpoint {#http-endpoint}

All DWN messages are sent to the base [[ref:Service Endpoint]] URL via HTTP `POST`:

```
POST https://dwn.example.com/
```

The server ****MUST**** accept `POST` requests at the root path (`/`).

### Request Format {#http-request-format}

DWN HTTP requests use a **header/body split** pattern that separates the JSON-RPC
envelope from binary data:

- The `dwn-request` HTTP header ****MUST**** contain the JSON-serialized
  [[ref:JSON-RPC Envelope]] (the full `JsonRpcRequest` object).
- The HTTP request body ****MAY**** contain the raw binary data associated with
  the message (e.g., record data for a `RecordsWrite`).

::: example RecordsWrite HTTP Request
```http
POST / HTTP/1.1
Host: dwn.example.com
Content-Type: application/octet-stream
dwn-request: {"jsonrpc":"2.0","id":"abc-123","method":"dwn.processMessage","params":{"target":"did:example:alice","message":{"descriptor":{"interface":"Records","method":"Write",...},"authorization":{...}}}}

<binary record data>
```
:::

::: example RecordsQuery HTTP Request (no body)
```http
POST / HTTP/1.1
Host: dwn.example.com
Content-Length: 0
dwn-request: {"jsonrpc":"2.0","id":"abc-456","method":"dwn.processMessage","params":{"target":"did:example:alice","message":{"descriptor":{"interface":"Records","method":"Query",...},"authorization":{...}}}}
```
:::

#### The `dwn-request` Header {#dwn-request-header}

- The `dwn-request` header ****MUST**** be present on every `POST` request to the
  DWN endpoint.
- The value ****MUST**** be a JSON-serialized `JsonRpcRequest` object.
- The server ****MUST**** return HTTP `400` if the header is missing or contains
  invalid JSON.

#### Request Body {#request-body}

- For messages that include data (e.g., `RecordsWrite`), the binary data stream
  ****MUST**** be sent as the HTTP request body.
- The `Content-Type` header ****SHOULD**** be `application/octet-stream` for the
  raw data body.
- For messages without data, the body ****MUST**** be empty and `Content-Length`
  ****SHOULD**** be `0`.
- The server detects the presence of a body via the `Content-Length` header
  (value > 0) or the presence of a `Transfer-Encoding` header.

### Response Format {#http-response-format}

The response format depends on whether the DWN reply includes a data stream.

#### Response with Data Stream {#response-with-data}

When the DWN reply includes a data stream (e.g., a `RecordsRead` reply with record data):

- The `dwn-response` HTTP header ****MUST**** contain the JSON-serialized
  `JsonRpcResponse` object, with the `entry.data` field **removed**.
- The HTTP response body ****MUST**** contain the raw binary data stream.
- The `Content-Type` header ****MUST**** be `application/octet-stream`.

::: example RecordsRead HTTP Response with Data
```http
HTTP/1.1 200 OK
Content-Type: application/octet-stream
dwn-response: {"jsonrpc":"2.0","id":"abc-789","result":{"reply":{"status":{"code":200,"detail":"OK"},"entry":{"recordsWrite":{"descriptor":{"interface":"Records","method":"Write","dataFormat":"image/png",...}}}}}}

<binary record data>
```
:::

#### Response without Data Stream {#response-without-data}

For all other replies (queries, writes, deletes, protocol operations, sync, etc.):

- The HTTP response body ****MUST**** contain the JSON-serialized `JsonRpcResponse` object.
- The `Content-Type` header ****MUST**** be `application/json`.

::: example RecordsQuery HTTP Response
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"jsonrpc":"2.0","id":"abc-456","result":{"reply":{"status":{"code":200,"detail":"OK"},"entries":[...],"cursor":"..."}}}
```
:::

#### Error Responses {#http-error-responses}

- If the `dwn-request` header is missing: HTTP `400` with a JSON-RPC error
  response body (code `-50400`).
- If the `dwn-request` header contains invalid JSON: HTTP `400` with a JSON-RPC
  error response body (code `-32700`).
- If the DWN message handler throws an unexpected error: HTTP `500` with the
  JSON-RPC response containing the error.
- All other DWN-level errors (401, 404, 409, etc.) are returned as successful HTTP
  responses (HTTP `200`) with the DWN status code in the `reply.status.code` field
  inside the JSON-RPC result. The HTTP status code reflects transport-level success;
  the DWN status code reflects application-level outcome.

::: note
This design means that an HTTP `200` does not necessarily indicate a successful DWN
operation. Clients ****MUST**** inspect `result.reply.status.code` in the JSON-RPC
response to determine the outcome. HTTP-level error codes (4xx, 5xx) indicate
transport or envelope failures only.
:::

### CORS {#cors}

Servers ****MUST**** include the following CORS headers on every HTTP response:

| Header | Value |
|---|---|
| `Access-Control-Allow-Origin` | `*` |
| `Access-Control-Allow-Methods` | `GET, POST, OPTIONS` |
| `Access-Control-Allow-Headers` | `*` |
| `Access-Control-Expose-Headers` | `dwn-response, dwn-payment` |

Servers ****MUST**** respond to `OPTIONS` preflight requests with HTTP `204 No Content`
and the above headers.

::: note
The `dwn-response` and `dwn-payment` headers must be listed in
`Access-Control-Expose-Headers` to ensure browser-based clients can read them.
Without this, the headers are opaque to JavaScript running in a browser context.
:::

### Method Restrictions {#http-method-restrictions}

All DWN interface methods are supported over HTTP. The HTTP transport is the
****only**** transport that supports `RecordsWrite` messages with data, because
the binary data is carried in the HTTP request body.

## WebSocket Transport {#websocket-transport}

The WebSocket transport is used for real-time subscriptions and lightweight
request-response messaging. It provides persistent, bidirectional communication
between a [[ref:DWN Client]] and a [[ref:DWN Server]].

### Connection Establishment {#ws-connection}

A client establishes a WebSocket connection by sending an HTTP `Upgrade` request
to the same base [[ref:Service Endpoint]] URL used for HTTP:

```
GET wss://dwn.example.com/ HTTP/1.1
Upgrade: websocket
Connection: Upgrade
```

- The server ****MUST**** accept WebSocket upgrade requests at the root path.
- The WebSocket connection ****MUST**** use TLS (`wss://`).
- If the upgrade fails, the server ****MUST**** return HTTP `400`.

### Message Framing {#ws-framing}

All WebSocket messages are JSON-encoded **text frames**. Binary frames are not used.

- **Client → Server:** JSON-serialized `JsonRpcRequest` objects.
- **Server → Client:** JSON-serialized `JsonRpcResponse` objects.

Each text frame contains exactly one JSON-RPC message. Batching multiple messages
in a single frame is not supported.

### Request-Response Correlation {#ws-correlation}

The `id` field in the JSON-RPC request and response ****MUST**** be used to correlate
requests with responses. Since a WebSocket connection may carry multiple concurrent
requests, clients ****MUST**** use unique `id` values per connection and match
incoming responses by `id`.

::: example WebSocket Request-Response
```
Client → {"jsonrpc":"2.0","id":"q1","method":"dwn.processMessage","params":{"target":"did:example:alice","message":{...}}}
Server → {"jsonrpc":"2.0","id":"q1","result":{"reply":{"status":{"code":200},"entries":[...]}}}
```
:::

### Subscriptions {#ws-subscriptions}

WebSocket is the ****only**** transport that supports DWN subscriptions
(`RecordsSubscribe` and `MessagesSubscribe`).

#### Opening a Subscription {#ws-open-subscription}

To open a subscription, the client sends a `rpc.subscribe.dwn.processMessage`
request with a `subscription` object containing a client-assigned `id`:

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
        "filter": { "protocol": "https://social.example/protocol" }
      },
      "authorization": { ... }
    }
  },
  "subscription": { "id": "sub-1" }
}
```

The server processes the subscribe message on the DWN. If successful, the server
returns a success response and begins pushing events for the subscription.

To resume a subscription after a disconnect (cursor mode), the client includes a
`cursor` field in the descriptor with a [[ref:Cursor]] value obtained from a prior
subscription event:

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
        "filter": { "protocol": "https://social.example/protocol" },
        "cursor": "<opaque-cursor-string>"
      },
      "authorization": { ... }
    }
  },
  "subscription": { "id": "sub-2" }
}
```

In cursor mode, the server replays stored events after the cursor, sends an
[EOSE marker](#ws-eose), then continues with live events.

#### Subscription Events {#ws-subscription-events}

After a successful subscribe response, the server pushes
[[ref:SubscriptionMessage]] objects as JSON-RPC responses where:

- `id` is the **subscription** `id` (not the original request `id`).
- `result.subscription` contains a [[ref:SubscriptionMessage]] — a discriminated
  union with a `type` field of `"event"` or `"eose"`.

##### Event Messages {#ws-event-messages}

An event message carries the DWN event payload and an opaque [[ref:Cursor]]
string identifying the event's position in the EventLog.

```json
{
  "jsonrpc": "2.0",
  "id": "sub-1",
  "result": {
    "subscription": {
      "type": "event",
      "cursor": "<opaque-cursor-string>",
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
| `type` | `"event"` | Discriminator for an event message |
| `cursor` | `string` | Opaque EventLog cursor. Used for [flow control](#ws-flow-control) acknowledgment and for reconnection. Clients ****SHOULD**** persist this value. |
| `event` | `object` | The event payload containing `message` (the DWN message) and optionally `initialWrite` (the initial `RecordsWrite` for update/delete events). |

##### EOSE (End-of-Stored-Events) {#ws-eose}

When a subscription is created with a `cursor` (cursor mode), the server replays
all stored events after that cursor position. After replay is complete, the server
sends an EOSE marker to signal that all subsequent messages are live events.

Subscriptions created without a cursor (snapshot mode) never receive EOSE.

```json
{
  "jsonrpc": "2.0",
  "id": "sub-1",
  "result": {
    "subscription": {
      "type": "eose",
      "cursor": "<opaque-cursor-string>"
    }
  }
}
```

| Field | Type | Description |
|---|---|---|
| `type` | `"eose"` | Discriminator for the End-of-Stored-Events marker |
| `cursor` | `string` | Opaque cursor of the last replayed stored event. If no stored events matched (i.e. the subscriber was already caught up), this echoes the input cursor. |

##### Subscription Modes {#ws-subscription-modes}

Subscriptions operate in one of two modes depending on the DWN subscribe message:

- **Snapshot mode** (no `cursor` in the subscribe descriptor): The DWN returns an
  initial set of matching records in the subscribe reply's `entries` field, then
  delivers only live events through the subscription. No EOSE is sent.

- **Cursor mode** (`cursor` present in the subscribe descriptor): The DWN replays
  stored events from the EventLog starting after the cursor, sends an EOSE marker,
  then continues with live events. No `entries` are returned in the subscribe reply.

Cursor mode enables clients to reconnect after a disconnect without missing events.
See the [DWN specification](https://github.com/enboxorg/dwn-spec) for full details
on subscription modes and EventLog semantics.

##### Deduplication {#ws-deduplication}

Clients ****SHOULD**** deduplicate events by `messageCid`, as the DWN specification
notes that duplicate events may be received. This is especially relevant during the
catch-up → live transition in cursor mode, where the EventLog implementation buffers
live events during replay.

#### Closing a Subscription {#ws-close-subscription}

To close a subscription, the client sends a `rpc.subscribe.close` request:

```json
{
  "jsonrpc": "2.0",
  "id": "req-2",
  "method": "rpc.subscribe.close",
  "subscription": { "id": "sub-1" }
}
```

The server ****MUST**** stop sending events for the closed subscription and release
associated resources. All active subscriptions ****MUST**** be automatically closed
when the WebSocket connection is closed.

#### Flow Control (Backpressure) {#ws-flow-control}

To prevent a fast-producing server from overwhelming a slow-consuming client,
the WebSocket transport implements a **sliding window** flow control mechanism
using the `rpc.ack` method.

##### Server Behavior

The server maintains a per-subscription window of **unacknowledged events**. The
maximum window size is configured by the server and advertised as `maxInFlight`
in the [ServerInfo](#server-info-object) response.

- When the server has an event to deliver, it checks the window:
  - If the number of unacknowledged events is **less than** `maxInFlight`, the
    event is sent immediately.
  - If the window is **full**, the event is buffered.
- When the server receives an `rpc.ack` for a subscription, it acknowledges all
  events up to and including the specified cursor (cumulative acknowledgment) and
  flushes buffered events into the freed window slots.
- If the buffer exceeds a server-defined maximum (****RECOMMENDED**** 1000), the
  server ****MUST**** close the subscription to prevent unbounded memory growth.

::: note
Both `"event"` and `"eose"` messages count toward the `maxInFlight` window.
Clients ****MUST**** acknowledge EOSE messages the same way as event messages.
:::

##### `rpc.ack` Method {#method-rpc-ack}

The client sends `rpc.ack` to acknowledge receipt and processing of subscription
events. This is a JSON-RPC **notification** (no `id` field is required, and no
response is expected on success).

```json
{
  "jsonrpc": "2.0",
  "method": "rpc.ack",
  "params": { "cursor": "abc123" },
  "subscription": { "id": "sub-1" }
}
```

**Parameters:**

- `cursor` — The cursor from the most recently processed [[ref:SubscriptionMessage]].
  ****MUST**** be a non-empty string.

**Subscription:**

- `id` — The identifier of the subscription being acknowledged.

**Behavior:**

- The acknowledgment is **cumulative**: acking cursor X acknowledges all events
  with cursors up to and including X.
- Clients ****SHOULD**** send `rpc.ack` after processing each event or batch of events.
- If the cursor is unknown (e.g., stale or duplicate), the server ****SHOULD****
  ignore the acknowledgment silently.

##### Client Implementation

Clients ****MUST**** send `rpc.ack` after processing each subscription message to
advance the server's flow-control window. A client that does not send acknowledgments
will stop receiving events once the server's window fills up, and will eventually
have its subscription closed if the buffer overflows.

A ****RECOMMENDED**** client implementation sends `rpc.ack` immediately after the
subscription message handler returns:

```
receive SubscriptionMessage → invoke handler → send rpc.ack(cursor)
```

### Heartbeat {#ws-heartbeat}

The server ****MUST**** send WebSocket `ping` frames at a regular interval
(****RECOMMENDED**** every 30 seconds) to detect stale connections.

If a client does not respond with a `pong` frame before the next ping interval,
the server ****SHOULD**** close the connection.

Clients ****MUST**** respond to `ping` frames with `pong` frames as defined by
the [WebSocket protocol (RFC 6455)](https://datatracker.ietf.org/doc/html/rfc6455).

### Method Restrictions {#ws-method-restrictions}

- `RecordsWrite` messages with data ****MUST NOT**** be sent over WebSocket.
  Clients ****MUST**** use the [HTTP transport](#http-transport) for `RecordsWrite`
  operations that include record data.
- Subscribe methods (`rpc.subscribe.dwn.processMessage`) are ****only**** supported
  over WebSocket. Clients ****MUST NOT**** send subscribe requests over HTTP.
- All other `dwn.processMessage` requests (queries, reads without data, deletes,
  protocol operations, sync) ****MAY**** be sent over either HTTP or WebSocket.

| DWN Method | HTTP | WebSocket |
|---|---|---|
| `RecordsWrite` (with data) | Yes | No |
| `RecordsRead` (response has data) | Yes | No |
| `RecordsQuery` | Yes | Yes |
| `RecordsSubscribe` | No | Yes |
| `RecordsDelete` | Yes | Yes |
| `RecordsCount` | Yes | Yes |
| `ProtocolsConfigure` | Yes | Yes |
| `ProtocolsQuery` | Yes | Yes |
| `MessagesRead` | Yes | No |
| `MessagesSubscribe` | No | Yes |
| `MessagesSync` | Yes | Yes |


## REST Convenience API {#rest-api}

In addition to the JSON-RPC endpoint, [[ref:DWN Servers]] ****MUST**** expose a set
of read-only REST endpoints for simple, unauthenticated resource access. These
endpoints enable web browsers, `<img>` tags, `<video>` tags, curl, and other HTTP
clients to access published DWN data without constructing DWN messages.

All REST convenience routes create anonymous (unsigned) DWN messages internally.
They can only access data that is publicly readable (i.e., the protocol's `$actions`
grants `read` to `anyone`, or the record is published).

### Record Read {#rest-record-read}

```
GET /:did/read/records/:recordId
GET /:did/records/:recordId
```

Reads a single record by its Record ID.

**Path Parameters:**
- `:did` — The DID of the [[ref:Tenant]].
- `:recordId` — The Record ID of the record to read.

**Response:**
- **200 with data:** The response body is the raw record data. The `Content-Type`
  header ****MUST**** be set to the record's `dataFormat` value (e.g., `image/png`,
  `application/json`). The `dwn-response` header contains the full DWN `RecordsRead`
  reply as JSON.
- **404:** The record does not exist or is not publicly readable. Servers ****MUST****
  return `404` (not `401`) for unauthorized reads to prevent information disclosure
  about record existence.

### Protocol Record Read {#rest-protocol-record-read}

```
GET /:did/read/protocols/:protocol/*protocolPath
```

Reads the most recently published record at a given protocol path.

**Path Parameters:**
- `:did` — The DID of the [[ref:Tenant]].
- `:protocol` — The protocol URI, encoded as base64url.
- `*protocolPath` — The protocol path (e.g., `post/comment`).

**Query Parameters:**

Additional filter properties ****MAY**** be passed as dot-notation query parameters.
For example:

```
GET /did:example:alice/read/protocols/aHR0cHM6Ly9leGFtcGxl/post?filter.schema=https://example.com/post&filter.dataFormat=application/json
```

**Behavior:**

The server queries for records matching the protocol and protocol path with
`pagination.limit` of `1` and `dateSort` of `publishedDescending` (most recent
first). If a matching record is found, the server performs a `RecordsRead` and
returns the result.

**Response:** Same as [Record Read](#rest-record-read).

### Protocol Definition Read {#rest-protocol-definition-read}

```
GET /:did/read/protocols/:protocol
```

Reads the protocol definition for a given protocol URI.

**Path Parameters:**
- `:did` — The DID of the [[ref:Tenant]].
- `:protocol` — The protocol URI, encoded as base64url.

**Response:**
- **200:** The response body is the JSON-serialized protocol definition object.
  `Content-Type` is `application/json`.
- **404:** The protocol is not installed or not publicly visible.

### Protocol Query {#rest-protocol-query}

```
GET /:did/query/protocols
```

Lists all protocols installed on the tenant's DWN.

**Path Parameters:**
- `:did` — The DID of the [[ref:Tenant]].

**Response:**
- **200:** The response body is a JSON array of protocol definition entries.
  `Content-Type` is `application/json`.

### Record Query {#rest-record-query}

```
GET /:did/query
```

Queries records with optional filters, pagination, and sorting.

**Path Parameters:**
- `:did` — The DID of the [[ref:Tenant]].

**Query Parameters:**

Filter and pagination options are passed as dot-notation query parameters:

| Parameter | Description |
|---|---|
| `filter.protocol` | Protocol URI to filter by |
| `filter.protocolPath` | Protocol path to filter by |
| `filter.schema` | Schema URI to filter by |
| `filter.dataFormat` | MIME type to filter by |
| `filter.recordId` | Specific record ID |
| `pagination.limit` | Maximum number of results |
| `pagination.cursor` | Pagination cursor from a previous response |
| `dateSort` | Sort order (`createdAscending`, `createdDescending`, `publishedAscending`, `publishedDescending`) |

::: example Record Query
```
GET /did:example:alice/query?filter.protocol=https://social.example&filter.protocolPath=post&pagination.limit=10&dateSort=publishedDescending
```
:::

**Response:**
- **200:** The response body is a JSON-serialized DWN `RecordsQueryReply` object
  containing `status`, `entries`, and optionally `cursor` for pagination.
  `Content-Type` is `application/json`.


## Server Information {#server-information}

### Info Endpoint {#info-endpoint}

A [[ref:DWN Server]] ****MUST**** expose an information endpoint that describes the
server's capabilities and requirements:

```
GET /info
```

**Response:**

```json
{
  "server": "@enbox/dwn-server",
  "version": "1.0.0",
  "sdkVersion": "0.0.7",
  "url": "https://dwn.example.com",
  "webSocketSupport": true,
  "maxFileSize": 1073741824,
  "maxInFlight": 32,
  "registrationRequirements": ["proof-of-work-sha256-v0", "terms-of-service"],
  "providerAuth": {
    "authorizeUrl": "https://provider.example.com/authorize",
    "tokenUrl": "https://provider.example.com/token",
    "refreshUrl": "https://provider.example.com/refresh",
    "managementUrl": "https://provider.example.com/account"
  },
  "payment": {
    "methods": ["per-message-payment-v0"],
    "schemes": [
      "urn:enbox:payment:l402:v0",
      "urn:enbox:payment:cashu:v0"
    ],
    "currency": "sat",
    "infoUrl": "https://provider.example.com/pricing"
  }
}
```

::: note
The `providerAuth` object is only present when `provider-auth-v0` is included in
`registrationRequirements`. Clients ****MUST NOT**** assume its presence unless the
registration requirement is advertised.
:::

::: note
The `payment` object is only present when the server supports per-message payment.
Clients ****MUST NOT**** assume its presence. Payment is orthogonal to registration —
a server ****MAY**** require registration AND support per-message payment, or support
payment without registration, or neither.
:::

#### ServerInfo Object {#server-info-object}

| Property | Type | Required | Description |
|---|---|---|---|
| `server` | `string` | Yes | Server implementation identifier |
| `version` | `string` | Yes | Server version (semver) |
| `sdkVersion` | `string` | Yes | DWN SDK version supported |
| `url` | `string` | Yes | The canonical base URL of the server |
| `webSocketSupport` | `boolean` | Yes | Whether the server supports WebSocket connections |
| `maxFileSize` | `number` | Yes | Maximum data size in bytes for a single `RecordsWrite` |
| `maxInFlight` | `number` | No | Maximum number of unacknowledged subscription events per subscription before the server pauses delivery. Defaults to `32` if not present. See [Flow Control](#ws-flow-control). |
| `registrationRequirements` | `string[]` | Yes | List of registration requirements (empty array if open) |
| `providerAuth` | `object` | Conditional | Provider auth configuration. ****MUST**** be present when `provider-auth-v0` is listed in `registrationRequirements`. |
| `providerAuth.authorizeUrl` | `string` | Yes | URL to redirect the user for authentication, signup, or payment. ****MAY**** be on the DWN server domain or an external domain. |
| `providerAuth.tokenUrl` | `string` | Yes | URL where the client exchanges an authorization code for a registration token. |
| `providerAuth.refreshUrl` | `string` | No | URL to refresh an expired registration token. If absent, tokens do not support refresh. |
| `providerAuth.managementUrl` | `string` | No | URL for user-facing account management. If absent, no management UI is available. |
| `payment` | `object` | No | Payment capabilities. ****MUST**** be present when the server supports [Per-Message Payment](#per-message-payment). |
| `payment.methods` | `string[]` | Yes | Payment flow methods supported. See [Payment Methods](#payment-methods). |
| `payment.schemes` | `string[]` | Yes | [[ref:Payment Scheme]] URN identifiers supported. See [Payment Scheme Registry](#payment-scheme-registry). |
| `payment.currency` | `string` | Yes | Base currency unit for pricing (e.g., `"sat"`, `"usd-cents"`). |
| `payment.infoUrl` | `string` | No | URL to a human-readable pricing page. |

#### Registration Requirement Values {#registration-requirement-values}

| Value | Description |
|---|---|
| `proof-of-work-sha256-v0` | Client must complete a SHA-256 proof-of-work challenge |
| `terms-of-service` | Client must accept the server's terms of service |
| `provider-auth-v0` | The server requires authentication with the DWN provider's auth service. When present, the `providerAuth` object ****MUST**** be included in the Server Information response. |

### Health Endpoint {#health-endpoint}

Servers ****SHOULD**** expose a health check endpoint:

```
GET /health
```

**Response:** `200 OK` with body `{"ok": true}` and `Content-Type: application/json`.

## Tenant Registration {#tenant-registration}

Multi-tenant [[ref:DWN Servers]] ****MAY**** require tenant registration before accepting
DWN messages for a given DID. Registration prevents abuse by requiring proof of
computational work, acceptance of terms of service, and/or authentication with the
DWN provider's auth service.

A server that requires registration ****MUST**** advertise its requirements in the
[ServerInfo](#server-info-object) `registrationRequirements` array. A server with
an empty `registrationRequirements` array is open and does not require registration.

### Registration Flow {#registration-flow}

1. The client fetches the [ServerInfo](#info-endpoint) to discover registration requirements.
2. If `proof-of-work-sha256-v0` is required, the client fetches a proof-of-work challenge.
3. If `terms-of-service` is required, the client fetches the terms of service text.
4. If `provider-auth-v0` is required, the client initiates the
   [Provider Authentication](#provider-auth-registration) flow.
5. The client computes the proof-of-work response and/or terms-of-service hash.
6. The client submits the registration request.

### Proof-of-Work Challenge {#pow-challenge}

```
GET /registration/proof-of-work
```

**Response:** A JSON object describing the challenge:

```json
{
  "challengeNonce": "a1b2c3d4e5f6...",
  "difficulty": 25,
  "timeToLiveSeconds": 120
}
```

| Property | Type | Description |
|---|---|---|
| `challengeNonce` | `string` | A server-generated random nonce |
| `difficulty` | `number` | Number of leading zero bits required in the hash |
| `timeToLiveSeconds` | `number` | How long the challenge is valid |

#### Solving the Challenge {#solving-challenge}

The client ****MUST**** find a `responseNonce` such that:

```
SHA-256(challengeNonce + responseNonce)
```

produces a hash with at least `difficulty` leading zero bits. The `responseNonce`
is a random string the client generates and iterates until the hash meets the
difficulty threshold.

### Terms of Service {#terms-of-service}

```
GET /registration/terms-of-service
```

**Response:** The terms of service as plain text (`Content-Type: text/plain`).

The client computes a SHA-256 hash of the terms text (hex-encoded) to include in
the registration request, indicating acceptance.

### Provider Authentication Registration {#provider-auth-registration}

When `provider-auth-v0` is listed in `registrationRequirements`, the server supports
an OAuth2-inspired authentication flow where the DWN provider authenticates and
authorizes the user through its own auth service. This enables paid DWN service
providers to onboard tenants via login, payment, or plan selection flows.

#### Flow Overview {#provider-auth-flow}

1. The client fetches [ServerInfo](#info-endpoint) and discovers `provider-auth-v0`
   in `registrationRequirements`, along with the `providerAuth` configuration object.
2. The client constructs the authorize URL by appending query parameters to
   `providerAuth.authorizeUrl`:
   - `redirect_uri` — Where the provider redirects after authentication. ****MUST****
     be an HTTPS URL (for browser-based wallets) or a native deep link / universal
     link (for mobile apps).
   - `state` — A CSRF protection nonce. ****MUST**** be cryptographically random.
   - `client_id` — ****MAY**** be included as a client or wallet identifier.
3. The client opens the authorize URL (via browser redirect, WebView, or system browser).
4. The user authenticates with the provider. The authentication experience is
   provider-defined and ****MAY**** include login, account creation, payment, or
   plan selection.
5. On success, the provider redirects back to `redirect_uri` with query parameters:
   - `code` — An authorization code.
   - `state` — The same `state` value from step 2.

   On failure, the provider redirects back with:
   - `error` — An error code.
   - `error_description` — A human-readable error message.
   - `state` — The same `state` value from step 2.

6. The client validates that the returned `state` matches the value sent in step 2.
   If the values do not match, the client ****MUST**** abort the flow.
7. The client exchanges the authorization code for a registration token via
   `POST {providerAuth.tokenUrl}`.
8. The client registers DIDs via `POST /registration` using the registration token.

#### Token Exchange Endpoint {#provider-auth-token-exchange}

```
POST {providerAuth.tokenUrl}
Content-Type: application/json
```

**Request Body:**

```json
{
  "grantType": "authorization_code",
  "code": "<authorization_code>",
  "redirectUri": "<redirect_uri_from_step_2>"
}
```

| Field | Required | Description |
|---|---|---|
| `grantType` | Yes | ****MUST**** be `"authorization_code"`. |
| `code` | Yes | The authorization code received from the provider redirect. |
| `redirectUri` | Yes | The `redirect_uri` used in the authorize request. ****MUST**** match exactly. |

**Response Body:**

```json
{
  "registrationToken": "<opaque_token>",
  "refreshToken": "<optional_refresh_token>",
  "expiresIn": 3600,
  "tokenType": "bearer"
}
```

| Field | Required | Description |
|---|---|---|
| `registrationToken` | Yes | An opaque token used in subsequent `POST /registration` requests. |
| `refreshToken` | No | A token that ****MAY**** be used to obtain a new registration token after expiration. Only present if `providerAuth.refreshUrl` is advertised. |
| `expiresIn` | Yes | Token lifetime in seconds. |
| `tokenType` | Yes | ****MUST**** be `"bearer"`. |

**Error Response:**

- **400:** `{ "error": "<error_code>", "errorDescription": "<message>" }` — The
  authorization code is invalid, expired, or the `redirectUri` does not match.

#### Token Refresh Endpoint {#provider-auth-token-refresh}

If the server advertises `providerAuth.refreshUrl`, the client ****MAY**** refresh
an expired registration token.

```
POST {providerAuth.refreshUrl}
Content-Type: application/json
```

**Request Body:**

```json
{
  "grantType": "refresh_token",
  "refreshToken": "<refresh_token>"
}
```

| Field | Required | Description |
|---|---|---|
| `grantType` | Yes | ****MUST**** be `"refresh_token"`. |
| `refreshToken` | Yes | The refresh token from a previous token exchange or refresh response. |

**Response Body:** Same shape as the [Token Exchange](#provider-auth-token-exchange) response.

**Error Response:**

- **400:** `{ "error": "<error_code>", "errorDescription": "<message>" }` — The
  refresh token is invalid or expired.

#### Design Notes {#provider-auth-design-notes}

- **Opaque tokens:** Registration tokens are intentionally opaque to the client.
  The server ****MAY**** implement them as JWTs, blind RSA tokens, ecash tokens,
  or any other format. Clients ****MUST NOT**** parse or interpret the token contents.

- **Privacy preservation:** Nothing in this protocol requires the provider to
  correlate DIDs with user accounts. A privacy-preserving provider ****MAY**** issue
  blinded tokens that cannot be linked back to the authentication session.

- **Reusable tokens:** A single registration token ****MAY**** be used to register
  multiple DIDs, subject to provider-defined limits. The server ****SHOULD****
  document any per-token DID registration limits in its terms of service.

- **Browser and native support:** The `redirect_uri` parameter supports both HTTPS
  URLs (for browser-based wallets) and native deep links or universal links (for
  mobile and desktop apps). Clients ****SHOULD**** use platform-appropriate redirect
  mechanisms (e.g., custom URL schemes on mobile, loopback URLs for desktop apps).

### Registration Submission {#registration-submission}

```
POST /registration
Content-Type: application/json
```

**Request Body:**

```json
{
  "registrationData": {
    "did": "did:example:alice",
    "termsOfServiceHash": "e3b0c44298fc1c149afbf4c8996fb924..."
  },
  "proofOfWork": {
    "challengeNonce": "a1b2c3d4e5f6...",
    "responseNonce": "x7y8z9..."
  },
  "providerAuth": {
    "registrationToken": "..."
  }
}
```

| Field | Required | Description |
|---|---|---|
| `registrationData.did` | Yes | The DID to register as a tenant |
| `registrationData.termsOfServiceHash` | Conditional | SHA-256 hex hash of the ToS text. Required if `terms-of-service` is in the registration requirements. ****MAY**** be omitted when using `provider-auth-v0` if the provider handles ToS acceptance in the auth flow. |
| `proofOfWork` | Conditional | Proof-of-work solution. Required if `proof-of-work-sha256-v0` is in the registration requirements. |
| `proofOfWork.challengeNonce` | Conditional | The challenge nonce from the server. |
| `proofOfWork.responseNonce` | Conditional | The client's solution nonce. |
| `providerAuth` | Conditional | Provider authentication credentials. Required if `provider-auth-v0` is in the registration requirements. |
| `providerAuth.registrationToken` | Conditional | The registration token obtained from the [Token Exchange](#provider-auth-token-exchange) endpoint. |

A registration request ****MUST**** include `proofOfWork`, `providerAuth`, or both,
depending on the server's `registrationRequirements`. If the server lists both
`proof-of-work-sha256-v0` and `provider-auth-v0`, the client ****MUST**** satisfy
both requirements unless the server explicitly documents that either is sufficient.

**Response:**
- **200:** `{ "success": true }` — Registration succeeded.
- **400:** `{ "code": "...", "message": "..." }` — Validation failure (invalid
  proof-of-work, wrong terms hash, expired challenge, invalid registration token, etc.).


## Per-Message Payment {#per-message-payment}

A [[ref:DWN Server]] ****MAY**** require payment for processing individual DWN messages.
This enables usage-based monetization where providers charge per operation — for
writes, reads, queries, or any combination thereof — using the payment method(s) of
their choice.

Per-message payment is ****OPTIONAL**** for both servers and clients. Servers that do
not enable payment never return `-50402` errors and behave identically to servers
without payment support. Clients that do not implement payment handling still work
against free servers and receive typed errors from paid servers that they can surface
to users or handle at the application layer.

Per-message payment is orthogonal to [Tenant Registration](#tenant-registration).
A server ****MAY**** require registration, support per-message payment, both, or neither.

### Payment Flow {#payment-flow}

The per-message payment flow uses a challenge/response pattern within the existing
JSON-RPC transport:

```
Client                              DWN Server
  │                                      │
  │  POST / (dwn-request header)         │
  │─────────────────────────────────────>│
  │                                      │  Server evaluates pricing policy:
  │                                      │  this message requires payment
  │  JSON-RPC error response             │
  │  code: -50402 (Payment Required)     │
  │  data: { paymentId,                  │
  │          paymentRequirements: [...] } │
  │<─────────────────────────────────────│
  │                                      │
  │  Client selects a payment scheme     │
  │  Client fulfills the payment         │
  │                                      │
  │  POST / (dwn-request header          │
  │        + dwn-payment header)         │
  │─────────────────────────────────────>│
  │                                      │  Server verifies payment proof
  │                                      │  Server processes DWN message
  │                                      │  Server settles payment
  │  JSON-RPC success response           │
  │<─────────────────────────────────────│
```

1. The client sends a normal `dwn.processMessage` request.
2. The server evaluates its pricing policy. If the message requires payment and no
   valid payment proof is attached, the server returns a JSON-RPC error with code
   `-50402` and a `data` field containing `PaymentRequiredData`.
3. The client inspects the `paymentRequirements` array, selects a
   [[ref:Payment Scheme]] it supports, and fulfills the payment using its
   [[ref:Payment Handler]] for that scheme.
4. The client retries the **same** request with the original `dwn-request` header
   and an additional `dwn-payment` header containing the `PaymentProof`.
5. The server verifies the payment proof, processes the DWN message, settles the
   payment, and returns the normal JSON-RPC response.

If the message does not require payment (e.g., the pricing policy assigns a zero
price, or the server does not have payment enabled), the server ****MUST**** process
the message immediately without returning `-50402`.

### The `-50402` Error Response {#payment-required-error}

When a server requires payment for a message, it ****MUST**** return a JSON-RPC
error response with code `-50402`. The `data` field ****MUST**** contain a
`PaymentRequiredData` object:

```json
{
  "jsonrpc": "2.0",
  "id": "abc-123",
  "error": {
    "code": -50402,
    "message": "Payment required",
    "data": {
      "paymentId": "pay_7f3a8b2c-1d4e-5f6a-9b8c-7d6e5f4a3b2c",
      "paymentRequirements": [
        {
          "scheme": "urn:enbox:payment:l402:v0",
          "amount": "100",
          "description": "RecordsWrite (1.2 MB)",
          "invoice": "lnbc1000n1pj9...",
          "macaroon": "0201036c6e6402..."
        },
        {
          "scheme": "urn:enbox:payment:cashu:v0",
          "amount": "100",
          "description": "RecordsWrite (1.2 MB)",
          "mints": ["https://mint.example.com"],
          "unit": "sat"
        }
      ]
    }
  }
}
```

Like all DWN-level errors, the `-50402` response is returned inside a JSON-RPC
envelope over an HTTP `200` response. The HTTP status code reflects transport-level
success; the JSON-RPC error code reflects the payment requirement.

#### `PaymentRequiredData` Object {#payment-required-data}

| Field | Type | Required | Description |
|---|---|---|---|
| `paymentId` | `string` | Yes | A server-generated identifier for this payment challenge. The client ****MUST**** include this value in the `PaymentProof` when retrying. Servers ****SHOULD**** use a UUID v4 or similarly unique value. |
| `paymentRequirements` | `PaymentRequirements[]` | Yes | An array of one or more payment options. Each entry describes one [[ref:Payment Scheme]] the server accepts for this message. |

#### `PaymentRequirements` Object {#payment-requirements-object}

| Field | Type | Required | Description |
|---|---|---|---|
| `scheme` | `string` | Yes | The [[ref:Payment Scheme]] URN identifier (e.g., `"urn:enbox:payment:l402:v0"`). |
| `amount` | `string` | Yes | The price in the server's base currency unit (as advertised in `ServerInfo.payment.currency`). String to avoid floating-point precision issues. |
| `description` | `string` | No | A human-readable description of the charge (e.g., `"RecordsWrite (1.2 MB)"`). |

Each [[ref:Payment Scheme]] extends `PaymentRequirements` with additional
scheme-specific fields. See [Payment Scheme Registry](#payment-scheme-registry)
for the fields defined by each registered scheme.

### The `dwn-payment` Header {#dwn-payment-header}

When retrying a request after receiving a `-50402` error, the client ****MUST****
include a `dwn-payment` HTTP header containing a JSON-serialized `PaymentProof`
object.

The `dwn-payment` header is ****OPTIONAL****. If a server does not require payment
for a given message, the header is ignored if present. If a server requires payment
and the header is absent, the server returns `-50402`.

::: example RecordsWrite with Payment Proof
```http
POST / HTTP/1.1
Host: dwn.example.com
Content-Type: application/octet-stream
dwn-request: {"jsonrpc":"2.0","id":"abc-123","method":"dwn.processMessage","params":{"target":"did:example:alice","message":{...}}}
dwn-payment: {"paymentId":"pay_7f3a8b2c-1d4e-5f6a-9b8c-7d6e5f4a3b2c","scheme":"urn:enbox:payment:l402:v0","macaroon":"0201036c6e6402...","preimage":"79852a07912..."}

<binary record data>
```
:::

#### `PaymentProof` Object {#payment-proof-object}

| Field | Type | Required | Description |
|---|---|---|---|
| `paymentId` | `string` | Yes | The `paymentId` from the `PaymentRequiredData` that prompted this payment. |
| `scheme` | `string` | Yes | The [[ref:Payment Scheme]] URN identifier. ****MUST**** match one of the schemes offered in the `paymentRequirements`. |

Each [[ref:Payment Scheme]] extends `PaymentProof` with additional scheme-specific
fields containing the proof of payment. See [Payment Scheme Registry](#payment-scheme-registry).

### Server Behavior {#payment-server-behavior}

#### Pricing Policy {#pricing-policy}

The server ****MUST**** evaluate a pricing policy to determine whether a message
requires payment and how much it costs. The pricing policy is server-defined and
****MAY**** consider any combination of:

- The DWN interface and method (`Records`/`Write`, `Records`/`Query`, etc.)
- The data size (for `RecordsWrite`)
- The protocol URI
- The target tenant
- The message author

If the evaluated price is zero, the server ****MUST**** process the message without
requiring payment, even if payment is enabled.

#### Payment Verification {#payment-verification}

When a `dwn-payment` header is present, the server ****MUST****:

1. Parse and validate the `PaymentProof` JSON.
2. Verify that the `paymentId` matches a recently issued payment challenge.
3. Dispatch to the [[ref:Payment Scheme]] adapter matching `proof.scheme`.
4. Verify the proof according to the scheme's rules (e.g., macaroon signature
   verification for L402, proof redemption check for Cashu, signature verification
   for x402).
5. If verification fails, the server ****MUST**** return a `-50402` error with
   fresh `paymentRequirements`.

#### Payment Settlement {#payment-settlement}

After the DWN message is successfully processed, the server ****SHOULD**** settle
the payment (e.g., broadcast an on-chain transaction for x402, or mark the payment
as complete for L402/Cashu). Settlement is scheme-specific and ****MAY**** be
asynchronous.

If settlement fails after the message has been processed, the server ****SHOULD****
log the failure for manual reconciliation. The server ****MUST NOT**** reverse the
DWN message processing — the message is committed regardless of settlement outcome.

#### Idempotency {#payment-idempotency}

The server ****MUST**** ensure that a given `paymentId` can only be used once. If
a client retries with the same `paymentId` after a successful payment, the server
****MUST**** return the original response without charging again.

Servers ****SHOULD**** expire unused `paymentId` values after a reasonable timeout
(****RECOMMENDED**** 5 minutes).

### Client Behavior {#payment-client-behavior}

#### Detection {#payment-detection}

Clients ****MUST**** check for error code `-50402` in JSON-RPC error responses.
When detected:

1. Parse the `error.data` field as a `PaymentRequiredData` object.
2. Iterate the `paymentRequirements` array.
3. For each entry, check whether a registered [[ref:Payment Handler]] supports
   the `scheme`.
4. If a handler is found, invoke it to produce a `PaymentProof`.
5. Retry the original request with the `dwn-payment` header.

Clients ****MUST NOT**** retry more than once per `-50402` response to prevent
infinite payment loops. If the retry also returns `-50402`, the client ****MUST****
surface the error to the application layer.

#### No Handler Available {#payment-no-handler}

If the client does not have a [[ref:Payment Handler]] for any of the offered
schemes, the client ****SHOULD**** surface the `PaymentRequiredData` to the
application layer. This allows applications to present payment options to the user
or handle the error appropriately.

Clients ****MUST NOT**** silently drop `-50402` errors. The error ****MUST**** be
propagated to the caller.

#### Feature Detection {#payment-feature-detection}

Clients ****MAY**** inspect the `payment` field in [ServerInfo](#server-info-object)
to determine payment capabilities before sending messages. This allows clients to:

- Discover which [[ref:Payment Scheme]]s a server accepts.
- Display pricing information to users.
- Pre-select a DWN provider based on cost.

Clients ****MUST**** check `payment.methods` to determine the payment flow model.
If the client does not recognize any of the listed methods, it ****SHOULD NOT****
attempt to handle `-50402` errors from that server.

### Payment Methods {#payment-methods}

The `payment.methods` array in [ServerInfo](#server-info-object) declares which
payment flow models the server supports. This is the primary feature detection
signal for clients.

| Value | Description |
|---|---|
| `per-message-payment-v0` | Per-message challenge/response as defined in this section. The server returns `-50402` on individual messages that require payment. |

::: note
Future specifications may define additional methods such as `prepaid-balance-v0`
(deposit credits, deduct per message) or `subscription-v0` (recurring payment for
a quota). Methods are orthogonal to each other and to
[registration requirements](#registration-requirement-values).
:::

### Payment Scheme Registry {#payment-scheme-registry}

[[ref:Payment Scheme]] identifiers ****MUST**** use the URN format
`urn:enbox:payment:<scheme-name>:<version>` to ensure global uniqueness and prevent
collisions between independently developed schemes.

Third parties ****MAY**** define their own schemes using their own URN namespace
(e.g., `urn:example:payment:stripe:v0`). Servers and clients negotiate scheme
support through the `paymentRequirements` array — no central registry is required
for interoperability.

The following schemes are defined by this specification:

#### `urn:enbox:payment:l402:v0` — Lightning L402 {#scheme-l402}

L402 combines [macaroons](https://research.google/pubs/macaroons-cookies-with-contextual-caveats-for-decentralized-authorization-in-the-cloud/)
(bearer authentication tokens with embedded caveats) and Lightning Network invoices.
The client pays a Lightning invoice to obtain a preimage, then presents the macaroon
and preimage together as proof of payment.

**Additional `PaymentRequirements` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `invoice` | `string` | Yes | A BOLT-11 Lightning invoice for the required amount. |
| `macaroon` | `string` | Yes | A hex-encoded macaroon with the invoice's payment hash as a caveat. |

**Additional `PaymentProof` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `macaroon` | `string` | Yes | The macaroon from the payment requirements (hex-encoded). |
| `preimage` | `string` | Yes | The Lightning payment preimage obtained by paying the invoice (hex-encoded). |

**Verification:** The server verifies the macaroon signature chain using its root
key, validates all caveats (amount, expiry, etc.), and confirms that
`SHA-256(preimage) == payment_hash` from the macaroon's payment hash caveat.

**Settlement:** Settlement is atomic with payment. The Lightning payment transfers
value when the client obtains the preimage. No additional settlement step is required.

::: example L402 Payment Requirements
```json
{
  "scheme": "urn:enbox:payment:l402:v0",
  "amount": "100",
  "description": "RecordsQuery",
  "invoice": "lnbc1000n1pj9nwz2pp5xt7...",
  "macaroon": "0201036c6e640204..."
}
```
:::

::: example L402 Payment Proof
```json
{
  "paymentId": "pay_7f3a8b2c-1d4e-5f6a-9b8c-7d6e5f4a3b2c",
  "scheme": "urn:enbox:payment:l402:v0",
  "macaroon": "0201036c6e640204...",
  "preimage": "79852a0791225dee00be0a6cf31a1619782c21d35995e118bfc74ad812174035"
}
```
:::

#### `urn:enbox:payment:cashu:v0` — Cashu Ecash {#scheme-cashu}

[Cashu](https://cashu.space) ecash tokens are bearer instruments — cryptographic
proofs of value issued by a mint. The client sends ecash proofs that the server
redeems with the mint. Cashu provides instant settlement, privacy (blind-signed
tokens are unlinkable to the payer), and support for very small amounts.

**Additional `PaymentRequirements` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `mints` | `string[]` | Yes | Accepted mint URLs. The client ****MUST**** provide proofs from one of these mints. |
| `unit` | `string` | No | The Cashu unit (e.g., `"sat"`). Defaults to the server's `payment.currency`. |

**Additional `PaymentProof` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `tokens` | `string` | Yes | A Cashu token string (`cashuA...` or `cashuB...` encoded) containing proofs totaling at least the required amount. |

**Verification:** The server decodes the Cashu token, verifies the mint URL is in
the accepted list, checks proof state with the mint (unspent), and verifies the
total proof amount meets or exceeds the required amount.

**Settlement:** The server redeems the proofs with the mint (swap or receive). This
is instant and atomic — once redeemed, the value is transferred.

::: example Cashu Payment Requirements
```json
{
  "scheme": "urn:enbox:payment:cashu:v0",
  "amount": "100",
  "description": "RecordsWrite (1.2 MB)",
  "mints": ["https://mint.example.com"],
  "unit": "sat"
}
```
:::

::: example Cashu Payment Proof
```json
{
  "paymentId": "pay_7f3a8b2c-1d4e-5f6a-9b8c-7d6e5f4a3b2c",
  "scheme": "urn:enbox:payment:cashu:v0",
  "tokens": "cashuAeyJ0b2tlbiI6W3sicH..."
}
```
:::

#### `urn:enbox:payment:x402-exact:v0` — x402 Exact (EVM/SVM) {#scheme-x402}

[x402](https://github.com/coinbase/x402) uses signed on-chain token transfer
authorizations. The client signs an EIP-3009 `transferWithAuthorization` (EVM) or
equivalent authorization (SVM) that a facilitator service broadcasts and settles
on-chain.

**Additional `PaymentRequirements` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `network` | `string` | Yes | Chain identifier (e.g., `"eip155:8453"` for Base, `"solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp"` for Solana mainnet). |
| `asset` | `string` | Yes | Token contract address (EVM) or mint address (SVM). |
| `payTo` | `string` | Yes | The provider's receiving address. |
| `maxTimeoutSeconds` | `number` | No | Maximum time the server will wait for settlement. Defaults to `60`. |
| `extra` | `object` | No | Additional chain-specific metadata (e.g., `{ "name": "USDC", "version": "2" }` for EIP-712 domain). |

**Additional `PaymentProof` fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `payload` | `string` | Yes | Base64-encoded x402 `PaymentPayload` containing the signed authorization. |

**Verification:** The server forwards the payload to an x402 facilitator's
`/verify` endpoint, or verifies the EIP-712 signature locally.

**Settlement:** The server forwards the payload to the facilitator's `/settle`
endpoint. The facilitator broadcasts the on-chain transaction and returns the
transaction hash.

::: note
The x402 scheme requires an external facilitator service for settlement. The
facilitator broadcasts the transaction and pays gas fees. See the
[x402 specification](https://github.com/coinbase/x402) for facilitator API details.
:::

::: example x402 Payment Requirements
```json
{
  "scheme": "urn:enbox:payment:x402-exact:v0",
  "amount": "10000",
  "description": "RecordsWrite (1.2 MB)",
  "network": "eip155:8453",
  "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "payTo": "0x1234567890abcdef1234567890abcdef12345678",
  "maxTimeoutSeconds": 60,
  "extra": { "name": "USDC", "version": "2" }
}
```
:::

### Design Notes {#payment-design-notes}

- **Transport-layer concern:** Per-message payment is a transport/server concern,
  not a DWN protocol concern. The DWN specification and the DWN SDK do not need
  to be aware of payment. Payment requirements and proofs are exchanged via HTTP
  headers and JSON-RPC error codes, outside the DWN message envelope.

- **Multiple schemes simultaneously:** A server ****MAY**** offer multiple payment
  schemes in a single `paymentRequirements` array. The client selects the first
  scheme it supports. This allows providers to accept both Lightning and Cashu (or
  any combination) without requiring clients to support all methods.

- **Payment is optional on both sides:** Servers that don't enable payment never
  return `-50402`. Clients that don't implement payment handlers work against free
  servers and receive typed errors from paid servers. Neither side breaks when the
  other doesn't support payment.

- **Pricing granularity:** The pricing policy is server-defined and not specified
  in this document. A server ****MAY**** charge flat per-message rates, per-byte
  rates for writes, different rates for different interfaces, or implement dynamic
  pricing. The `PaymentRequirements.amount` reflects the evaluated price for the
  specific message that triggered the `-50402`.

- **Data-bearing retries:** When a `RecordsWrite` with data receives a `-50402`,
  the client ****MUST**** resend both the `dwn-request` header and the request
  body (binary data) on retry, along with the `dwn-payment` header. The server
  ****MUST NOT**** cache request bodies across the challenge/response boundary.


## Sync Orchestration {#sync-orchestration}

The DWN specification defines the `MessagesSync` interface and the Sparse Merkle Tree
data structure for set reconciliation. This section specifies how a [[ref:DWN Client]]
(typically an agent or another DWN Server) ****SHOULD**** orchestrate the sync process
between two DWN nodes.

::: note
The `MessagesSync` message format, SMT structure, and tree membership rules are
defined in the [DWN specification](https://github.com/enboxorg/dwn-spec). This
section specifies the orchestration protocol that uses those primitives.
:::

### Sync Roles {#sync-roles}

In a sync session, there are two participants:

- **Local node:** The DWN instance the client controls directly (typically via in-process calls).
- **Remote node:** The DWN instance the client communicates with over the network
  (via the transports defined in this specification).

Sync is bidirectional: the client performs a **pull** phase (remote → local) and a
**push** phase (local → remote). Both phases use the same `MessagesSync` protocol
but in opposite directions.

### Authorization for Sync {#sync-authorization}

To sync with a remote DWN, the client ****MUST**** have authorization to perform
the following operations on the remote node:

- `MessagesSync` — To compare tree state and enumerate leaves.
- `MessagesRead` — To fetch the full content of missing messages.

For pushing messages, the client ****MUST**** have authorization to process the
messages being sent (e.g., `RecordsWrite` permission if pushing record writes).

These permissions are typically obtained via [[ref:Permission Grants]]. For
protocol-scoped sync, the grants ****MUST**** include:

| Scope | Purpose |
|---|---|
| `Messages` / `Sync` scoped to protocol | Compare tree state for the protocol |
| `Messages` / `Read` scoped to protocol | Fetch missing messages |
| `Protocols` / `Query` scoped to protocol | Verify protocol definition availability |

::: note
A future revision of the DWN specification may simplify these into a unified
`Messages` / `Read` scope that covers sync, read, and subscribe operations for
a given protocol — mirroring how protocol `$actions` already treats `read` as a
unified action covering read, query, subscribe, and count.
:::

### Pull Sync Flow {#pull-sync-flow}

Pull sync fetches messages from the remote node that the local node is missing.

**Phase 1 — Root Comparison:**

1. Send `MessagesSync` with `action: "root"` to the remote node (optionally with
   `protocol` for scoped sync).
2. Query the local node for its root hash.
3. If the root hashes are identical, the nodes are in sync — skip to push sync.

**Phase 2 — Tree Walk:**

1. Starting from prefix `""`, send `MessagesSync` with `action: "subtree"` to both
   the remote and local nodes.
2. Compare the subtree hashes:
   - If hashes match: the subtree is identical, skip it.
   - If one side returns the default hash (empty): all entries under this prefix
     exist only on the other side. Switch to `action: "leaves"` to enumerate them.
   - If both sides differ: extend the prefix by appending `"0"` and `"1"` and
     recurse on both children.
3. Continue until reaching the configured walk depth (****RECOMMENDED**** depth
   of 16), then switch to `action: "leaves"`.

**Phase 3 — Leaf Enumeration and Set Difference:**

1. For each divergent prefix, send `MessagesSync` with `action: "leaves"` to both
   nodes.
2. Compute the set difference: `messageCids` present on the remote but not locally.
3. For each missing `messageCid`, send `MessagesRead` with `{ messageCid }` to
   the remote node to retrieve the full message and any associated data.

**Phase 4 — Message Application:**

1. Process each fetched message on the local node via `processMessage`.
2. Messages ****MUST**** be applied respecting dependency order as defined in the
   DWN specification's [Sync Flow](https://github.com/enboxorg/dwn-spec) section:
   - `ProtocolsConfigure` before records referencing that protocol.
   - Parent records before child records.
   - Initial `RecordsWrite` before updates or deletes for the same record.
   - Permission grants before messages that reference them.
3. Implementations ****SHOULD**** use topological sorting or multiple passes with
   retry to satisfy ordering constraints.

### Push Sync Flow {#push-sync-flow}

Push sync sends local messages to the remote node that the remote is missing.

The flow is identical to pull sync but with reversed roles:

1. Compare root hashes (local vs. remote).
2. Walk the tree to identify divergent subtrees.
3. Enumerate leaves and compute the set difference: `messageCids` present locally
   but not on the remote.
4. For each missing message, retrieve it from the local node and send it to the
   remote node via `dwn.processMessage` over the [HTTP transport](#http-transport).

::: note
For `RecordsWrite` messages with data, the push ****MUST**** use the HTTP transport
since the WebSocket transport does not support data payloads.
:::

### Scoped Sync {#scoped-sync}

Sync ****MAY**** be scoped to a specific protocol by including the `protocol` field
in `MessagesSync` messages. This enables selective synchronization — for example,
syncing only a specific application's data between nodes without transferring
unrelated records.

When performing scoped sync, all `MessagesSync` messages in the session ****MUST****
include the same `protocol` value. The resulting tree comparison operates on the
per-protocol SMT rather than the global tree.

### Sync Scheduling {#sync-scheduling}

Implementations ****SHOULD**** perform sync on a periodic schedule.
A ****RECOMMENDED**** default interval is 2 minutes.

Implementations ****MAY**** also trigger sync on demand — for example, after a local
write, after receiving a subscription event, or in response to a user action.

Implementations ****SHOULD**** implement backoff when repeated sync attempts find
no changes, and ****SHOULD**** increase sync frequency when changes are detected.


## Security Considerations {#security-considerations}

### TLS Requirements {#tls-requirements}

All HTTP and WebSocket connections ****MUST**** use TLS.
[[ref:Service Endpoint]] URLs ****MUST**** use the `https` scheme.
WebSocket connections ****MUST**** use `wss`.

Plaintext `http` and `ws` connections ****MUST NOT**** be used in production.
Implementations ****MAY**** permit plaintext connections for local development only.

### Rate Limiting {#rate-limiting}

Servers ****SHOULD**** implement rate limiting to prevent abuse. Rate limits ****MAY****
vary by tenant, by source IP, or by operation type.

Servers ****SHOULD**** return HTTP `429 Too Many Requests` when a rate limit is exceeded.

### Message Size Limits {#message-size-limits}

The server's maximum file size is advertised in the [ServerInfo](#server-info-object)
`maxFileSize` field. Clients ****MUST NOT**** send `RecordsWrite` data exceeding
this limit. Servers ****SHOULD**** reject oversized payloads with HTTP `413 Payload
Too Large`.

The `dwn-request` header value ****SHOULD NOT**** exceed 8 KB. Implementations
****MAY**** reject requests with excessively large headers.

### WebSocket Security {#websocket-security}

- Servers ****SHOULD**** limit the number of concurrent WebSocket connections per
  source IP to prevent resource exhaustion.
- Servers ****SHOULD**** limit the number of active subscriptions per connection.
- Servers ****MUST**** close WebSocket connections that fail heartbeat checks.
- Servers ****SHOULD**** set a maximum WebSocket message frame size and reject
  oversized frames.
- Servers ****MUST**** enforce the [flow control](#ws-flow-control) buffer limit
  to prevent unbounded memory growth from unacknowledged subscriptions.

### Header Injection {#header-injection}

The `dwn-request`, `dwn-response`, and `dwn-payment` headers carry JSON payloads.
Implementations ****MUST**** validate that these values are well-formed JSON before
processing. Implementations ****MUST NOT**** reflect unsanitized header values in
responses.

### Payment Security {#payment-security}

- Servers ****MUST**** expire `paymentId` values after a bounded timeout to prevent
  replay attacks using stale payment challenges.
- Servers ****MUST**** ensure each `paymentId` is used at most once.
- Servers ****MUST**** verify payment proofs before processing the DWN message.
  A server ****MUST NOT**** process a message and then fail payment verification.
- For the L402 scheme, servers ****MUST**** use cryptographically strong root keys
  for macaroon signing and ****SHOULD**** rotate root keys periodically.
- For the Cashu scheme, servers ****MUST**** verify proof state (unspent) with the
  mint before processing the message, to prevent double-spend of ecash proofs.
- For the x402 scheme, servers ****SHOULD**** verify the payment signature before
  processing the message, either locally or via the facilitator's `/verify` endpoint.
- The `dwn-payment` header ****SHOULD NOT**** exceed 8 KB. Implementations ****MAY****
  reject requests with excessively large payment headers.

### Transport-Layer Privacy {#transport-privacy}

::: note
DWN messages carried over HTTPS or WebSocket are protected by TLS, which encrypts
the wire content from network observers. However, TLS alone does not conceal all
metadata. The target DWN's service endpoint URL is visible in DNS lookups and TLS
SNI, the sender's IP address is visible to the server, and request timing and
sizes are observable by network intermediaries. When record-level encryption is
used (as defined in the [DWN specification](https://github.com/enboxorg/dwn-spec)),
the record data is additionally protected at rest — but the transport metadata
remains exposed.

For deployments that require stronger transport-layer privacy, implementations
****MAY**** wrap DWN JSON-RPC payloads inside a [DIDComm v2](https://identity.foundation/didcomm-messaging/spec/v2.1/)
encrypted message envelope before transmission. DIDComm's routing protocol
enables onion-style multi-hop delivery through mediators, where each intermediary
sees only the next hop and an opaque encrypted payload — no single party along
the route learns both the original sender and the final recipient. The DIDComm
envelope is stripped upon delivery to the target DWN server, and the inner DWN
message is then processed through the standard authorization and handling pipeline.

This layering is strictly additive: the DIDComm envelope provides ephemeral,
transit-scoped privacy (protecting the journey), while DWN's own authorization
model, CID-based signatures, and record-level encryption provide persistent,
data-scoped security (protecting the destination). The two layers serve
fundamentally different purposes and ****MUST NOT**** be conflated:

- **DIDComm envelopes are disposable.** They use per-message ECDH key agreement
  (ECDH-1PU for authenticated encryption, or ECDH-ES for anonymous encryption)
  to protect a single message in transit. Once delivered, the envelope is
  discarded. DIDComm signatures (when used) wrap the full message body, making
  them unsuitable for content-addressed storage where data and metadata are
  stored and verified independently.

- **DWN authorization is persistent.** DWN messages sign a CID of the descriptor
  — a content-addressed commitment that remains verifiable regardless of how the
  message was transported or re-serialized. DWN's multi-layered authorization
  (owner, delegate, permission grant, protocol rules) and hierarchical key
  derivation for encryption are designed for long-lived data, not ephemeral
  communication.

Implementations that adopt DIDComm as a transport envelope ****MUST**** ensure that
DWN message authentication and authorization are performed on the unwrapped DWN
message, not on any properties of the DIDComm envelope. The DIDComm layer
provides no authorization guarantees that are meaningful to the DWN processing
pipeline.
:::

## Normative References

[[spec]]
~ [DWN Specification](https://github.com/enboxorg/dwn-spec). Enbox.

[[spec:RFC6455]]
~ [The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455). I. Fette, A. Melnikov. December 2011.

[[spec:RFC7049]]
~ [Concise Binary Object Representation (CBOR)](https://datatracker.ietf.org/doc/html/rfc7049). C. Bormann, P. Hoffman. October 2013.

[[spec:RFC8259]]
~ [The JavaScript Object Notation (JSON) Data Interchange Format](https://datatracker.ietf.org/doc/html/rfc8259). T. Bray. December 2017.

[[spec:RFC2119]]
~ [Key words for use in RFCs to Indicate Requirement Levels](https://datatracker.ietf.org/doc/html/rfc2119). S. Bradner. March 1997.

[[spec:RFC9110]]
~ [HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110). R. Fielding, M. Nottingham, J. Reschke. June 2022.
