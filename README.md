# A2A WebSocket Custom Protocol Binding

> **Status: Experimental** — This is a community-contributed custom protocol binding for A2A. It is not part of the core A2A specification.

This repository contains the specification for the **WebSocket custom protocol binding** for the [Agent2Agent (A2A) protocol](https://github.com/a2aproject/A2A).

## Overview

The binding carries A2A over a single, long-lived WebSocket connection. It has two layers:

1. **Transport — WebSocket ([RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)).** A full-duplex connection opened with an HTTP/1.1 `Upgrade` handshake, using the `a2a.v1` sub-protocol.
2. **Application — JSON-RPC 2.0.** Each WebSocket text message carries one JSON-RPC object, using the same methods, request and response objects, ProtoJSON serialization, and error codes as the A2A [JSON-RPC binding](https://a2a-protocol.org/v1.0.0/specification/#9-json-rpc-protocol-binding).

Because one connection carries many concurrent requests, the binding adds only what that requires:

- **Multiplexing by JSON-RPC `id`:** responses and stream events are matched to their request by `id`
- **`serviceParams`:** a request member that carries per-request A2A service parameters, since WebSocket messages have no headers
- **Control messages** with the reserved `ws/` prefix:

| Method                        | Direction       | Required | Purpose                                        |
| :---------------------------- | :-------------- | :------- | :--------------------------------------------- |
| `ws/StreamEnd`                | Server → Client | Yes      | Ends a stream                                  |
| `ws/CancelStream`             | Client → Server | Yes      | Stops a stream without cancelling its task     |
| `ws/Authenticate`             | Client → Server | No       | Authenticates or refreshes credentials in-band |
| `ws/ReauthenticationRequired` | Server → Client | No       | Warns that the connection's credentials expire soon |

The binding does not add new A2A operations or a bidirectional streaming model. Streaming remains server-to-client; each client turn is its own `SendMessage` or `SendStreamingMessage` request.

## Why WebSocket?

HTTP/2 already multiplexes requests and supports server streaming, so neither is the reason for this binding. It fills these gaps:

- **Client-to-server signaling on a live stream.** With SSE or plain HTTP, the client cannot send anything on a stream that is in progress. Over WebSocket, the client can stop one stream, refresh its credentials, or start new requests while events are still arriving.
- **Browser-native.** The gRPC binding needs a `grpc-web` proxy and Protocol Buffers tooling in browsers. WebSocket is available in every modern browser with JSON only.
- **One connection the application controls.** The application owns a single connection and multiplexes its own requests over it, so authentication and server state are tied to that connection. The HTTP/1.1 upgrade also passes through proxies that downgrade HTTP/2.

| Aspect                         | JSON-RPC (HTTP)          | gRPC                 | WebSocket                                    |
| :----------------------------- | :----------------------- | :------------------- | :------------------------------------------- |
| Application layer              | JSON-RPC 2.0             | Protocol Buffers RPC | JSON-RPC 2.0 (same objects as JSON-RPC)      |
| Connection                     | One HTTP request per RPC | HTTP/2, multiplexed by the runtime | One persistent connection, multiplexed by `id` |
| Streaming                      | Server-Sent Events       | HTTP/2 server stream | Repeated `result` messages, then `ws/StreamEnd` |
| Client signaling during a stream | No                     | Cancel only          | Yes (`ws/CancelStream`, `ws/Authenticate`, new requests) |
| Browser support                | Yes                      | Requires `grpc-web`  | Yes (native `WebSocket` API)                 |
| Authentication                 | Per request              | Per request          | Per connection, with in-band refresh         |

Agents that offer this binding should also offer a standard binding, so that clients without WebSocket support can still reach them.

## Protocol Binding Specification

The `protocolBinding` identifier for this binding is versioned. The current version is `v1`:

```
https://a2a-protocol.org/bindings/experimental-websocket/v1
```

A breaking change requires a new identifier (`.../v2`) and a matching sub-protocol (`a2a.v2`).

### Versioned Specifications

| Version | Document | Description |
| :------ | :------- | :---------- |
| `v1` | [`spec/v1/websocket.md`](spec/v1/websocket.md) | Core binding specification — protocol requirements, connection lifecycle, message framing, service parameters, method inventory, error handling, streaming, authentication, Agent Card declaration, and data type mappings |

### Agent Card Example

```json
{
  "supportedInterfaces": [
    {
      "url": "wss://agent.example.com/a2a/ws",
      "protocolBinding": "https://a2a-protocol.org/bindings/experimental-websocket/v1",
      "protocolVersion": "1.0"
    },
    {
      "url": "https://agent.example.com/a2a/rpc",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "1.0"
    }
  ]
}
```

## Reference Implementations

No reference implementations are available yet. Planned:

| Language | Repository | Status |
| :------- | :--------- | :----- |
| Rust     | [github.com/a2aproject/a2a-rs](https://github.com/a2aproject/a2a-rs) | Planned |
| Python   | — | Planned |
| TypeScript | — | Planned (browser-compatible client) |

## A2A Protocol

This binding is designed to work with the [A2A protocol specification](https://github.com/a2aproject/A2A). Refer to the main A2A repository for the core specification, proto definitions, and other protocol bindings.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines. This project follows the same process as the main A2A project.

## License

This project is licensed under the [Apache License 2.0](LICENSE), the same license as the main A2A project.

## Code of Conduct

This project follows the A2A [Code of Conduct](CODE_OF_CONDUCT.md).
