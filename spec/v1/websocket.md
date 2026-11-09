# WebSocket Custom Protocol Binding for A2A

This document specifies the **WebSocket** custom protocol binding for the Agent2Agent (A2A) protocol.

This binding is best understood as two distinct layers:

1. **Transport layer - WebSocket ([RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)).** A single, long-lived, full-duplex connection established via an HTTP/1.1 `Upgrade` handshake. WebSocket provides nothing more than ordered, message-framed, bidirectional delivery of text messages. It defines no request/response correlation, no method dispatch, and no error model of its own.
2. **Application layer - JSON-RPC 2.0.** Every A2A operation is carried as a [JSON-RPC 2.0](https://www.jsonrpc.org/specification) request/response object - the same request, response, and error objects defined by the A2A [JSON-RPC binding](https://a2a-protocol.org/v1.0.0/specification/#9-json-rpc-protocol-binding), with one such object per WebSocket text message. The method inventory, `params`/`result` shapes, ProtoJSON serialization, and numeric error codes are identical to that binding; only the framing differs (a WebSocket text message instead of an HTTP request body or SSE `data:` line).

On top of these two layers, the binding defines a small set of **A2A extensions** to the JSON-RPC object that exist because a single persistent connection carries many concurrent RPCs:

- **Multiplexing by request `id`:** responses, stream events, and errors are correlated to their originating request by the JSON-RPC `id` (see [Section 3.5](#35-multiplexing))
- **`serviceParams` member:** an additional member of the JSON-RPC request object that carries per-request A2A service parameters, since individual WebSocket messages have no headers (see [Section 4.2](#42-per-request-message-scoped))
- **`ws/StreamEnd` notification:** sent by the server to terminate a stream, since the connection stays open after a stream completes (see [Section 3.4](#34-streaming-messages))
- **`ws/CancelStream` notification:** sent by the client to stop one stream without closing the connection (see [Section 3.4](#34-streaming-messages))

The binding also defines two **optional** authentication extensions for long-lived connections: a server-originated `ws/ReauthenticationRequired` notification and a `ws/Authenticate` method for in-band authentication and token refresh (see [Section 9](#9-authentication-and-authorization)).

The control messages above are ordinary JSON-RPC 2.0 notifications and requests; only their method names, which use the reserved `ws/` prefix, are defined by this binding. The `serviceParams` member is the only addition to the JSON-RPC object itself. Receivers **MUST** ignore object members they do not recognize rather than rejecting the message.

## 1. Protocol Requirements

- **Transport Layer:** WebSocket ([RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)), established via an HTTP/1.1 `Upgrade` handshake
- **Application Layer:** JSON-RPC 2.0 ([specification](https://www.jsonrpc.org/specification)), reusing the request, response, and error objects of the A2A [JSON-RPC binding](https://a2a-protocol.org/v1.0.0/specification/#9-json-rpc-protocol-binding)
- **Definition:** Uses the same method inventory and message types as the JSON-RPC binding, derived from `specification/a2a.proto`
- **Serialization:** JSON following ProtoJSON conventions ([Section 5.5](https://a2a-protocol.org/v1.0.0/specification/#55-json-field-naming-convention)), UTF-8 encoded
- **Sub-protocol:** `a2a.v1`, negotiated via the `Sec-WebSocket-Protocol` header during the handshake. The `v1` identifies the major version of this binding, not the A2A protocol version, and changes together with the Transport Identifier
- **Message Type:** Text messages only, each carrying exactly one JSON-RPC 2.0 object; binary messages are reserved for future use. A message **MAY** be fragmented into multiple frames as permitted by [RFC 6455 Section 5.4](https://datatracker.ietf.org/doc/html/rfc6455#section-5.4)
- **Service Name:** Not applicable. Operations are addressed by the JSON-RPC `method` member using the method names of `lf.a2a.v1.A2AService` from `specification/a2a.proto` (e.g., `SendMessage`), with no service prefix
- **Connection Model:** A single persistent, full-duplex connection carrying many concurrent RPCs, multiplexed by the JSON-RPC `id`
- **Service Parameters:** Connection-scoped parameters are sent on the `Upgrade` request as HTTP headers, or as URL query parameters for clients that cannot set headers (such as browsers); per-request parameters are sent in the `serviceParams` member, which overrides connection-scoped values. `A2A-Version` is always connection-scoped
- **Streaming:** Server-streaming only. A sequence of JSON-RPC `result` responses sharing the request `id`, terminated by a `ws/StreamEnd` notification; clients may stop a stream with a `ws/CancelStream` notification without closing the connection
- **Authentication:** Established once per connection, using the schemes declared in the Agent Card `securitySchemes`. Credentials are sent during the HTTP `Upgrade` handshake (e.g., `Authorization: Bearer <token>`) or, for clients that cannot set handshake headers, with the optional `ws/Authenticate` method as the first message. The authenticated identity applies to all messages on the connection
- **Transport Security:** `wss://` (WebSocket over TLS) **MUST** be used in production; TLS 1.3 **SHOULD** be used and TLS 1.2 is the minimum. `ws://` is permitted only for local development
- **Transport Identifier:** `https://a2a-protocol.org/bindings/experimental-websocket/v1` (used in Agent Card `protocolBinding` fields). A breaking change to this binding **MUST** use a new identifier (e.g., `.../v2`) and a matching new sub-protocol (e.g., `a2a.v2`)

## 2. Connection Lifecycle

### 2.1. Connection Establishment

WebSocket connections are initiated by the client using the standard HTTP/1.1 Upgrade handshake as defined in [RFC 6455 Section 4](https://datatracker.ietf.org/doc/html/rfc6455#section-4).

**Handshake requirements:**

1. The client **MUST** include `Sec-WebSocket-Protocol: a2a.v1` in the upgrade request
2. The server **MUST** confirm the sub-protocol by echoing `Sec-WebSocket-Protocol: a2a.v1` in the 101 response
3. If the server does not support the `a2a.v1` sub-protocol, it **SHOULD** reject the upgrade with an appropriate HTTP error (e.g., 400 Bad Request)
4. The client **MUST** fail the connection if the 101 response does not echo `a2a.v1`, as required by [RFC 6455 Section 4.1](https://datatracker.ietf.org/doc/html/rfc6455#section-4.1)
5. The client **MUST** send the `A2A-Version` service parameter on the upgrade request, as a header or query parameter (see [Section 4.1](#41-during-connection-upgrade-connection-scoped))
6. The client **SHOULD** include credentials in the upgrade request if the Agent Card declares security requirements (see [Section 9](#9-authentication-and-authorization))

**Example handshake:**

```
GET /a2a/ws HTTP/1.1
Host: agent.example.com
Origin: https://app.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: a2a.v1
A2A-Version: 1.0
Authorization: Bearer <token>
```

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: a2a.v1
```

#### 2.1.1. Clients That Cannot Set Handshake Headers

The browser `WebSocket` API only allows a client to set the URL and the list of sub-protocols; it cannot set `Authorization` or `A2A-*` headers. Such clients:

- **MUST** send connection-scoped service parameters, including `A2A-Version`, as URL query parameters (e.g., `wss://agent.example.com/a2a/ws?A2A-Version=1.0`)
- **SHOULD** authenticate with cookies the server has issued for the connection's origin, or with the `ws/Authenticate` method as the first message after the upgrade (see [Section 9](#9-authentication-and-authorization))
- **SHOULD NOT** place bearer tokens or other credentials in the URL, because URLs are commonly recorded in server, proxy, and browser logs

Such clients cannot observe the HTTP status or `WWW-Authenticate` header of a rejected upgrade; they only see the connection fail (close code `1006`). Servers that support these clients **MAY** therefore complete the upgrade without credentials, provided that:

- The client **MUST** send `ws/Authenticate` before any other request
- The server **MUST** reject any other request received before successful authentication with error code `-32000` and reason `WS_UNAUTHENTICATED` (see [Section 7.3](#73-binding-specific-errors))
- The server **MUST** close the connection with close code `4001` if authentication does not succeed within a short timeout (recommended: 10 seconds)

### 2.2. Connection Maintenance

- Servers **SHOULD** send WebSocket ping frames ([RFC 6455 Section 5.5.2](https://datatracker.ietf.org/doc/html/rfc6455#section-5.5.2)) at a configurable interval to keep the connection alive (recommended: 30 seconds)
- Clients reply to pings with pong frames as required by RFC 6455; browsers do this automatically, so no application code is needed
- If no pong is received within a reasonable timeout (recommended: 10 seconds), the server **SHOULD** close the connection
- Implementations **SHOULD** close a connection that has been idle for a maximum idle timeout (recommended: 5 minutes). A connection is idle only when no requests or streams are in flight and no application messages have been exchanged; an open stream waiting for task events is not idle

### 2.3. Connection Termination

Connections are terminated using the WebSocket Close handshake ([RFC 6455 Section 7](https://datatracker.ietf.org/doc/html/rfc6455#section-7)):

- Either party **MAY** initiate a close by sending a Close frame
- The closing party **SHOULD** include a status code and optional reason string
- Active streams **MUST** be terminated when the connection closes
- The underlying tasks are **NOT** affected by connection closure; clients **MAY** reconnect and resubscribe (see [Section 8.5](#85-reconnection))

**WebSocket close codes used by this binding:**

| Close Code | Meaning                 | Usage                                                                                 |
| ---------- | ----------------------- | ------------------------------------------------------------------------------------- |
| `1000`     | Normal Closure          | Clean shutdown initiated by either party                                              |
| `1001`     | Going Away              | Server shutting down or client navigating away                                        |
| `1002`     | Protocol Error          | WebSocket framing violation                                                           |
| `1003`     | Unsupported Data        | Binary message received when text is expected                                         |
| `1007`     | Invalid Payload Data    | Text message is not valid UTF-8                                                       |
| `1008`     | Policy Violation        | Authorization denied or rate limit exceeded                                           |
| `1009`     | Message Too Big         | Message exceeds the server's maximum message size                                     |
| `1011`     | Internal Error          | Unexpected server-side error                                                          |
| `1013`     | Try Again Later         | Server is temporarily overloaded; client should retry                                 |
| `4000`     | A2A Protocol Error      | A2A-level protocol violation, such as repeated malformed messages (custom code)       |
| `4001`     | Authentication Required | Credentials missing after an unauthenticated upgrade, expired, or revoked (custom code) |
| `4002`     | Version Not Supported   | The connection's `A2A-Version` is not supported (custom code)                         |

Authentication failures *during* the upgrade are reported at the HTTP layer (`401 Unauthorized` or `403 Forbidden`) and never use a close code.

## 3. Message Framing

Each WebSocket text message contains **exactly one JSON-RPC 2.0 object**. Request, response, and error objects are the same objects defined by the A2A [JSON-RPC binding](https://a2a-protocol.org/v1.0.0/specification/#9-json-rpc-protocol-binding); the only difference is that they travel in a WebSocket text message rather than an HTTP body or SSE `data:` line. Every object **MUST** include the `"jsonrpc": "2.0"` member. Batch requests (JSON arrays) are not used by this binding.

Both peers send and receive JSON-RPC objects: the client sends A2A requests and the `ws/CancelStream` notification, and the server sends responses and the `ws/StreamEnd` and `ws/ReauthenticationRequired` notifications. The binding extensions are summarized at the start of this document.

### 3.1. Request Message Format

Clients send requests using the standard JSON-RPC 2.0 request object:

```json
{
  "jsonrpc": "2.0",
  "id": "<request-id>",
  "method": "<A2A-method-name>",
  "params": { /* method-specific request object */ },
  "serviceParams": { /* optional A2A service parameters */ }
}
```

| Field           | Type              | Required | Description                                                                                                                                                                                                 |
| --------------- | ----------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `jsonrpc`       | `string`          | Yes      | **MUST** be exactly `"2.0"`.                                                                                                                                                                                |
| `id`            | `string`/`number` | Yes      | Request identifier used to correlate responses (string UUID recommended). **MUST** be unique among the requests in flight on the connection and **SHOULD NOT** be reused within the connection. Every A2A request **MUST** carry an `id`; notifications are used only for the binding control messages listed in [Section 5](#5-method-inventory). |
| `method`        | `string`          | Yes      | The A2A method name (see [Section 5](#5-method-inventory)).                                                                                                                                                 |
| `params`        | `object`          | No       | The method-specific request parameters, serialized as ProtoJSON. Omitted for parameterless methods.                                                                                                         |
| `serviceParams` | `object`          | No       | **Binding extension.** A2A service parameters as a flat string key-value map (see [Section 4](#4-service-parameter-transmission)).                                                                          |

### 3.2. Response Message Format

Servers send successful responses using the standard JSON-RPC 2.0 response object:

```json
{
  "jsonrpc": "2.0",
  "id": "<request-id>",
  "result": { /* method-specific response object */ }
}
```

| Field     | Type              | Required | Description                                                                             |
| --------- | ----------------- | -------- | --------------------------------------------------------------------------------------- |
| `jsonrpc` | `string`          | Yes      | **MUST** be exactly `"2.0"`.                                                            |
| `id`      | `string`/`number` | Yes      | The request ID from the corresponding request message.                                  |
| `result`  | `object`          | Yes      | The method-specific response, serialized as ProtoJSON. Mutually exclusive with `error`. |

### 3.3. Error Response Format

Errors use the standard [JSON-RPC 2.0 error object](https://www.jsonrpc.org/specification#error_object), identical to the A2A JSON-RPC binding ([Section 9.5](https://a2a-protocol.org/v1.0.0/specification/#95-error-handling)):

```json
{
  "jsonrpc": "2.0",
  "id": "<request-id>",
  "error": {
    "code": -32001,
    "message": "<human-readable description>",
    "data": [ /* optional structured error details */ ]
  }
}
```

| Field           | Type              | Required | Description                                                                                                            |
| --------------- | ----------------- | -------- | ---------------------------------------------------------------------------------------------------------------------- |
| `jsonrpc`       | `string`          | Yes      | **MUST** be exactly `"2.0"`.                                                                                           |
| `id`            | `string`/`number` | Yes      | The request ID from the corresponding request. `null` if the request ID could not be parsed.                           |
| `error.code`    | `number`          | Yes      | The numeric JSON-RPC error code (see [Section 7](#7-error-handling)).                                                  |
| `error.message` | `string`          | Yes      | A human-readable error description.                                                                                    |
| `error.data`    | `array`           | No       | Optional array of objects, each with a `@type` key (`google.protobuf.Any` in ProtoJSON), for structured error context. |

A message that is not valid JSON **MUST** be answered with error code `-32700` and `"id": null`. Because each message is framed separately, a malformed message does not affect other requests and the connection **SHOULD** remain open. Servers **MAY** close the connection with close code `4000` if a client repeatedly sends malformed messages.

### 3.4. Streaming Messages

For streaming methods (`SendStreamingMessage`, `SubscribeToTask`), the server emits a **sequence of ordinary JSON-RPC 2.0 response objects that all share the originating request's `id`**, each carrying one `StreamResponse` in `result`. This is exactly the format the JSON-RPC binding uses for SSE events ([Section 9.4.2](https://a2a-protocol.org/v1.0.0/specification/#942-sendstreamingmessage)) - the payloads are identical; only the carrier (a WebSocket text message) differs.

```json
{
  "jsonrpc": "2.0",
  "id": "<request-id>",
  "result": { /* StreamResponse object */ }
}
```

If the agent responds with a `Message` rather than a `Task`, the stream carries exactly one `result` containing that `Message` and then ends, as required by [Section 3.1.2](https://a2a-protocol.org/v1.0.0/specification/#312-send-streaming-message) of the A2A specification.

**Stream end.** Because the WebSocket connection stays open after a stream completes (unlike an SSE response, which ends when the HTTP response closes), the server signals completion with a `ws/StreamEnd` notification whose `params.id` is the originating request's `id`:

```json
{
  "jsonrpc": "2.0",
  "method": "ws/StreamEnd",
  "params": { "id": "<request-id>" }
}
```

Stream errors are signaled using the standard error response ([Section 3.3](#33-error-response-format)) carrying the originating request `id`. No `ws/StreamEnd` follows an error.

**Stream cancellation.** A client **MAY** stop receiving a stream without closing the connection by sending a `ws/CancelStream` notification:

```json
{
  "jsonrpc": "2.0",
  "method": "ws/CancelStream",
  "params": { "id": "<request-id>" }
}
```

On receiving `ws/CancelStream`, the server **MUST** stop sending responses for that stream and send `ws/StreamEnd` for it. It **MUST NOT** cancel the underlying task; clients use `CancelTask` for that. A `ws/CancelStream` for an `id` that has no active stream is ignored.

### 3.5. Multiplexing

Carrying many JSON-RPC RPCs over a single connection requires correlating each response to its request. The binding does this with the JSON-RPC `id`:

- Each request **MUST** carry an `id` that is unique among the requests in flight on the connection
- Responses, streaming responses, and errors are correlated to their originating request by the `id` field; control notifications reference it in `params.id`
- Clients **MAY** send new requests while actively receiving streaming responses for other requests
- Servers **MUST** be prepared to handle multiple in-flight requests concurrently, and **MAY** limit how many are in flight per connection
- There is no ordering guarantee across different request IDs; within a single stream (same `id`), responses **MUST** be delivered in order

### 3.6. Message Size Limits

- Servers **SHOULD** enforce a maximum message size (recommended: 1 MiB per message, after reassembling fragmented frames)
- If a message exceeds the limit, the server **MUST** close the connection with close code `1009` (Message Too Big)
- Clients **SHOULD** avoid inlining large binary content and **SHOULD** use file references instead

## 4. Service Parameter Transmission

A2A service parameters defined in [Section 3.2.6 of the specification](https://a2a-protocol.org/v1.0.0/specification/#326-service-parameters) **MUST** be transmitted using one of two mechanisms. The HTTP JSON-RPC binding carries them as HTTP headers on each request; because individual WebSocket messages have no headers, this binding sends them either on the upgrade request (connection-scoped) or in the `serviceParams` member of a request (message-scoped).

The following rules apply to both mechanisms:

- Service parameter keys are **case-insensitive**, consistent with the A2A specification; receivers **MUST** compare keys case-insensitively
- Service parameter values are **case-sensitive** and **MUST** be UTF-8 encoded strings
- Multiple values for the same service parameter (e.g., `A2A-Extensions`) **SHOULD** be comma-separated in a single entry
- There is no prescribed size limit, but implementations **SHOULD** keep service parameters compact
- All keys prefixed with `a2a-` are reserved for A2A service parameters

### 4.1. During Connection Upgrade (Connection-Scoped)

Service parameters that apply to all requests on a connection are transmitted on the WebSocket upgrade request, as HTTP headers:

```
GET /a2a/ws HTTP/1.1
Upgrade: websocket
A2A-Version: 1.0
A2A-Extensions: https://example.com/extensions/geolocation/v1
```

Clients that cannot set handshake headers (see [Section 2.1.1](#211-clients-that-cannot-set-handshake-headers)) send them as URL query parameters instead, with values percent-encoded:

```
wss://agent.example.com/a2a/ws?A2A-Version=1.0&A2A-Extensions=https%3A%2F%2Fexample.com%2Fextensions%2Fgeolocation%2Fv1
```

If the same parameter is sent both as a header and as a query parameter, the header value takes precedence. Connection-scoped service parameters apply to the entire connection lifetime unless overridden per-request.

**Protocol version.** `A2A-Version` is connection-scoped, and every request on the connection uses that version:

- The client **MUST** send `A2A-Version` on the upgrade request, as required by [Section 3.6.1](https://a2a-protocol.org/v1.0.0/specification/#361-client-responsibilities) of the A2A specification
- If `A2A-Version` is absent or empty, the server **MUST** treat the connection as version `0.3`, as required by [Section 3.6.2](https://a2a-protocol.org/v1.0.0/specification/#362-server-responsibilities)
- If the version is not supported, the server **SHOULD** reject the upgrade with HTTP `400 Bad Request`. To reach clients that cannot observe the HTTP response, it **MAY** instead complete the upgrade and immediately close the connection with close code `4002`
- A per-request `a2a-version` in `serviceParams`, if present, **MUST** equal the connection's version; otherwise the server **MUST** respond with `VersionNotSupportedError` (`-32009`) for that request

### 4.2. Per-Request (Message-Scoped)

Service parameters that vary per-request **MUST** be transmitted in the `serviceParams` member of the request object:

```json
{
  "jsonrpc": "2.0",
  "id": "req-1",
  "method": "SendMessage",
  "params": { /* ... */ },
  "serviceParams": {
    "a2a-extensions": "https://example.com/extensions/geolocation/v1"
  }
}
```

**Service Parameter Requirements:**

- `serviceParams` is a flat string key-value map, added as a member of the JSON-RPC request object
- Keys **SHOULD** be lowercase for consistency with HTTP header conventions
- A `serviceParams` object **MUST NOT** contain two keys that differ only in case; the server **MUST** reject such a request with `InvalidRequestError` (`-32600`)
- Per-request service parameters **override** connection-scoped parameters with the same key for that request, except for `a2a-version` (see [Section 4.1](#41-during-connection-upgrade-connection-scoped))

**Reserved Keys:**

This binding does not reserve any service parameter keys beyond the `a2a-` prefix reserved by the A2A specification. Method names prefixed with `ws/` are reserved for this binding's control messages.

## 5. Method Inventory

The binding supports the full A2A method inventory. Method names use PascalCase, matching the JSON-RPC and gRPC bindings. The table below reflects the `A2AService` methods as defined in the current A2A proto (`specification/a2a.proto`). Implementations targeting an older version of A2A **MUST** use the method names, request types, and response types from the proto for that version.

| Method Name                        | Type          | Request Params Type                       | Response Result Type                      | A2A Operation                   |
| ---------------------------------- | ------------- | ----------------------------------------- | ----------------------------------------- | ------------------------------- |
| `SendMessage`                      | Request/Reply | `SendMessageRequest`                      | `SendMessageResponse`                     | Send Message                    |
| `SendStreamingMessage`             | Streaming     | `SendMessageRequest`                      | `StreamResponse` (stream of events)       | Send Streaming Message          |
| `GetTask`                          | Request/Reply | `GetTaskRequest`                          | `Task`                                    | Get Task                        |
| `ListTasks`                        | Request/Reply | `ListTasksRequest`                        | `ListTasksResponse`                       | List Tasks                      |
| `CancelTask`                       | Request/Reply | `CancelTaskRequest`                       | `Task`                                    | Cancel Task                     |
| `SubscribeToTask`                  | Streaming     | `SubscribeToTaskRequest`                  | `StreamResponse` (stream of events)       | Subscribe to Task               |
| `CreateTaskPushNotificationConfig` | Request/Reply | `TaskPushNotificationConfig`              | `TaskPushNotificationConfig`              | Create Push Notification Config |
| `GetTaskPushNotificationConfig`    | Request/Reply | `GetTaskPushNotificationConfigRequest`    | `TaskPushNotificationConfig`              | Get Push Notification Config    |
| `ListTaskPushNotificationConfigs`  | Request/Reply | `ListTaskPushNotificationConfigsRequest`  | `ListTaskPushNotificationConfigsResponse` | List Push Notification Configs  |
| `DeleteTaskPushNotificationConfig` | Request/Reply | `DeleteTaskPushNotificationConfigRequest` | Empty object `{}`                         | Delete Push Notification Config |
| `GetExtendedAgentCard`             | Request/Reply | `GetExtendedAgentCardRequest`             | `AgentCard`                               | Get Extended Agent Card         |

**Tenant routing.** When the selected `AgentInterface` in the Agent Card declares a `tenant`, clients **MUST** set the `tenant` field in the `params` of every request to exactly that value, and omit it otherwise, as defined for the `AgentInterface.tenant` field in `specification/a2a.proto`.

**Binding control methods.** In addition to the A2A operations, the binding defines the following methods. They are not A2A operations and use the reserved `ws/` prefix so they cannot collide with current or future A2A method names:

| Method Name                   | Kind                        | Direction        | Required | Purpose                                                            |
| ----------------------------- | --------------------------- | ---------------- | -------- | ------------------------------------------------------------------ |
| `ws/StreamEnd`                | Notification                | Server → Client  | Yes      | Terminates a stream ([Section 3.4](#34-streaming-messages))        |
| `ws/CancelStream`             | Notification                | Client → Server  | Yes      | Stops a stream without cancelling its task ([Section 3.4](#34-streaming-messages)) |
| `ws/ReauthenticationRequired` | Notification                | Server → Client  | No       | Asks the client to refresh its credentials ([Section 9](#9-authentication-and-authorization)) |
| `ws/Authenticate`             | Request/Reply               | Client → Server  | No       | Authenticates or refreshes credentials in-band ([Section 9](#9-authentication-and-authorization)) |

Bidirectional streaming (for example, the `SendLiveMessage` method defined by the SLIMRPC binding) is not part of the A2A core proto and is out of scope for this version of the binding.

## 6. Core Methods

The behavior, inputs, outputs, and errors of each operation are defined in [Section 3.1](https://a2a-protocol.org/v1.0.0/specification/#31-core-operations) of the A2A specification, and request and response bodies are identical to the JSON-RPC binding. This section lists each method with its types and any rules specific to this binding. Because every request is correlated by `id`, a long-running request (for example, a blocking `SendMessage`) does not prevent the client from sending or receiving other requests on the same connection.

### 6.1. SendMessage

Sends a message to initiate or continue a task.

**Request:** [`SendMessageRequest`](https://a2a-protocol.org/v1.0.0/specification/#321-sendmessagerequest)

**Response:** [`SendMessageResponse`](https://a2a-protocol.org/v1.0.0/specification/#311-send-message), containing either a `Task` or a `Message`

**Example:**

```json
{
  "jsonrpc": "2.0",
  "id": "req-1",
  "method": "SendMessage",
  "params": {
    "message": {
      "messageId": "msg-uuid",
      "role": "ROLE_USER",
      "parts": [{ "text": "Plan a trip to Paris" }]
    }
  }
}
```

```json
{
  "jsonrpc": "2.0",
  "id": "req-1",
  "result": {
    "task": {
      "id": "task-uuid",
      "contextId": "ctx-uuid",
      "status": { "state": "TASK_STATE_WORKING" }
    }
  }
}
```

### 6.2. SendStreamingMessage

Sends a message and returns a stream of update events. Requires `capabilities.streaming = true`.

**Request:** [`SendMessageRequest`](https://a2a-protocol.org/v1.0.0/specification/#321-sendmessagerequest)

**Response:** A sequence of responses sharing the request `id`, each carrying a [`StreamResponse`](https://a2a-protocol.org/v1.0.0/specification/#323-stream-response) in `result`, followed by `ws/StreamEnd` ([Section 3.4](#34-streaming-messages))

**Example stream** for a request with `"id": "req-2"`:

```json
{"jsonrpc": "2.0", "id": "req-2", "result": {"task": {"id": "task-uuid", "contextId": "ctx-uuid", "status": {"state": "TASK_STATE_WORKING"}}}}
{"jsonrpc": "2.0", "id": "req-2", "result": {"artifactUpdate": {"taskId": "task-uuid", "contextId": "ctx-uuid", "artifact": {"artifactId": "artifact-uuid", "parts": [{"text": "Summary..."}]}}}}
{"jsonrpc": "2.0", "id": "req-2", "result": {"statusUpdate": {"taskId": "task-uuid", "contextId": "ctx-uuid", "status": {"state": "TASK_STATE_COMPLETED"}}}}
{"jsonrpc": "2.0", "method": "ws/StreamEnd", "params": {"id": "req-2"}}
```

Each line is a separate WebSocket text message. If the agent responds with a `Message` instead of a `Task`, the stream carries that single `Message`, followed by `ws/StreamEnd`.

### 6.3. GetTask

Retrieves the current state of a task.

**Request:** [`GetTaskRequest`](https://a2a-protocol.org/v1.0.0/specification/#313-get-task)

**Response:** [`Task`](https://a2a-protocol.org/v1.0.0/specification/#411-task)

### 6.4. ListTasks

Lists tasks visible to the authenticated identity, with optional filtering and pagination.

**Request:** [`ListTasksRequest`](https://a2a-protocol.org/v1.0.0/specification/#314-list-tasks)

**Response:** [`ListTasksResponse`](https://a2a-protocol.org/v1.0.0/specification/#314-list-tasks)

### 6.5. CancelTask

Requests cancellation of an ongoing task.

**Request:** [`CancelTaskRequest`](https://a2a-protocol.org/v1.0.0/specification/#315-cancel-task)

**Response:** [`Task`](https://a2a-protocol.org/v1.0.0/specification/#411-task)

`CancelTask` cancels the task itself, whereas `ws/CancelStream` only stops one stream and leaves the task running ([Section 3.4](#34-streaming-messages)). Active streams for a canceled task receive the final `TASK_STATE_CANCELED` status update, followed by `ws/StreamEnd`.

### 6.6. SubscribeToTask

Subscribes to streaming updates for an existing task. Requires `capabilities.streaming = true`.

**Request:** [`SubscribeToTaskRequest`](https://a2a-protocol.org/v1.0.0/specification/#316-subscribe-to-task)

**Response:** A stream of [`StreamResponse`](https://a2a-protocol.org/v1.0.0/specification/#323-stream-response) objects, framed as for `SendStreamingMessage`. The first event is the `Task` in its current state

If the task is in a terminal state, the server responds with `UnsupportedOperationError` (`-32004`) and no stream is opened. Clients also use this method to resume streams after reconnecting ([Section 8.5](#85-reconnection)).

### 6.7. Push Notification Configuration Methods

These methods require `capabilities.pushNotifications = true`; otherwise the server responds with `PushNotificationNotSupportedError` (`-32003`). Push notifications themselves are delivered by HTTP POST to the configured webhook, not over the WebSocket connection ([Section 3.5](https://a2a-protocol.org/v1.0.0/specification/#35-task-update-delivery-mechanisms) of the A2A specification).

#### 6.7.1. CreateTaskPushNotificationConfig

Creates a push notification configuration for a task.

**Request:** [`TaskPushNotificationConfig`](https://a2a-protocol.org/v1.0.0/specification/#431-pushnotificationconfig)

**Response:** [`TaskPushNotificationConfig`](https://a2a-protocol.org/v1.0.0/specification/#431-pushnotificationconfig), with the server-assigned `id`

#### 6.7.2. GetTaskPushNotificationConfig

Retrieves a push notification configuration.

**Request:** [`GetTaskPushNotificationConfigRequest`](https://a2a-protocol.org/v1.0.0/specification/#318-get-push-notification-config)

**Response:** [`TaskPushNotificationConfig`](https://a2a-protocol.org/v1.0.0/specification/#431-pushnotificationconfig)

#### 6.7.3. ListTaskPushNotificationConfigs

Lists the push notification configurations for a task.

**Request:** [`ListTaskPushNotificationConfigsRequest`](https://a2a-protocol.org/v1.0.0/specification/#319-list-push-notification-configs)

**Response:** [`ListTaskPushNotificationConfigsResponse`](https://a2a-protocol.org/v1.0.0/specification/#319-list-push-notification-configs)

#### 6.7.4. DeleteTaskPushNotificationConfig

Removes a push notification configuration.

**Request:** [`DeleteTaskPushNotificationConfigRequest`](https://a2a-protocol.org/v1.0.0/specification/#3110-delete-push-notification-config)

**Response:** Empty object `{}`, the ProtoJSON form of `google.protobuf.Empty`

### 6.8. GetExtendedAgentCard

Retrieves the agent's extended Agent Card. Requires `capabilities.extendedAgentCard = true`.

**Request:** [`GetExtendedAgentCardRequest`](https://a2a-protocol.org/v1.0.0/specification/#3111-get-extended-agent-card). `params` may be omitted when no `tenant` is set

**Response:** [`AgentCard`](https://a2a-protocol.org/v1.0.0/specification/#441-agentcard)

The server uses the connection's authenticated identity ([Section 9](#9-authentication-and-authorization)). On a connection that has not authenticated yet, it responds with `-32000` and reason `WS_UNAUTHENTICATED` ([Section 7.3](#73-binding-specific-errors)).

## 7. Error Handling

The binding uses the error model of the A2A JSON-RPC binding unchanged. Every A2A error is a JSON-RPC 2.0 error object correlated to its request by `id`, and the numeric codes are the same as in [Section 9.5](https://a2a-protocol.org/v1.0.0/specification/#95-error-handling) of the A2A specification. The binding adds rules for two situations that do not arise over HTTP: errors that affect the whole connection, and errors on a stream that shares the connection with other requests.

### 7.1. Error Response Structure

Errors use the JSON-RPC error object defined in [Section 3.3](#33-error-response-format). It carries the three elements required by [Section 3.3.2](https://a2a-protocol.org/v1.0.0/specification/#332-error-handling) of the A2A specification:

- **Error Code:** `error.code`, a numeric JSON-RPC error code
- **Error Message:** `error.message`, a human-readable description
- **Error Details:** `error.data`, an optional array of objects, each with a `@type` key using the ProtoJSON `Any` representation. Implementations **SHOULD** use well-known types such as `google.rpc.ErrorInfo` and `google.rpc.BadRequest`

**Example A2A error:**

```json
{
  "jsonrpc": "2.0",
  "id": "req-3",
  "error": {
    "code": -32001,
    "message": "Task not found",
    "data": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "TASK_NOT_FOUND",
        "domain": "a2a-protocol.org",
        "metadata": { "taskId": "nonexistent-task-id" }
      }
    ]
  }
}
```

### 7.2. A2A Error Code Mapping

All A2A errors defined in [Section 3.3.2](https://a2a-protocol.org/v1.0.0/specification/#332-error-handling) of the A2A specification **MUST** be mapped to the JSON-RPC codes below, which are the canonical mappings from [Section 5.4](https://a2a-protocol.org/v1.0.0/specification/#54-error-code-mappings) and [Section 9.5](https://a2a-protocol.org/v1.0.0/specification/#95-error-handling). Each error applies only to the request that caused it; the connection and any other requests in flight are unaffected.

| A2A Error Type                        | JSON-RPC `error.code` | Notes                                                         |
| ------------------------------------- | --------------------- | ------------------------------------------------------------- |
| `JSONParseError`                      | `-32700`              | Message is not valid JSON; sent with `"id": null`             |
| `InvalidRequestError`                 | `-32600`              | Object is not a valid JSON-RPC request                        |
| `MethodNotFoundError`                 | `-32601`              | Requested method does not exist, including unknown `ws/` methods |
| `InvalidParamsError`                  | `-32602`              | Method parameters are invalid                                 |
| `InternalError`                       | `-32603`              | Server-side internal error                                    |
| `TaskNotFoundError`                   | `-32001`              | Task ID does not exist or is not accessible                   |
| `TaskNotCancelableError`              | `-32002`              | Task is in a non-cancelable state                             |
| `PushNotificationNotSupportedError`   | `-32003`              | Agent does not support push notifications                     |
| `UnsupportedOperationError`           | `-32004`              | Operation not supported by the agent                          |
| `ContentTypeNotSupportedError`        | `-32005`              | Content type not accepted by the agent                        |
| `InvalidAgentResponseError`           | `-32006`              | Agent produced an unexpected response                         |
| `ExtendedAgentCardNotConfiguredError` | `-32007`              | Extended Agent Card is not configured                         |
| `ExtensionSupportRequiredError`       | `-32008`              | Required extension is not supported by the client             |
| `VersionNotSupportedError`            | `-32009`              | Per-request `a2a-version` does not match the connection's version |

When the version requested for the whole connection is not supported, the failure is reported on the upgrade instead (HTTP `400`, or close code `4002`), as described in [Section 4.1](#41-during-connection-upgrade-connection-scoped).

### 7.3. Binding-Specific Errors

Errors introduced by this binding use code `-32000`, from the JSON-RPC range reserved for implementation-defined server errors. Codes `-32001` to `-32099` are reserved for A2A errors and **MUST NOT** be used by this binding. Because all binding errors share one code, the server **MUST** include a `google.rpc.ErrorInfo` entry in `error.data` with `domain` set to `a2a-protocol.org` and one of the following `reason` values, so that clients can tell them apart:

| `reason`                  | Meaning                                                                                       | Client action                                       |
| ------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| `WS_UNAUTHENTICATED`      | The connection has not authenticated yet (see [Section 2.1.1](#211-clients-that-cannot-set-handshake-headers)), or the credentials in a `ws/Authenticate` request were rejected | Send `ws/Authenticate` with valid credentials, then retry |
| `WS_CREDENTIALS_EXPIRED`  | The connection's credentials have expired or been revoked                                     | Refresh credentials or reconnect, then retry        |
| `WS_PERMISSION_DENIED`    | The authenticated identity is not permitted to perform the request ([Section 9.6](#96-authorization)) | Do not retry with the same identity                 |
| `WS_RATE_LIMITED`         | The client exceeded a rate limit                                                              | Retry after the delay in `google.rpc.RetryInfo`, if present |
| `WS_TOO_MANY_IN_FLIGHT`   | The client exceeded the server's limit on concurrent requests or streams per connection      | Wait for in-flight requests to finish, then retry   |

**Example binding error:**

```json
{
  "jsonrpc": "2.0",
  "id": "req-12",
  "error": {
    "code": -32000,
    "message": "Rate limit exceeded",
    "data": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "WS_RATE_LIMITED",
        "domain": "a2a-protocol.org"
      },
      {
        "@type": "type.googleapis.com/google.rpc.RetryInfo",
        "retryDelay": "2s"
      }
    ]
  }
}
```

### 7.4. Connection-Level Errors

Some failures cannot be reported against a single request, or make the connection unusable. These are reported with the WebSocket close codes listed in [Section 2.3](#23-connection-termination):

| Condition                                                   | Server behavior                                                                                                 |
| ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Message is not valid UTF-8                                  | Close with `1007`; no JSON-RPC error is sent because the message cannot be decoded                             |
| Binary message received                                     | Close with `1003`                                                                                               |
| Message exceeds the maximum size                            | Close with `1009` ([Section 3.6](#36-message-size-limits))                                                     |
| Message is not valid JSON                                   | Respond with `-32700` and keep the connection open; **MAY** close with `4000` if it happens repeatedly ([Section 3.3](#33-error-response-format)) |
| Credentials expire or are revoked                           | Respond to each in-flight request with `-32000` (`WS_CREDENTIALS_EXPIRED`), then close with `4001`             |
| Rate limit repeatedly exceeded after `WS_RATE_LIMITED` errors | Close with `1008`                                                                                             |
| Unexpected server failure affecting the whole connection    | Close with `1011`                                                                                               |

Before closing the connection for any reason other than a decoding or framing failure, the server **SHOULD** send an error response for each request still in flight, so clients can tell which requests did not complete.

**Requests in flight when the connection closes.** If the connection closes before a response arrives, the client **MUST** treat the request as failed with an unknown outcome: the server may or may not have processed it. Clients **MAY** safely retry read operations (`GetTask`, `ListTasks`, `GetExtendedAgentCard`). Before retrying `SendMessage` or `SendStreamingMessage`, clients **SHOULD** check the task state with `GetTask` or resubscribe with `SubscribeToTask` where a task ID is known, and **SHOULD** reuse the same `messageId` so the agent can detect the duplicate (see [Section 3.3.1](https://a2a-protocol.org/v1.0.0/specification/#331-idempotency) of the A2A specification).

### 7.5. Stream Errors

An error during an active stream is sent as a JSON-RPC error object carrying the stream's request `id`. It ends the stream: the server **MUST NOT** send further responses for that `id`, and no `ws/StreamEnd` follows the error. Other streams and requests on the connection are unaffected.

```json
{
  "jsonrpc": "2.0",
  "id": "req-2",
  "error": {
    "code": -32603,
    "message": "Stream processing failed unexpectedly",
    "data": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "INTERNAL",
        "domain": "a2a-protocol.org"
      }
    ]
  }
}
```

A stream error means event delivery failed; it does not by itself mean the task failed. A task that fails is reported as a normal `statusUpdate` with `TASK_STATE_FAILED`, followed by `ws/StreamEnd`. After a stream error, clients **MAY** use `GetTask` or `SubscribeToTask` to learn the task's current state.

### 7.6. Errors and Control Messages

- Notifications never receive a response, as required by JSON-RPC 2.0. A server **MUST NOT** send an error in reply to a `ws/CancelStream` notification, even if it is malformed or refers to an unknown `id`; it ignores such notifications
- A client **MUST** ignore a `ws/StreamEnd` notification for an `id` that has no active stream
- A request for a `ws/` method the server does not implement, such as the optional `ws/Authenticate`, receives `MethodNotFoundError` (`-32601`)
- Unknown notifications from either peer, including unknown `ws/` notifications, **MUST** be ignored

## 8. Streaming

The binding supports A2A's server-streaming operations, `SendStreamingMessage` and `SubscribeToTask`, with the same server-to-client streaming model as the other bindings. Each stream is identified by the `id` of the request that opened it and is delivered as a sequence of JSON-RPC responses on the shared connection. The message formats are defined in [Section 3.4](#34-streaming-messages); this section defines stream behavior.

The binding does not define any client-to-server streaming operation. Each client turn, such as a reply to `TASK_STATE_INPUT_REQUIRED`, is sent as its own `SendMessage` or `SendStreamingMessage` request, as in the other bindings.

### 8.1. Stream Mechanism

1. The client sends a request for a streaming method with a new `id`
2. The server sends one or more responses with that `id`, each carrying a `StreamResponse` in `result`
3. The server ends the stream with a `ws/StreamEnd` notification referencing that `id`, or with a JSON-RPC error for that `id` ([Section 7.5](#75-stream-errors))

A stream is active from the moment the server receives the request until it sends `ws/StreamEnd` or an error for its `id`, or the connection closes. Several streams may be active on one connection at the same time, and responses from different streams may be interleaved ([Section 3.5](#35-multiplexing)).

### 8.2. Event Ordering

Within a stream, the server **MUST** send events in the order they were generated and **MUST NOT** reorder them, as required by [Section 3.5.2](https://a2a-protocol.org/v1.0.0/specification/#352-streaming-event-delivery) of the A2A specification. No ordering is defined between different streams.

A stream follows one of these patterns:

- **Task lifecycle stream:** an initial `Task`, followed by zero or more `TaskStatusUpdateEvent` or `TaskArtifactUpdateEvent` updates. The stream **MUST** end after a status update with a terminal state (`TASK_STATE_COMPLETED`, `TASK_STATE_FAILED`, `TASK_STATE_CANCELED`, `TASK_STATE_REJECTED`), as required by [Section 3.1.2](https://a2a-protocol.org/v1.0.0/specification/#312-send-streaming-message) of the A2A specification
- **Message-only stream** (`SendStreamingMessage` only): exactly one `Message`

When a task enters an interrupted state, the server **MAY** end the stream after that status update. The two interrupted states differ:

- `TASK_STATE_INPUT_REQUIRED`: the task cannot continue until the client sends another message, so the server usually ends the stream. The client's reply opens a new stream
- `TASK_STATE_AUTH_REQUIRED`: if the agent expects to receive the credential out of band, it **SHOULD** keep the stream open, so the client sees the task resume without a follow-up message, as described in [Section 7.6.1](https://a2a-protocol.org/v1.0.0/specification/#761-in-task-authorization-agent-responsibilities) of the A2A specification

For `SubscribeToTask`, the first event **MUST** be the `Task` in its current state, so that no updates are lost between a `GetTask` call and the subscription.

### 8.3. Multiple Streams for the Same Task

An agent **MAY** serve several streams for the same task, on the same connection or on different connections, as permitted by [Section 3.5.2](https://a2a-protocol.org/v1.0.0/specification/#352-streaming-event-delivery) of the A2A specification. When it does:

- Every active stream for the task **MUST** receive the same events in the same order
- Ending one stream, whether by `ws/CancelStream`, error, or connection close, **MUST NOT** affect other streams for the task
- The task's lifecycle is independent of any individual stream

For example, a client that calls `SubscribeToTask` for a task it is already streaming on the same connection receives a second, independent stream with its own `id`.

### 8.4. Stream Termination

A stream ends in one of the following ways:

| Termination                | Signaled by                                                  | Task affected?            |
| -------------------------- | ------------------------------------------------------------ | ------------------------- |
| **Normal completion**      | Final event (terminal state, an interrupted state as allowed by [Section 8.2](#82-event-ordering), or the single `Message`), then `ws/StreamEnd` | No                        |
| **Client cancellation**    | Client sends `ws/CancelStream`; server stops sending events and sends `ws/StreamEnd` | No; use `CancelTask` to cancel the task |
| **Server-initiated end**   | `ws/StreamEnd` before a terminal state, other than after an interrupted state (for example, during server shutdown or when a stream limit is reached) | No                        |
| **Stream error**           | JSON-RPC error for the stream's `id`; no `ws/StreamEnd` follows | Not necessarily ([Section 7.5](#75-stream-errors)) |
| **Connection close**       | WebSocket Close frame or connection loss; all active streams end implicitly | No                        |

If a stream for a task ends before a terminal state was received, the task has not finished; if the last state received was not an interrupted state, it is still running. A client that still needs its updates **SHOULD** call `SubscribeToTask`. Servers **SHOULD** send `ws/StreamEnd` for every active stream before closing a connection normally.

### 8.5. Reconnection

If the connection is lost, the underlying tasks continue running on the server. To resume, clients **SHOULD**:

1. Open a new connection, repeating the handshake, version negotiation, and authentication ([Section 2.1](#21-connection-establishment))
2. Call `SubscribeToTask` for each task that was still in progress, using the task IDs from earlier responses. Request `id` values from the previous connection have no meaning on the new one
3. Rebuild their view of each task from the `Task` returned as the first event of each subscription, since events sent while disconnected are not replayed
4. Tolerate duplicate events that may result from the overlap between the old and new streams

A message-only stream cannot be resumed because it has no task. A client that did not receive the `Message` **MAY** resend the request, reusing the same `messageId` ([Section 7.4](#74-connection-level-errors)).

**Retry behavior.** Clients **SHOULD** reconnect with exponential backoff and jitter (recommended: starting at 1 second, with a maximum of 30 seconds). The close code determines whether reconnecting can help:

| Close Code                             | Client behavior                                                         |
| -------------------------------------- | ----------------------------------------------------------------------- |
| `1001`, `1006` (abnormal), `1011`      | Reconnect with backoff                                                  |
| `1013`                                 | Reconnect with backoff, waiting at least the initial delay              |
| `4001`                                 | Obtain fresh credentials, then reconnect                                |
| `1008`                                 | Do not reconnect until the cause (authorization or rate limiting) is resolved |
| `1002`, `1003`, `1007`, `1009`, `4000`, `4002` | Do not reconnect with the same behavior; the client must change what it sends first |

### 8.6. Flow Control

All streams on a connection share one underlying TCP connection, so a client that reads slowly delays every stream on it. To keep one stream from starving the others:

- Servers **SHOULD** interleave responses from different streams fairly rather than draining one stream at a time
- Servers **SHOULD** limit how many events they buffer per stream and per connection
- If a client does not keep up, a server **MAY** end the slowest streams with `ws/StreamEnd` rather than buffering without limit; the client can resume them with `SubscribeToTask`
- Servers **MAY** limit the number of active streams per connection; requests beyond the limit receive `-32000` with reason `WS_TOO_MANY_IN_FLIGHT` ([Section 7.3](#73-binding-specific-errors))

### 8.7. Capability Declaration

Agents that support streaming **MUST** declare `capabilities.streaming = true` in their Agent Card. If streaming is not declared, the server **MUST** respond to `SendStreamingMessage` and `SubscribeToTask` with `UnsupportedOperationError` (`-32004`), as required by [Section 3.3.4](https://a2a-protocol.org/v1.0.0/specification/#334-capability-validation) of the A2A specification.

## 9. Authentication and Authorization

The binding uses the authentication schemes the agent declares in its Agent Card `securitySchemes` and `securityRequirements`, as defined in [Section 7](https://a2a-protocol.org/v1.0.0/specification/#7-authentication-and-authorization) of the A2A specification. It does not define new schemes; it defines how their credentials travel over a WebSocket connection.

The A2A specification requires that credentials accompany every request and that the server authenticate every request ([Sections 7.3 and 7.4](https://a2a-protocol.org/v1.0.0/specification/#73-client-authentication-process)). Individual WebSocket messages have no headers, so this binding attaches credentials to the **connection** instead: the client presents them once, during the handshake or with `ws/Authenticate`, and the server authenticates every request against the connection's current credentials. The requirements of [Section 12.6](https://a2a-protocol.org/v1.0.0/specification/#126-authentication-and-authorization) of the A2A specification are covered as follows:

- **Standard schemes and credential transmission:** [Section 9.1](#91-credential-transmission)
- **Authentication challenges:** [Sections 9.2](#92-authentication-during-the-handshake), [9.3](#93-in-band-authentication-with-wsauthenticate), and [9.5](#95-credential-refresh)
- **Transport security practices:** [Sections 9.4](#94-connection-authentication-context) and [9.7](#97-origin-validation)

### 9.1. Credential Transmission

Each scheme type defined by the A2A specification ([Section 4.5](https://a2a-protocol.org/v1.0.0/specification/#45-security-objects)) is carried as follows:

| Security Scheme                                  | On the upgrade request                                   | For clients that cannot set handshake headers          | With `ws/Authenticate` |
| ------------------------------------------------ | -------------------------------------------------------- | ------------------------------------------------------ | ---------------------- |
| `HTTPAuthSecurityScheme` (e.g., `Bearer`, `Basic`) | `Authorization: <scheme> <credentials>`                  | `ws/Authenticate`                                      | Yes                    |
| `APIKeySecurityScheme`, `location: header`       | The header named in `name`                               | `ws/Authenticate`                                      | Yes                    |
| `APIKeySecurityScheme`, `location: cookie`       | The cookie named in `name`                               | The cookie, sent automatically by the browser          | Yes                    |
| `APIKeySecurityScheme`, `location: query`        | The query parameter named in `name`                      | The query parameter named in `name` (discouraged)      | Yes                    |
| `OAuth2SecurityScheme`                           | `Authorization: Bearer <access-token>`                   | `ws/Authenticate`                                      | Yes                    |
| `OpenIdConnectSecurityScheme`                    | `Authorization: Bearer <token>`                          | `ws/Authenticate`                                      | Yes                    |
| `MutualTlsSecurityScheme`                        | Client certificate in the TLS handshake                  | Client certificate selected by the browser             | No                     |

The following rules apply to all schemes:

- Credentials **MUST** only be sent over `wss://` ([Section 1](#1-protocol-requirements))
- Credentials **MUST NOT** be sent in `serviceParams`, and servers **MUST NOT** use `serviceParams` values for authentication. A connection has exactly one set of credentials at a time
- Credentials **MUST NOT** be sent in the `Sec-WebSocket-Protocol` header. That header is echoed in the 101 response and is commonly logged
- Credentials **SHOULD NOT** be sent in the URL. Agents **SHOULD NOT** declare `location: query` API keys for this binding; servers that accept them **MUST** redact them from logs
- Servers **MUST NOT** include credentials in error messages, close reasons, or logs

### 9.2. Authentication During the Handshake

Clients that can set handshake headers **SHOULD** send credentials on the upgrade request. The server validates them before completing the upgrade:

1. The client reads `securitySchemes` and `securityRequirements` from the Agent Card and obtains credentials out of band
2. The client sends the credentials on the upgrade request, as listed in [Section 9.1](#91-credential-transmission)
3. The server **MUST** validate the credentials before sending the 101 response
4. If credentials are invalid, the server **MUST** reject the upgrade with HTTP `401 Unauthorized`. If they are missing, it **MUST** do the same, unless it supports unauthenticated upgrades ([Section 2.1.1](#211-clients-that-cannot-set-handshake-headers)). The response **SHOULD** include a `WWW-Authenticate` header naming the required scheme (for bearer tokens, as defined in [RFC 6750 Section 3](https://datatracker.ietf.org/doc/html/rfc6750#section-3))
5. If the credentials are valid but the identity may not use the agent at all, the server **MUST** reject the upgrade with HTTP `403 Forbidden`

**Example rejected upgrade:**

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="agent.example.com", error="invalid_token", error_description="The access token expired"
Content-Length: 0
```

### 9.3. In-Band Authentication with `ws/Authenticate`

`ws/Authenticate` is an **optional** JSON-RPC request that presents credentials over an open connection. It serves two purposes:

- **Initial authentication** for clients that cannot set handshake headers ([Section 2.1.1](#211-clients-that-cannot-set-handshake-headers)). A server that accepts unauthenticated upgrades **MUST** implement `ws/Authenticate`
- **Credential refresh** on a long-lived connection, before the current credentials expire ([Section 9.5](#95-credential-refresh))

**Request:**

```json
{
  "jsonrpc": "2.0",
  "id": "auth-1",
  "method": "ws/Authenticate",
  "params": {
    "credentials": {
      "oauth": "<access-token>"
    }
  }
}
```

`params.credentials` is a map from the name of a scheme in the Agent Card `securitySchemes` to the credential for that scheme. The credential is the value the client would otherwise send on the upgrade request:

- For `HTTPAuthSecurityScheme`, the value that follows the scheme name in the `Authorization` header
- For `APIKeySecurityScheme`, the API key
- For `OAuth2SecurityScheme` and `OpenIdConnectSecurityScheme`, the token sent as the bearer token

The map **MAY** contain several entries, so that a client can satisfy a `securityRequirements` entry that requires more than one scheme. Together with any client certificate from the TLS handshake, the credentials **MUST** satisfy at least one entry of `securityRequirements`.

**Successful response:**

```json
{
  "jsonrpc": "2.0",
  "id": "auth-1",
  "result": {
    "expiresAt": "2026-11-10T10:00:00.000Z"
  }
}
```

`result.expiresAt` is optional. When present, it is the time, as a ProtoJSON `Timestamp`, after which the server will no longer accept the connection's credentials. It may be earlier than the token's own expiry if server policy limits it.

**Processing rules:**

- On success, the server **MUST** use the new credentials for every request it receives after sending the response. Active streams continue and are authorized under the new credentials
- When refreshing, the new credentials **MUST** identify the same principal as the current ones. If they do not, the server **MUST** reject the request with `-32000` and reason `WS_PERMISSION_DENIED` and keep the current credentials. A client that needs to act as another identity **MUST** open a new connection
- If the credentials are invalid, the server **MUST** respond with `-32000` and reason `WS_UNAUTHENTICATED`. A failed refresh does not affect the current credentials, which remain valid until they expire
- If `params.credentials` names a scheme that is not in `securitySchemes`, or names a `MutualTlsSecurityScheme`, the server **MUST** respond with `InvalidParamsError` (`-32602`)
- Servers **MUST** limit failed `ws/Authenticate` attempts per connection (recommended: 3) and close the connection with close code `1008` when the limit is exceeded
- A client **MUST** wait for the response to its initial `ws/Authenticate` before sending other requests; requests sent earlier receive `WS_UNAUTHENTICATED` ([Section 2.1.1](#211-clients-that-cannot-set-handshake-headers))
- A server that does not implement `ws/Authenticate` responds with `MethodNotFoundError` (`-32601`) ([Section 7.6](#76-errors-and-control-messages)). This is how clients discover whether in-band refresh is available

**Example failure:**

```json
{
  "jsonrpc": "2.0",
  "id": "auth-1",
  "error": {
    "code": -32000,
    "message": "Invalid credentials",
    "data": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "WS_UNAUTHENTICATED",
        "domain": "a2a-protocol.org",
        "metadata": { "schemes": "oauth" }
      }
    ]
  }
}
```

`WS_UNAUTHENTICATED` and `WS_CREDENTIALS_EXPIRED` errors **SHOULD** list, in the `schemes` metadata entry, the comma-separated names of the `securitySchemes` the server accepts. This is the in-band equivalent of the `WWW-Authenticate` challenge, as recommended by [Section 3.3.2](https://a2a-protocol.org/v1.0.0/specification/#332-error-handling) of the A2A specification.

### 9.4. Connection Authentication Context

Once authenticated, the connection carries a single authenticated identity, which applies to every request on it:

- The server **MUST** associate the authenticated identity with the connection and **MUST** check, before processing each request, that the connection's credentials have not expired
- The server **SHOULD** re-check revocation of the credentials while the connection is open (recommended: at least every 60 seconds), for example through token introspection or a revocation list. For JWT access tokens, the server **MUST** honor the `exp` claim
- For credentials that have no expiry of their own, such as API keys, cookies, and `Basic` credentials, the server **SHOULD** set a maximum authentication lifetime for the connection and enforce it as an expiry
- A client certificate is valid for the connection until the certificate expires. It cannot be refreshed in-band; the client opens a new connection with a new certificate
- When the credentials expire or are revoked, the server **MUST NOT** process further requests on the connection. It responds to each request in flight, including active streams, with `-32000` and reason `WS_CREDENTIALS_EXPIRED`, then closes the connection with close code `4001` ([Section 7.4](#74-connection-level-errors))

### 9.5. Credential Refresh

Access tokens usually live for minutes or hours, while a WebSocket connection may stay open much longer. To avoid losing active streams when a token expires, clients **SHOULD** refresh credentials before they expire, using either method:

- **In-band refresh:** send `ws/Authenticate` with new credentials ([Section 9.3](#93-in-band-authentication-with-wsauthenticate)). The connection and its streams are unaffected
- **Reconnection:** open a new connection with new credentials, call `SubscribeToTask` on it for each task in progress, and then close the old connection with close code `1000`. Opening the new connection before closing the old one avoids a gap in events; clients tolerate the duplicate events that may result ([Section 8.5](#85-reconnection))

Clients **SHOULD** track when their credentials expire, using `expiresAt` from `ws/Authenticate` or the token's own expiry, and refresh ahead of time rather than wait for the server.

**`ws/ReauthenticationRequired` notification.** A server **MAY** warn a client that the connection's credentials are about to expire:

```json
{
  "jsonrpc": "2.0",
  "method": "ws/ReauthenticationRequired",
  "params": {
    "expiresAt": "2026-11-10T10:00:00.000Z"
  }
}
```

- `params.expiresAt` is required. It is the time after which the server will reject requests with `WS_CREDENTIALS_EXPIRED` and close the connection with `4001`
- Servers that send this notification **SHOULD** send it ahead of the expiry (recommended: at least 60 seconds before), and **MUST NOT** close the connection for expired credentials before `expiresAt`, unless the credentials are revoked
- A client that receives it **SHOULD** refresh its credentials using either method above. If `ws/Authenticate` returns `MethodNotFoundError` (`-32601`), the client reconnects instead
- Clients that do not support this notification ignore it ([Section 7.6](#76-errors-and-control-messages)) and see the `4001` close when the credentials expire

Revocation takes effect immediately. The server does not need to send `ws/ReauthenticationRequired` first.

### 9.6. Authorization

Once a request is authenticated, the server authorizes it against the connection's identity and its own policies, as described in [Section 7.5](https://a2a-protocol.org/v1.0.0/specification/#75-server-authorization-responsibilities) of the A2A specification:

- Authorization **MUST** be checked for every request, not only at the handshake, because what an identity may do can depend on the task, skill, or data involved
- Tasks belong to the authenticated identity, not to the connection. After reconnecting with credentials for the same identity, a client **MAY** call `SubscribeToTask` for tasks it created on an earlier connection
- If the identity may not access a task, the server **MUST** respond with `TaskNotFoundError` (`-32001`), so that it does not reveal that the task exists ([Section 3.3.2](https://a2a-protocol.org/v1.0.0/specification/#332-error-handling) of the A2A specification)
- For other authorization failures, such as missing OAuth scopes, the server **MUST** respond with `-32000` and reason `WS_PERMISSION_DENIED` ([Section 7.3](#73-binding-specific-errors)). It **SHOULD** name the missing permission or scope in the `ErrorInfo` metadata, as long as that reveals nothing about resources the client cannot access
- Request `id`s, and therefore streams, are scoped to their connection. `ws/CancelStream` can only affect streams on the connection it is sent on
- An intermediary, such as a gateway, that serves several end users **MUST NOT** send requests for different users over one connection, unless the server's authorization policy treats the intermediary itself as the principal

**In-task authorization.** `ws/Authenticate` authenticates the client to the server. It is unrelated to `TASK_STATE_AUTH_REQUIRED` and **MUST NOT** be used to deliver credentials that an agent needs to continue a task. Those are exchanged out of band or through an extension, as described in [Section 7.6](https://a2a-protocol.org/v1.0.0/specification/#76-in-task-authorization) of the A2A specification.

### 9.7. Origin Validation

Browsers attach cookies and client certificates to WebSocket upgrades automatically, including upgrades started by pages on other sites, and the same-origin policy does not apply to WebSocket connections. Without a check, a malicious page could open an authenticated connection with the user's credentials (cross-site WebSocket hijacking). Therefore:

- A server that authenticates with credentials a browser sends automatically (cookies, client certificates, or HTTP authentication cached by the browser) **MUST** validate the `Origin` header of the upgrade request against an allowlist, and **MUST** reject upgrades from other origins with HTTP `403 Forbidden`
- Cookies used for authentication **SHOULD** be set with the `Secure`, `HttpOnly`, and `SameSite` attributes
- Credentials sent explicitly in the `Authorization` header or with `ws/Authenticate` are not attached automatically, so this check is not required for them. Servers **MAY** still validate `Origin`

Clients **SHOULD** verify the server's identity by validating its TLS certificate during the TLS handshake, as recommended by [Section 7.2](https://a2a-protocol.org/v1.0.0/specification/#72-server-identity-verification) of the A2A specification.

### 9.8. Agent Card Security Declarations

Agents declare their security requirements with the standard `securitySchemes` and `securityRequirements` fields of the Agent Card. This binding adds no new fields:

- The declared schemes apply to the WebSocket interface in the same way as to the agent's other interfaces. The server **MUST** accept, over this binding, credentials that satisfy any entry of `securityRequirements`, using the mappings in [Section 9.1](#91-credential-transmission)
- Agents that serve browser clients **SHOULD** declare at least one scheme that such clients can use: a cookie-based API key, or a scheme that works with `ws/Authenticate`
- Support for `ws/Authenticate` and `ws/ReauthenticationRequired` is not declared in the Agent Card; clients discover it as described in [Sections 9.3](#93-in-band-authentication-with-wsauthenticate) and [9.5](#95-credential-refresh)

**Example:**

```json
{
  "securitySchemes": {
    "oauth": {
      "oauth2SecurityScheme": {
        "flows": {
          "clientCredentials": {
            "tokenUrl": "https://auth.example.com/oauth/token",
            "scopes": { "a2a.tasks": "Create and read tasks" }
          }
        }
      }
    }
  },
  "securityRequirements": [
    { "schemes": { "oauth": { "list": ["a2a.tasks"] } } }
  ]
}
```

## 10. Agent Card Declaration

Agents that support the WebSocket binding **MUST** declare it in their Agent Card using the `supportedInterfaces` field.

**Agent Card requirements for WebSocket:**

- `url`: An absolute `wss://` URL of the WebSocket endpoint (`ws://` only for local development). It **MUST NOT** contain credentials ([Section 9.1](#91-credential-transmission)) or `A2A-*` query parameters, which clients add themselves ([Section 4.1](#41-during-connection-upgrade-connection-scoped))
- `protocolBinding`: Set to `"https://a2a-protocol.org/bindings/experimental-websocket/v1"`. Clients that select this interface offer the `a2a.v1` sub-protocol in the handshake ([Section 2.1](#21-connection-establishment))
- `protocolVersion`: The A2A protocol version served at this URL; clients send it as the connection's `A2A-Version`
- `tenant`: Optional; if present, clients set it in the `params` of every request ([Section 5](#5-method-inventory))

Because this binding is experimental, agents **SHOULD** also declare a standard binding (e.g., `JSONRPC`) for clients that do not support it.

**Example Agent Card fragment:**

```json
{
  "name": "Travel Planner Agent",
  "description": "Plans trips and books travel",
  "version": "2.1.0",
  "supportedInterfaces": [
    {
      "url": "wss://agent.example.com/a2a/ws",
      "protocolBinding": "https://a2a-protocol.org/bindings/experimental-websocket/v1",
      "protocolVersion": "1.0"
    }
  ],
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  }
}
```

**Multi-binding example** (agent supporting both WebSocket and JSON-RPC):

```json
{
  "name": "Travel Planner Agent",
  "supportedInterfaces": [
    {
      "url": "wss://agents.example.com/a2a/ws",
      "protocolBinding": "https://a2a-protocol.org/bindings/experimental-websocket/v1",
      "protocolVersion": "1.0",
      "tenant": "travel-planner"
    },
    {
      "url": "https://agents.example.com/a2a/rpc",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "1.0",
      "tenant": "travel-planner"
    }
  ]
}
```

## 11. Data Type Mappings

The WebSocket binding uses JSON serialization following ProtoJSON conventions, identical to the JSON-RPC binding. All A2A data types are represented as defined in `specification/a2a.proto`.

| A2A Data Type                                       | Wire Representation                                                                                                |
| :-------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------- |
| Messages                                            | JSON objects with camelCase field names ([Section 5.5](https://a2a-protocol.org/v1.0.0/specification/#55-json-field-naming-convention)) |
| Timestamps                                          | ISO 8601 strings in UTC with a `Z` suffix, millisecond precision (e.g., `"2026-11-10T10:00:00.000Z"`) ([Section 5.6.1](https://a2a-protocol.org/v1.0.0/specification/#561-timestamps)) |
| Binary data (`bytes`, e.g., `Part.raw`)             | Base64-encoded strings                                                                                             |
| Enumerations                                        | String names as defined in the proto (e.g., `"TASK_STATE_WORKING"`)                                                |
| Integers (`int32`)                                  | JSON numbers                                                                                                       |
| `google.protobuf.Struct` / `Value` (e.g., `metadata`) | Native JSON objects and values                                                                                   |
| Optional fields                                     | Omitted when not set ([Section 5.7](https://a2a-protocol.org/v1.0.0/specification/#57-field-presence-and-optionality)) |
| `oneof` fields (e.g., `Part`, `StreamResponse`)     | Exactly one of the member fields present                                                                           |
| Empty results                                       | Empty object `{}`                                                                                                  |

The binding's own fields follow the same conventions: `serviceParams` is a JSON object of string values, `expiresAt` in `ws/Authenticate` and `ws/ReauthenticationRequired` is a timestamp, and error details use the ProtoJSON `Any` representation with a `@type` member ([Section 7.1](#71-error-response-structure)). Because each message is a text message, binary content is always base64-encoded; large content **SHOULD** be sent as a file reference ([Section 3.6](#36-message-size-limits)).
