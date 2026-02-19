# MangleCP Specification 02: Wire Format and Transport Bindings

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), RFC 8259 (JSON), RFC 8615 (Well-Known URIs)

---

## 1. Purpose

This document specifies the wire format for MangleCP messages and defines normative transport bindings for WebSocket, HTTP request/response, and stdio transports.

---

## 2. Message Envelope

Every MangleCP message is a JSON object conforming to the following envelope structure:

```json
{
  "type": "<string>",
  "id": "<string | null>",
  "manglecp": "<string>",
  "payload": {}
}
```

### 2.1 Envelope Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | REQUIRED | One of the defined message types (see Section 2.2) |
| `id` | string \| null | CONDITIONAL | MUST be present for request messages. MUST be echoed in the corresponding response. MAY be null for server-initiated messages (manifest, notifications). |
| `manglecp` | string | REQUIRED | Protocol version string. For this draft: `"2026-02-draft"`. |
| `payload` | object | REQUIRED | Type-specific payload. MAY be an empty object `{}` but MUST be present. |

### 2.2 Message Types

| Type String | Direction | Correlation | Defined In |
|-------------|-----------|-------------|------------|
| `"manifest"` | Server -> Client | None (server-initiated) | Spec 04 |
| `"intent_request"` | Client -> Server | `id` required | Spec 05 |
| `"intent_response"` | Server -> Client | `id` echoed | Spec 05 |
| `"invoke_request"` | Client -> Server | `id` required | Spec 07 |
| `"invoke_response"` | Server -> Client | `id` echoed | Spec 07 |
| `"progress"` | Server -> Client | `id` echoed | Spec 07 |
| `"error"` | Server -> Client | `id` echoed if correlatable | Spec 09 |

### 2.3 Correlation Rules

- A client MUST generate a unique `id` for each request within a session.
- A server MUST echo the `id` from the request in the corresponding response.
- If the server cannot correlate an error to a specific request, it MUST set `id` to `null`.
- The `id` MUST be a non-empty string. Implementations SHOULD use UUIDs or monotonically increasing counters.

### 2.4 Version Negotiation

- The client MUST send the `manglecp` version string in every request.
- The server MUST send the `manglecp` version string in every response and in the manifest.
- If the server does not support the client's requested version, it MUST respond with an `error` message with code `"unsupported_version"` and include a `supported_versions` array in the error details.

---

## 3. Serialization Requirements

### 3.1 JSON Conformance

All MangleCP messages MUST be valid JSON per [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259).

- Implementations MUST support the full Unicode range in string values.
- Numbers MUST conform to JSON number syntax. Implementations SHOULD preserve integer precision for values up to 2^53 (IEEE 754 safe integer range).
- For values requiring precision beyond 2^53, implementations SHOULD use string encoding with a type annotation.

### 3.2 Size Limits

- Servers MUST accept messages up to at least **16 MiB** in size.
- Servers MAY reject messages exceeding their configured maximum with error code `"message_too_large"`.
- Servers MUST advertise their maximum message size in the manifest's `limits.max_message_bytes` field.

### 3.3 Character Encoding

All messages MUST be encoded in UTF-8 without a byte order mark (BOM).

---

## 4. Transport Bindings

MangleCP is transport-agnostic at the message level. This section defines normative bindings for three transport classes.

### 4.1 WebSocket Transport

WebSocket is the RECOMMENDED transport for session-oriented communication.

#### Connection Establishment

1. Client opens a WebSocket connection to the server's MangleCP endpoint.
2. Server MUST send a `manifest` message as the **first application message** after the WebSocket handshake completes.
3. Client MUST NOT send any request messages before receiving the manifest.

#### Message Framing

- Each MangleCP message MUST be sent as a single WebSocket **text frame** containing the JSON-serialized message.
- Implementations MUST NOT split a single MangleCP message across multiple WebSocket frames.
- Implementations MUST NOT batch multiple MangleCP messages into a single frame.

#### Connection Lifecycle

- Either party MAY close the WebSocket connection at any time using the standard WebSocket close handshake.
- On unexpected connection loss, clients SHOULD reconnect and await a new manifest. Clients MUST NOT assume state from a previous session persists.

#### Ping/Pong

- Implementations SHOULD use WebSocket ping/pong frames for keepalive.
- Servers SHOULD send a ping at least every 30 seconds during idle periods.

### 4.2 HTTP Request/Response Transport

HTTP is the RECOMMENDED transport for stateless, request-scoped communication.

#### Endpoint Structure

Servers MUST expose the following endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/.well-known/manglecp/manifest.json` | GET | Manifest retrieval (RFC 8615) |
| `{base}/evaluate` | POST | Intent evaluation |
| `{base}/invoke` | POST | Macro-tool invocation |

The `{base}` path is specified in the manifest's `endpoints` object. If the manifest specifies relative URIs, they are resolved relative to the manifest's own URL.

#### Request Format

- Content-Type MUST be `application/json`.
- Accept MUST include `application/json`.
- The request body is a MangleCP message envelope (Section 2).

#### Response Format

- Content-Type MUST be `application/json`.
- The response body is a MangleCP message envelope.
- HTTP status codes MUST be used as follows:

| HTTP Status | MangleCP Meaning |
|-------------|-----------------|
| 200 OK | Successful response (intent_response or invoke_response) |
| 400 Bad Request | Malformed message, invalid facts, schema validation failure |
| 401 Unauthorized | Authentication required but not provided |
| 403 Forbidden | Authentication provided but insufficient |
| 408 Request Timeout | Evaluation exceeded `max_compute_ms` |
| 413 Payload Too Large | Message exceeds size limit OR derivation limit exceeded |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Server-side evaluation failure |
| 503 Service Unavailable | Server not ready (boot phase) |

- The response body MUST still contain a MangleCP error message with structured details, even for error HTTP status codes.

#### CORS

For browser-accessible deployments, servers SHOULD support CORS with appropriate `Access-Control-Allow-Origin` headers. Servers MUST NOT use wildcard CORS (`*`) when authentication is required.

### 4.3 Stdio Transport

Stdio transport is for local process communication (e.g., IDE plugins, CLI tools).

#### Protocol

- The server reads MangleCP messages from **stdin** and writes responses to **stdout**.
- Each message is a single line of JSON terminated by a newline character (`\n`).
- Diagnostic/log output MUST be written to **stderr**, never stdout.
- The server MUST write the manifest message to stdout immediately upon startup, before reading any input.

#### Lifecycle

- The client starts the server as a subprocess.
- When the client closes stdin (EOF), the server SHOULD shut down gracefully.
- The server MUST NOT write to stdout after detecting stdin EOF except to flush pending responses.

#### Authentication

- Stdio transport is considered **local-trust**. Authentication is OPTIONAL for stdio connections.
- Servers MAY accept an authentication token via a command-line flag or environment variable for stdio transports where privilege separation is desired.

---

## 5. Discovery

### 5.1 HTTP Discovery

Clients discovering MangleCP servers over HTTP MUST:

1. Fetch `GET /.well-known/manglecp/manifest.json` on the target host.
2. Parse the manifest to extract endpoint URIs and authentication requirements.
3. Use the endpoint URIs for subsequent intent evaluation and invocation requests.

If the well-known path returns 404, the host does not support MangleCP.

### 5.2 DNS-SD Discovery (Informative)

For local network discovery, MangleCP servers MAY advertise via DNS-SD with service type `_manglecp._tcp`. The TXT record SHOULD include:

- `path=/.well-known/manglecp/manifest.json`
- `v=2026-02-draft`

This binding is informative and not required for conformance.

### 5.3 Stdio Discovery

For stdio transport, no discovery is needed. The client starts the server process and reads the manifest from stdout.

---

## 6. Concurrency

### 6.1 Request Pipelining

- Over session transports (WebSocket, stdio), clients MAY send multiple requests without waiting for responses (pipelining).
- Servers MUST process pipelined requests and MUST correlate each response to the correct request via `id`.
- Servers MAY process pipelined requests concurrently or serially; the protocol does not mandate ordering.
- If ordering matters, clients MUST wait for each response before sending the next request.

### 6.2 Cancellation

- Over session transports, clients MAY send a `cancel` message to abort an in-progress request:

```json
{
  "type": "cancel",
  "id": null,
  "manglecp": "2026-02-draft",
  "payload": {
    "request_id": "<id of the request to cancel>"
  }
}
```

- Servers SHOULD make a best-effort attempt to cancel the referenced request. If cancellation succeeds, the server MUST respond with an error message with code `"cancelled"` and the cancelled request's `id`.
- Cancellation is advisory. The server MAY complete the request normally if cancellation is not feasible.
- For HTTP transport, cancellation is handled by the client closing the HTTP connection.
