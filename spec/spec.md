
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

## Service Endpoint Discovery {#service-endpoint-discovery}

### DID Document Service Entry {#did-document-service-entry}

A [[ref:DWN Client]] discovers the network location of a DID's [[ref:Decentralized Web Node]]
by resolving the DID and inspecting the DID document for a service entry of type
`DecentralizedWebNode`.

The service entry ****MUST**** conform to the following requirements:

- The `id` property ****MUST**** have a fragment value of `#dwn`.
- The `type` property ****MUST**** be `"DecentralizedWebNode"`.
- The `serviceEndpoint` property ****MUST**** be either:
  - A `string` value containing a single HTTPS URL.
  - An `array` of `string` values, each containing an HTTPS URL. Multiple
    endpoints indicate replicated nodes that synchronize to the same state.

::: example DID Document with DWN Service Entry
```json
{
  "id": "did:example:alice",
  "service": [{
    "id": "#dwn",
    "type": "DecentralizedWebNode",
    "serviceEndpoint": [
      "https://dwn.example.com",
      "https://dwn-backup.example.com"
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
****SHOULD**** treat them as equivalent replicas. The client ****MAY**** select any
endpoint for a given request. Clients ****SHOULD**** implement failover by attempting
alternative endpoints when a request fails.

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
  and `result.event` containing the event payload.

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
| `Access-Control-Expose-Headers` | `dwn-response` |

Servers ****MUST**** respond to `OPTIONS` preflight requests with HTTP `204 No Content`
and the above headers.

::: note
The `dwn-response` header must be listed in `Access-Control-Expose-Headers` to ensure
browser-based clients can read it. Without this, the header is opaque to JavaScript
running in a browser context.
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

#### Receiving Events {#ws-events}

Subsequent events are pushed as JSON-RPC response objects where:

- `id` is the **subscription** `id` (not the original request `id`).
- `result.event` contains the event payload.

```json
{
  "jsonrpc": "2.0",
  "id": "sub-1",
  "result": {
    "event": {
      "message": { ... },
      "initialWrite": { ... }
    }
  }
}
```

Clients ****SHOULD**** deduplicate events by `messageCid`, as the DWN specification
notes that duplicate events may be received.

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
  "registrationRequirements": ["proof-of-work-sha256-v0", "terms-of-service"]
}
```

#### ServerInfo Object {#server-info-object}

| Property | Type | Required | Description |
|---|---|---|---|
| `server` | `string` | Yes | Server implementation identifier |
| `version` | `string` | Yes | Server version (semver) |
| `sdkVersion` | `string` | Yes | DWN SDK version supported |
| `url` | `string` | Yes | The canonical base URL of the server |
| `webSocketSupport` | `boolean` | Yes | Whether the server supports WebSocket connections |
| `maxFileSize` | `number` | Yes | Maximum data size in bytes for a single `RecordsWrite` |
| `registrationRequirements` | `string[]` | Yes | List of registration requirements (empty array if open) |

#### Registration Requirement Values {#registration-requirement-values}

| Value | Description |
|---|---|
| `proof-of-work-sha256-v0` | Client must complete a SHA-256 proof-of-work challenge |
| `terms-of-service` | Client must accept the server's terms of service |

### Health Endpoint {#health-endpoint}

Servers ****SHOULD**** expose a health check endpoint:

```
GET /health
```

**Response:** `200 OK` with body `{"ok": true}` and `Content-Type: application/json`.

## Tenant Registration {#tenant-registration}

Multi-tenant [[ref:DWN Servers]] ****MAY**** require tenant registration before accepting
DWN messages for a given DID. Registration prevents abuse by requiring proof of
computational work and/or acceptance of terms of service.

A server that requires registration ****MUST**** advertise its requirements in the
[ServerInfo](#server-info-object) `registrationRequirements` array. A server with
an empty `registrationRequirements` array is open and does not require registration.

### Registration Flow {#registration-flow}

1. The client fetches the [ServerInfo](#info-endpoint) to discover registration requirements.
2. If `proof-of-work-sha256-v0` is required, the client fetches a proof-of-work challenge.
3. If `terms-of-service` is required, the client fetches the terms of service text.
4. The client computes the proof-of-work response and/or terms-of-service hash.
5. The client submits the registration request.

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
  }
}
```

| Field | Required | Description |
|---|---|---|
| `registrationData.did` | Yes | The DID to register as a tenant |
| `registrationData.termsOfServiceHash` | Conditional | SHA-256 hex hash of the ToS text. Required if `terms-of-service` is in the registration requirements. |
| `proofOfWork.challengeNonce` | Conditional | The challenge nonce from the server. Required if `proof-of-work-sha256-v0` is in the registration requirements. |
| `proofOfWork.responseNonce` | Conditional | The client's solution nonce. Required if `proof-of-work-sha256-v0` is in the registration requirements. |

**Response:**
- **200:** `{ "success": true }` — Registration succeeded.
- **400:** `{ "code": "...", "message": "..." }` — Validation failure (invalid
  proof-of-work, wrong terms hash, expired challenge, etc.).


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

### Header Injection {#header-injection}

The `dwn-request` and `dwn-response` headers carry JSON payloads. Implementations
****MUST**** validate that these values are well-formed JSON before processing.
Implementations ****MUST NOT**** reflect unsanitized header values in responses.

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
