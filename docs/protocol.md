# Protocol

An agent harness reports lifecycle events to a server as JSON-RPC 2.0 messages. The method names the event and the parameters follow that event's schema. A request waits for the server's result; a notification does not. The messages are independent of transport: HTTP is the default binding, and a stdio binding runs the server as a child process.

| Message | JSON-RPC id | Server reply | Harness behavior |
| --- | --- | --- | --- |
| Request | Present | Result or error with the matching id | Holds the guarded continuation until it validates a successful result. |
| Notification | Absent | No JSON-RPC reply | Continues after bounded local enqueueing, without waiting for the callback. |

The uppercase requirement words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY use the BCP 14 meanings described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174). Ordinary lowercase words retain their ordinary meaning.

## Integration binding

[The JSON-RPC schema](schema.md) is authoritative for message structure. This document adds delivery and semantic requirements that JSON Schema cannot express. [The standard event catalog](events.md) defines emission boundaries and names each parameter definition.

An integration binds an endpoint to an immutable contract version, a transport binding, supported methods, local waiting policy, request deadlines, result schemas and permitted result effects. It MUST supply a result schema for each enabled request method. The base success envelope permits any JSON result; the integration's result schema is its application contract.

Standard parameter schemas are fixed. A harness MUST NOT project or rename their fields during installation. Optional fields mean the harness did not expose that data. Validation MUST preserve supplied values without coercion, default insertion or transformation.

An adapter MAY support a subset of events. Its binding MUST identify the actual native boundary, available data, ability to wait and permitted effects. Unsupported waiting or effects MUST be rejected before use. Returned data MUST NOT enable hooks, widen access or grant effects outside that binding. A disabled method sends nothing.

## Messages

Each message is one UTF-8 JSON object. A complete message MUST be at most 1 MiB (1,048,576 bytes) in either direction.

| Message | Required members | Reply |
| --- | --- | --- |
| Request | jsonrpc, id, method, params | Matching JSON-RPC result or error |
| Notification | jsonrpc, method, params | None |
| Success | jsonrpc, id, result | Reply to a request |
| Error | jsonrpc, id, error | Reply to a request or to an unclassifiable malformed message |

jsonrpc MUST equal "2.0". method is a nonempty string and names the hook directly. params is required and object-shaped. An id is a nonempty string of at most 128 UTF-8 bytes. A client MUST create a fresh request id for each invocation. An id correlates a reply; it establishes neither ordering nor idempotence.

Batch messages, numeric or null request ids, duplicate object keys and unknown envelope members MUST be rejected. Every JSON number MUST be finite. Success replies MUST echo the request id. A reply MUST contain exactly one of result or error. Error contains an integer code, a nonempty message and optional data. Parse errors use id null; null error ids are otherwise permitted only for invalid requests whose id cannot be identified.

The JSON-RPC id is the only reply-expectation signal. Bindings are configured before event delivery.

| Failure | JSON-RPC error |
| --- | --- |
| Malformed JSON | -32700, id null |
| Invalid envelope | -32600 when a reply is applicable |
| Unknown request method | -32601 |
| Invalid request params | -32602 |
| Callback failure, invalid or oversized result | -32603 |
| Invalid notification method or params | None |

## Validation

A receiver MUST:

1. Enforce authentication, encoding, parser and resource bounds before callback execution.
2. Validate the generic RequestEnvelope or NotificationEnvelope.
3. Resolve method support from the installed contract.
4. Validate params against that method's schema before invoking its callback.
5. For a request, validate the callback result and the reply size bound before sending a result.

A client MUST match the reply id and validate both the reply envelope and the installed result schema before applying a result. The schema root validates standard invocations and reply envelopes. Generic envelope definitions permit classification before method lookup and support approved extensions; an unknown method is not automatically an invalid envelope.

Runtime checks MUST enforce UTF-8 byte bounds, finite numbers, URI policy, canonical base64 decoding and cross-message relationships. A supplied permission option id MUST identify an offered option and agree with its decision when the options are available. An accepted form answer MUST satisfy its form schema. Catalog names, permission option ids, and supplied batch-call or plan-step ids MUST be unique within their lists. Configuration-change keys MUST also be unique.

The origin in a network permission operation MUST use HTTP or HTTPS, contain a hostname and a valid optional port, and contain no credentials, query or fragment. Its path MUST be empty or "/". Other resource URIs remain references under the integration's access policy; observing them does not trigger a fetch.

Schemas inside catalogs and forms MUST use Draft-07. A different native dialect requires semantics-preserving conversion or rejection. A missing native dialect declaration does not establish compatibility. Embedded schemas remain data and cannot replace the installed contract. References MUST resolve from approved local resources; event processing MUST NOT fetch schemas remotely. Schema evaluation MUST have bounded resource use.

## Requests

A valid request receives a result or error after the callback completes. When a request guards a pending continuation, the harness MUST hold that continuation until it receives a valid successful reply. A transport failure, timeout, mismatched id, invalid result or oversized reply fails the request and MUST prevent that pending continuation.

The request deadline starts before local outbound queueing. A valid empty result permits continuation only if the result schema accepts it. Failure does not roll back completed work or prove server-side cancellation. A post-action request can delay the next action; it cannot undo the observed action. A native asynchronous hook cannot become a blocking boundary through configuration.

The protocol defines no universal context, denial, priority or instruction-role result fields. Result meaning and effects belong to the integration. A permission event describes a permission workflow and grants no authority by itself.

## Notifications

A sender MAY queue notifications locally and continue without waiting for their callbacks. It MUST bound that queue and expose local admission or transport failures through diagnostics.

A server admits a notification after authentication, parameter validation and bounded in-memory admission, without waiting for callback completion. The callback may already have finished when admission is confirmed. Accepted notifications can be lost on restart. Callback failures after admission remain server diagnostics.

A recognized notification MUST NOT receive a JSON-RPC reply, including on method or parameter failure. Malformed messages that cannot be classified as notifications follow invalid-request handling.

A process that queues notifications MUST either drain its pending delivery attempts before exit or hand them to a resident sender. Such a drain establishes admission attempts, not callback completion. No fixed drain allowance is part of the wire contract.

## Identity and ordering

A session is a continuing conversation or work context. A run is one invocation of agent work, potentially containing several model requests and tool calls. A model request is one model invocation.

Session-scoped events require sessionId. runId and agentId are optional stable correlation identifiers. Global catalog, environment and runtime events can omit sessionId; supplying runId or agentId requires it. Child lifecycle observations use the parent's scope and identify the child separately.

A call id identifies one tool invocation when available. It MUST remain stable across observations of that invocation. An adapter MUST NOT infer identity from names, arguments, clocks, per-process counters or arrival order. It MAY assign an id when it owns a reliable native execution handle. Without stable ids, receivers cannot pair lifecycle observations.

Calls can execute and finish out of order. A notification can finish after a later request. The protocol provides no global barrier, replay or exactly-once effects. A sender MUST NOT automatically retry a callback. Applications needing a causal precondition MUST supply it as input or await the earlier request. After a lost reply, execution remains uncertain. Idempotent application effects require an application-defined key and storage rule.

Catalog and plan snapshots describe state at emission time. Receipt order alone cannot establish which snapshot is current.

## Event and content semantics

A proposed tool call contains name and actual arguments, which can be any JSON value. The tool's inputSchema belongs to its definition. Observational calls MAY omit arguments when unavailable. tool.requested precedes execution and can precede permission checks; tool.started means execution actually began. A denied terminal outcome means execution never began. Unknown means termination is known but success or failure was not exposed, and MUST NOT be interpreted as success.

run.finishing is a checkpoint before accepting proposed completion. run.finished records an ended invocation. RunCompletion also represents interruption short of completion; agent.finished uses the same contract for its child invocation. run.paused and run.resumed apply only to the same live invocation. A harness that ends an invocation and later starts another MUST use run.finished and run.started, with resumesRunId only when that relation is known. handoff.finished reports transfer outcome, not destination work completion.

Payload contains ordered content and optional structuredContent holding any JSON value. Input requires at least one content block or the presence of structuredContent. The content variants are text, image, audio, resource_link and resource. Binary content uses canonical base64 and a MIME type. URIs are references and grant no fetch permission.

An omitted output means unavailable data. A present output with content empty is known empty content. Failure details can be omitted when unavailable. Execution status MUST come from the native outcome rather than text matching. ModelOutput additionally carries typed calls, allowing call-only responses with empty content.

input.submitted carries new input, including application and agent input. model.requested may expose assembled input; its complete or partial view MUST reflect what the binding exposes. Hidden provider internals and private reasoning are outside the model-input schema.

context.source.loaded means identified material entered agent context. Skill discovery, invocation or file reading alone cannot establish ingestion. catalog.changed contains a complete typed tools or skills snapshot, including empty snapshots; availability grants no permission and proves no loading. Configuration events MUST exclude credential values; absent values mean unexposed data, not deletion.

## Client identity and consent

A client describes itself with a `ClientInfo` object: its name and version, and optionally the harness it runs in. The description is self-reported. A server MUST NOT use it for authorization, which comes only from the binding's credentials.

When a user registers a server, the client sends `client.registered` with its `ClientInfo` and the methods the user consented to share; the default is every method the client supports. It sends `client.updated` with the new list whenever the selection changes and `client.unregistered` when the server is removed. A client MUST NOT send methods outside the consented list, except these three. Servers SHOULD ignore methods outside the list and MUST NOT conclude that an event did not happen because it was not shared.

`client.registered` and `client.updated` SHOULD be sent as requests. A server that needs an unconsented method for a guarded effect says so in its integration-defined result instead of failing later.

## Resource bounds

Implementations MUST bound queued work, active callbacks, incomplete messages, read times, parsing and validation. Capacity MUST remain reserved through callback completion, including after caller timeout. Numerical concurrency and local shutdown defaults are implementation policy.

Session identifiers are data; stateful servers MUST separately authorize access to the selected state. Message validation does not constrain a server process's filesystem or network privileges.

## HTTP binding

HTTP is the default binding. The client sends each message as one HTTP POST with `Content-Type: application/json`. A request returns HTTP 200 with its JSON-RPC result or error. A notification returns an empty HTTP 202 once admitted. HTTP connection reuse is an implementation detail; closing local resources requires no protocol message.

| Failure | HTTP response | JSON-RPC error |
| --- | --- | --- |
| Malformed JSON | 400 | -32700, id null |
| Invalid envelope | 400 | -32600 when a reply is applicable |
| Unknown request method | 200 | -32601 |
| Invalid request params | 200 | -32602 |
| Callback failure, invalid or oversized result | 200 | -32603 |
| Invalid notification method or params | Empty 400 | None |
| Authentication or authorization failure | 401 or 403 | None required |
| Unsupported media type | 415 | None required |
| Oversized request | 413 | None required |
| Admission capacity exhausted | 429 | None required |

HTTP admission errors are transport feedback, not JSON-RPC replies. Implementations MUST also bound incomplete connections and header read times.

### Endpoints and security

An endpoint can be loopback (127.0.0.1, ::1 or localhost) or remote. A non-loopback endpoint SHOULD use HTTPS and MUST require HTTP Authorization. Servers MUST validate the target and Host against the names they serve and reject cross-origin browser calls; a loopback server MUST also reject non-loopback peers. Clients MUST send HTTP Authorization when credentials are configured.

A client MAY send its `ClientInfo` on every call as an `ACLP-Client` header of product tokens, the client and then optionally the harness, for example `ACLP-Client: aclp-hooks/0.1.0 claude-code/2.1.4`.

### Discovery and authentication

A client given only a server URL discovers the endpoint with `GET /.well-known/aclp` at that URL's origin. The response is a JSON `DiscoveryDocument` from [the JSON-RPC schema](schema.md). Discovery requires no credentials and MUST NOT return secrets.

| Member | Meaning |
| --- | --- |
| `endpoint` | Absolute `http` or `https` URL that accepts ACLP messages. The endpoint rules above apply. |
| `auth.methods` | Supported ways to obtain credentials: `token`, `oauth` or none. |
| `auth.oauth` | OAuth 2.1 metadata, present exactly when `oauth` is listed: `issuer`, `authorization_endpoint`, `token_endpoint`, optional `device_authorization_endpoint`, `client_id` and `scopes_supported`. |
| `events` | Methods the server accepts. |
| `requests` | Methods the server answers as guarded requests; a subset of `events`. |
| `name` | Optional display name. |

With `token`, the user supplies a token that the client sends as `Authorization: Bearer <token>`. With `oauth`, the client uses the authorization code flow with PKCE and a loopback redirect, or the device authorization flow when `device_authorization_endpoint` is present, and sends the access token as a Bearer credential. Clients refresh access tokens with the refresh grant when one was issued. A server refuses a missing, expired or invalid credential with HTTP 401 and a `WWW-Authenticate` challenge so the client can obtain new credentials. Clients MUST NOT store credentials in harness configuration.

## stdio binding

The client starts the server as a child process and exchanges messages over its standard streams. The installation names the command, its arguments and its environment, including any credentials; there is no discovery and no Authorization header.

Messages are newline-delimited JSON. Each message is one line: compact UTF-8 JSON followed by a single line feed (U+000A), with no line feed inside the JSON text. A line MUST be at most 1 MiB excluding its line feed. The client writes messages to the server's standard input; the server writes replies to its standard output and nothing else. Standard error carries diagnostics only and is never parsed.

A server MUST reply to each request on one line with the matching id, and MAY reply out of order; the client matches replies by id. A notification is admitted once the server has read and validated it and receives no reply. A server MUST NOT write messages that are not replies. An oversized or malformed line receives the error reply the Messages table requires, when one is applicable, and is otherwise reported on standard error.

When admission capacity is exhausted, a server answers a request with JSON-RPC error -32001 and drops a notification with a diagnostic on standard error. Deadlines, fail-closed requests and notification draining apply as in the transport-neutral rules. A client MAY end a server that stops responding. Closing the server's standard input ends the session; the server SHOULD finish admitted callbacks, then exit.

The client sends `ClientInfo` only through `client.registered` and `client.updated`.

## Versioning and extensions

An endpoint binds an immutable contract version. Incompatible changes require a new endpoint and renewed installation approval. Strict object schemas reject unknown fields. Approved extension methods MUST declare fixed parameter and result schemas and use names distinct from standard methods and JSON-RPC's reserved rpc. namespace.

Long polling, server-originated messages, streaming channels, durable delivery and workload isolation are outside this protocol.

The underlying standards are [JSON-RPC 2.0](https://www.jsonrpc.org/specification) and [JSON Schema Draft-07](https://json-schema.org/draft-07).
