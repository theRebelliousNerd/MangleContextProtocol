# MangleCP Specification 07: Invocation Protocol and State Management

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), Spec 03 (Data Model), Spec 06 (Macro-Tools)

---

## 1. Purpose

This document specifies the macro-tool invocation layer -- the "Inner Doll" of the Matryoshka architecture. It defines the invoke request/response payloads, transactional state delta semantics, observability events, and the suggested intents mechanism for conversational chaining.

---

## 2. Invoke Request

### 2.1 Invoke Request Payload

```json
{
  "macro_id": "<string>",
  "args": {},
  "eval_time": "<Time | null>",
  "confirmation_token": "<string | null>"
}
```

### 2.2 Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `macro_id` | string | REQUIRED | The `macro_id` from a previously received MacroTool object. |
| `args` | object | REQUIRED | Invocation arguments. MUST validate against the macro-tool's `input_schema`. |
| `eval_time` | Time | OPTIONAL | Evaluation time for temporal rules during execution. If omitted, server uses wall clock. |
| `confirmation_token` | string | OPTIONAL | For tools with `requires_user_confirmation: true`, a token proving user consent (see Spec 08). |

### 2.3 Validation Rules

Before executing a macro-tool, the server MUST validate:

1. **Identity**: The `macro_id` corresponds to a macro-tool from a recent intent evaluation. Servers MAY cache macro-tool definitions for a configurable duration.
2. **Validity window**: If the macro-tool has a `validity.expires_at`, the current time MUST be before the expiration. Error: `"macro_expired"`.
3. **Schema validation**: The `args` object MUST validate against the macro-tool's `input_schema`. Error: `"schema_validation_failed"`.
4. **Confirmation**: If the macro-tool requires user confirmation and no `confirmation_token` is provided, the server MUST reject with error code `"confirmation_required"`.

### 2.4 Macro-Tool Caching

Servers MUST retain macro-tool definitions for at least the duration of the tool's validity window. If a macro-tool has no explicit validity window, servers SHOULD retain definitions for at least 5 minutes after the intent response.

Servers MAY evict cached macro-tools when memory pressure requires it. If a client invokes an evicted `macro_id`, the server MUST respond with error code `"macro_not_found"` and the client SHOULD re-evaluate the intent.

---

## 3. Invoke Response

### 3.1 Invoke Response Payload

```json
{
  "result": {},
  "state_delta": {
    "assert": [ /* Fact objects to add */ ],
    "retract": [ /* Fact patterns to remove */ ]
  },
  "observability": {
    "summary": "<string>",
    "events": [ /* structured events */ ],
    "duration_ms": "<integer>"
  },
  "next": {
    "suggested_intents": [
      {
        "name": "<string>",
        "params": {},
        "description": "<string>"
      }
    ],
    "continuation_facts": [ /* Facts the client should include in the next intent */ ]
  }
}
```

### 3.2 Result Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `result` | object | REQUIRED | The output of macro-tool execution. If the macro-tool defined an `output_schema`, the result MUST validate against it. |

The result object's structure is domain-specific. Examples:

```json
// Browser observation result
{
  "page_state": "loaded",
  "console_errors": 3,
  "interactive_elements": 42,
  "dom_summary": "Dashboard page with user profile card and activity feed"
}

// File modification result
{
  "files_created": ["src/routes/users.go", "src/routes/users_test.go"],
  "files_modified": ["src/routes/router.go"],
  "tests_passed": 12,
  "tests_failed": 0
}

// Diagnostic result
{
  "causal_chain": [
    {"step": 1, "event": "GET /api/users returned 404", "timestamp": "2026-02-19T14:29:59Z"},
    {"step": 2, "event": "UserList component received null data", "timestamp": "2026-02-19T14:30:00Z"},
    {"step": 3, "event": "TypeError: Cannot read properties of null", "timestamp": "2026-02-19T14:30:00Z"}
  ],
  "root_cause": "API endpoint /api/users not registered in router",
  "suggested_fix": "Add route handler in src/routes/router.go"
}
```

---

## 4. Transactional State Delta

### 4.1 Purpose

The state delta communicates **what changed** as a result of macro-tool execution. Clients can feed these facts back into subsequent intent evaluations, enabling conversational chaining without the server maintaining session state.

This formalizes codeNERD's `KernelTransaction` pattern: atomic retract+assert operations that update the fact state in a single logical transaction.

### 4.2 State Delta Structure

```json
{
  "assert": [
    {
      "pred": "phase_status",
      "args": ["spec-001", 2, "done"],
      "category": "server",
      "source": {"source_type": "server", "asserted_at": "2026-02-19T14:30:05Z"}
    }
  ],
  "retract": [
    {
      "pred": "phase_status",
      "args": ["spec-001", 2, "in_progress"]
    }
  ]
}
```

### 4.3 Assert Semantics

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `assert` | array of Fact objects | OPTIONAL | Facts to add to the client's working state. |

Assert facts:

- MUST include `category` (Spec 03 Section 6)
- SHOULD include `source` provenance (Spec 03 Section 5)
- MAY include temporal annotations `t`

### 4.4 Retract Semantics

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `retract` | array of Fact objects | OPTIONAL | Fact patterns to remove from the client's working state. |

Retract facts use **pattern matching**:

- A retract fact with all arguments specified removes that exact fact
- A retract fact with `null` arguments matches any value in that position (wildcard retraction)

```json
// Retract a specific fact
{"pred": "phase_status", "args": ["spec-001", 2, "in_progress"]}

// Retract all phase_status facts for spec-001 phase 2 (any status)
{"pred": "phase_status", "args": ["spec-001", 2, null]}

// Retract all phase_status facts for spec-001 (any phase, any status)
{"pred": "phase_status", "args": ["spec-001", null, null]}
```

### 4.5 Transactional Guarantees

When processing a state delta, clients SHOULD apply the operations in this order:

1. Apply all retractions first
2. Apply all assertions second

This ordering ensures that fact replacement (retract old value, assert new value) works correctly. This mirrors codeNERD's `KernelTransaction.Commit()` which executes retractions before assertions under a single lock.

### 4.6 State Delta and Statelessness

MangleCP is a stateless protocol. The state delta is an **advisory output**, not a server-side state mutation:

- The server MUST NOT assume the client applied the state delta
- The client MAY ignore, modify, or partially apply the state delta
- The client is responsible for maintaining its own working state
- Each intent request is evaluated independently from a fresh store

The state delta exists to help clients build an accurate picture of state changes for feeding into subsequent requests. It is NOT a shared mutable state mechanism.

---

## 5. Observability

### 5.1 Observability Object

The observability section provides human-readable execution summaries and machine-readable event traces.

```json
{
  "summary": "Executed 3 atomic actions: queried console errors, correlated with network requests, traced DOM impact. Found causal chain from API 404 to TypeError.",
  "events": [
    {
      "action": "query_console",
      "status": "success",
      "duration_ms": 12,
      "detail": "Found 3 console errors"
    },
    {
      "action": "correlate_network",
      "status": "success",
      "duration_ms": 8,
      "detail": "Matched request r42 to error"
    },
    {
      "action": "trace_dom_impact",
      "status": "success",
      "duration_ms": 15,
      "detail": "UserList component affected"
    }
  ],
  "duration_ms": 47
}
```

### 5.2 Event Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | REQUIRED | Name of the atomic action executed |
| `status` | string | REQUIRED | `"success"`, `"failure"`, `"skipped"` |
| `duration_ms` | integer | OPTIONAL | Wall-clock time for this action |
| `detail` | string | OPTIONAL | Human-readable detail |

### 5.3 Observability and Context Window Flooding

Servers MUST design observability output and state deltas to prevent uncontrolled flooding of the client's LLM context window:

- **State Delta Cap:** Servers SHOULD limit the number of specific assertions returned in `state_delta`. If an action touches 1,000 files, the server MUST NOT return 1,000 `file_modified` facts; instead, it SHOULD return a single rolled-up fact (e.g., `bulk_modification_completed`).
- **Event Trace Cap:** The `events` array SHOULD contain structurally significant milestones, not low-level trace spans. The maximum number of events MUST be a configurable parameter set by the server developer (e.g., defaulting to 20), rather than a hardcoded protocol limit.
- **Log Summarization:** The `summary` field MUST be a concise distillation (1-3 sentences) of the execution, not a raw dump of tool outputs.

Servers SHOULD provide enough observability for clients to understand what happened during execution, but MUST NOT expose security-sensitive internal details (credentials, internal URLs, raw database queries). Observability events SHOULD correspond to the atomic actions the macro-tool bundled, giving the client visibility into the execution chain without requiring the client to manage each step individually.

---

## 6. Suggested Intents

### 6.1 Purpose

Suggested intents enable conversational chaining. After a macro-tool executes, the server can recommend what the client should do next based on the execution results and derived facts.

This formalizes spec-generator's `component_reuse_candidate` pattern -- proactive recommendations derived from structural analysis.

### 6.2 Suggested Intent Object

```json
{
  "name": "<string>",
  "params": {},
  "description": "<string>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | REQUIRED | Intent name the client should evaluate next |
| `params` | object | OPTIONAL | Pre-filled intent parameters |
| `description` | string | RECOMMENDED | Informational description of why this intent is suggested and what it would accomplish |

### 6.3 Continuation Facts

```json
{
  "continuation_facts": [
    {"pred": "completed_phase", "args": ["spec-001", 2]},
    {"pred": "files_modified", "args": ["src/routes/router.go"]}
  ]
}
```

Continuation facts are facts the client SHOULD include in the next intent request if it follows the suggested intent. They carry forward relevant state from the current execution.

### 6.4 Chaining Pattern

The full conversational chaining pattern:

```
1. Client sends intent_request A
2. Server responds with macro_tools
3. Client invokes macro_tool M
4. Server responds with:
   - result
   - state_delta (retract old facts, assert new facts)
   - next.suggested_intents: [intent B]
   - next.continuation_facts: [facts for intent B]
5. Client applies state_delta to its working state
6. Client sends intent_request B with:
   - continuation_facts from step 4
   - any additional client-asserted facts
7. Cycle continues
```

This chaining is entirely client-driven. The server does not maintain session state between steps. Each intent evaluation is independent.

---

## 7. Progress Notifications

### 7.1 Purpose

For long-running macro-tool executions, servers MAY send `progress` messages to keep clients informed of execution status. Progress messages are OPTIONAL -- clients MUST NOT require them, and servers MAY omit them entirely.

### 7.2 Progress Message

```json
{
  "type": "progress",
  "id": "<request id>",
  "manglecp": "2026-02-draft",
  "payload": {
    "status": "<string>",
    "detail": "<string | null>",
    "percent": "<number | null>"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | REQUIRED | Current execution phase: `"started"`, `"running"`, `"finalizing"` |
| `detail` | string | OPTIONAL | Human-readable description of what is currently happening |
| `percent` | number | OPTIONAL | Estimated completion percentage (0-100). MAY be absent if progress is indeterminate. |

### 7.3 Transport Constraints

- Progress messages are ONLY available on **session transports** (WebSocket, stdio). They are NOT available on HTTP request/response transport.
- The `id` field MUST match the `id` of the `invoke_request` that triggered the execution.
- A server MUST still send a final `invoke_response` (or `error`) message after any progress messages. Progress messages do not replace the response.
- Clients MUST ignore `progress` messages for unknown `id` values.

### 7.4 Example

```json
{"type": "progress", "id": "inv-001", "manglecp": "2026-02-draft", "payload": {"status": "running", "detail": "Correlating network requests with console errors", "percent": 40}}
{"type": "progress", "id": "inv-001", "manglecp": "2026-02-draft", "payload": {"status": "running", "detail": "Tracing DOM impact", "percent": 75}}
{"type": "progress", "id": "inv-001", "manglecp": "2026-02-draft", "payload": {"status": "finalizing", "detail": "Assembling causal chain", "percent": 95}}
```

Followed by the final `invoke_response` with `id: "inv-001"`.

---

## 8. Streaming Invocation (Informative, v2)

For richer streaming semantics beyond progress notifications, a future protocol version MAY define:

- **Server-Sent Events (SSE):** Over HTTP, servers could use SSE to stream observability events during execution, followed by the final invoke response.
- **WebSocket streaming:** Structured streaming of partial results during execution.

Full streaming is NOT part of this draft specification. The `progress` message type (Section 7) provides a minimal, non-breaking interim solution for long-running operations.

---

## 9. Example: Full Invocation Cycle

### Invoke Request

```json
{
  "type": "invoke_request",
  "id": "inv-001",
  "manglecp": "2026-02-draft",
  "payload": {
    "macro_id": "diag-causal-chain-001",
    "args": {
      "error_id": "console-error-3",
      "include_network": true
    },
    "eval_time": "2026-02-19T14:30:10Z"
  }
}
```

### Invoke Response

```json
{
  "type": "invoke_response",
  "id": "inv-001",
  "manglecp": "2026-02-draft",
  "payload": {
    "result": {
      "causal_chain": [
        {"step": 1, "event": "GET /api/users returned 404", "timestamp": "2026-02-19T14:29:59Z"},
        {"step": 2, "event": "UserList component received null data", "timestamp": "2026-02-19T14:30:00Z"},
        {"step": 3, "event": "TypeError: Cannot read properties of null", "timestamp": "2026-02-19T14:30:00Z"}
      ],
      "root_cause": "API endpoint /api/users not registered in router",
      "confidence": 0.92
    },
    "state_delta": {
      "assert": [
        {"pred": "diagnosed_error", "args": ["console-error-3", "missing_route"], "category": "server", "source": {"source_type": "server"}},
        {"pred": "fix_candidate", "args": ["src/routes/router.go", "add_user_route"], "category": "derived", "source": {"source_type": "derived"}}

      ],
      "retract": [
        {"pred": "undiagnosed_error", "args": ["console-error-3"]}
      ]
    },
    "observability": {
      "summary": "Traced causal chain from API 404 to console TypeError through 3 steps. Root cause: missing route handler.",
      "events": [
        {"action": "query_console_errors", "status": "success", "duration_ms": 5},
        {"action": "correlate_network", "status": "success", "duration_ms": 12},
        {"action": "trace_dom_impact", "status": "success", "duration_ms": 8},
        {"action": "derive_root_cause", "status": "success", "duration_ms": 3}
      ],
      "duration_ms": 28
    },
    "next": {
      "suggested_intents": [
        {
          "name": "fix_error",
          "params": {"file": "src/routes/router.go", "fix_type": "add_user_route"},
          "description": "The diagnosed root cause has a straightforward fix: add the missing route handler."
        },
        {
          "name": "observe",
          "params": {"focus": "network"},
          "description": "Check if other API endpoints are also missing."
        }
      ],
      "continuation_facts": [
        {"pred": "diagnosed_error", "args": ["console-error-3", "missing_route"]},
        {"pred": "fix_candidate", "args": ["src/routes/router.go", "add_user_route"]}
      ]
    }
  }
}
```
