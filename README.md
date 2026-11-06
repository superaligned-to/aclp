# Agent Context Lifecycle Protocol

ACLP lets an agent harness report lifecycle events (input, model calls, tool calls, runs, sessions) to a server and wait for the server's answer at the points where the harness can pause. Messages are JSON-RPC 2.0 with a fixed schema per event, sent over HTTP by default or over stdio.

## Example

The harness asks a server for context before it sends user input to the model:

```http
POST /aclp HTTP/1.1
Content-Type: application/json
Authorization: Bearer <token>

{"jsonrpc": "2.0", "id": "input-1", "method": "input.submitted",
 "params": {"sessionId": "session-1", "content": [{"type": "text", "text": "Create a TypeScript class."}]}}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"jsonrpc": "2.0", "id": "input-1", "result": {"context": ["Prefer plain functions over classes."]}}
```

The harness validates the result against the schema it installed for this callback, then applies it. Without an `id`, the same message is a notification: the server answers `202 Accepted` and the harness does not wait.

## Specification

- [Protocol](docs/protocol.md): messages, validation, requests, notifications, identity and consent, and the HTTP and stdio bindings.
- [Events](docs/events.md): the 43 standard events and their parameters.
- [Schema](docs/schema.md): the Draft-07 JSON Schema for every message.

## SDKs

- TypeScript: [superaligned-to/aclp-ts](https://github.com/superaligned-to/aclp-ts)
- Rust: [superaligned-to/aclp-rs](https://github.com/superaligned-to/aclp-rs)

Licensed under [MIT](LICENSE).
