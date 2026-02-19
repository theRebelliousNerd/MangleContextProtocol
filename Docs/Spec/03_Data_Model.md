# MangleCP Specification 03: Data Model

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), Spec 02 (Wire Format), Mangle temporal.md

---

## 1. Purpose

This document specifies MangleCP's data model: the JSON encoding of facts (atemporal and temporal), the type system for fact arguments, fact provenance metadata, and the relationship between MangleCP's wire-format facts and Mangle's internal representations.

---

## 2. Fact Object

A Fact is the fundamental unit of state in MangleCP. All assertions, observations, and derived knowledge are expressed as facts.

### 2.1 Fact JSON Structure

```json
{
  "pred": "<string>",
  "args": [ "<value>", ... ],
  "t": "<temporal annotation | null>",
  "source": "<provenance annotation | null>",
  "category": "<string | null>"
}
```

### 2.2 Fact Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pred` | string | REQUIRED | The predicate symbol. MUST match `[a-z][a-z0-9_]*` (lowercase, underscores, no dots). |
| `args` | array | REQUIRED | Ordered arguments to the predicate. MAY be empty for 0-arity predicates. |
| `t` | object \| null | OPTIONAL | Temporal annotation. If absent or null, the fact is atemporal. See Section 4. |
| `source` | object \| null | OPTIONAL | Provenance annotation. See Section 5. |
| `category` | string \| null | OPTIONAL | Fact category: `"server"`, `"session"`, or `"derived"`. See Section 6. Servers MUST include this in `state_delta` responses. Clients MAY omit it in intent requests (defaults to `"session"`). |

### 2.3 Predicate Naming

Predicate names MUST:

- Start with a lowercase ASCII letter (`a-z`)
- Contain only lowercase ASCII letters, digits, and underscores
- Be at most 128 characters long

Predicate names SHOULD be descriptive and use snake_case: `net_request`, `user_intent`, `tool_registered`, `page_has_error`.

Predicate names MUST NOT start with `_manglecp_`. This prefix is reserved for protocol-internal predicates.

### 2.4 Named Arguments (Alternative Encoding)

As an alternative to positional `args`, facts MAY use `named_args` for improved developer ergonomics. Named arguments use the `arg_names` from the predicate's schema declaration (Spec 03 Section 8):

```json
{
  "pred": "console_event",
  "named_args": {
    "session_id": "s1",
    "level": "error",
    "message": "TypeError: Cannot read properties of null",
    "timestamp": 1771510200000
  },
  "t": {"at": "2026-02-19T14:30:00Z"}
}
```

This is semantically equivalent to the positional form:

```json
{
  "pred": "console_event",
  "args": ["s1", "error", "TypeError: Cannot read properties of null", 1771510200000],
  "t": {"at": "2026-02-19T14:30:00Z"}
}
```

**Rules:**

- A fact MUST have exactly one of `args` or `named_args`, never both.
- When `named_args` is used, the server MUST resolve argument positions using the `arg_names` from the predicate's schema declaration in the manifest's `facts_profile`.
- If a predicate has no `arg_names` declared, `named_args` MUST NOT be used for that predicate.
- All keys in `named_args` MUST correspond to declared `arg_names`. Unknown keys MUST produce an `"invalid_facts"` error.
- All required argument positions MUST be present. Missing keys MUST produce an `"invalid_facts"` error.
- Servers supporting `named_args` SHOULD advertise this in the manifest's `capabilities` via `"named_args": true`.

---

## 3. Argument Types

Fact arguments are JSON values with the following type mapping to Mangle's type system:

### 3.1 Type Mapping

| JSON Type | Mangle Type | Example JSON | Example Mangle |
|-----------|-------------|-------------|----------------|
| string | `name` (atom) or `string` | `"hello"` | `"hello"` |
| number (integer) | `int64` | `42` | `42` |
| number (float) | `float64` | `3.14` | `3.14` |
| boolean | (no native Mangle boolean) | `true` | `fn:true` |
| array | `fn:list` | `["a", "b"]` | `fn:list("a", "b")` |
| object | `fn:map` | `{"k": "v"}` | `fn:map("k", "v")` |
| null | (absent/wildcard) | `null` | `_` |

### 3.2 Integer Precision

- JSON integers MUST be encoded without a decimal point or exponent.
- Values in the range `[-(2^53 - 1), 2^53 - 1]` MUST be transmitted as JSON numbers.
- Values outside this range MUST be encoded as strings with a type wrapper: `{"_type": "int64", "value": "9007199254740993"}`.

### 3.3 Variable Arguments

In certain contexts (e.g., `facts_profile` schema declarations in the manifest), arguments may represent **variables** (unbound positions). Variables are encoded as:

```json
{"_var": "Name"}
```

Variable names MUST start with an uppercase ASCII letter, matching Mangle's convention.

### 3.4 Wildcard Arguments

A wildcard (matching any value) is encoded as JSON `null`. This maps to Mangle's `_` wildcard.

---

## 4. Temporal Annotations

MangleCP's temporal model maps directly to Mangle's DatalogMTL extension. Temporal annotations express when a fact is valid.

### 4.1 Annotation Forms

| Form | JSON Structure | Mangle Equivalent |
|------|---------------|-------------------|
| **Interval** | `{"start": <Time>, "end": <Time>}` | `@[start, end]` |
| **Point** | `{"at": <Time>}` | `@[T]` |
| **Open-ended** | `{"start": <Time>, "end": "_"}` | `@[start, _]` |
| **Unbounded start** | `{"start": "_", "end": <Time>}` | `@[_, end]` |

### 4.2 Time Representation

A `<Time>` value MUST be one of:

| Format | JSON Type | Example | Semantics |
|--------|-----------|---------|-----------|
| RFC 3339 timestamp | string | `"2026-02-19T14:30:00Z"` | Absolute time |
| Milliseconds since epoch | number | `1771510200000` | Absolute time (compact) |
| Unbounded marker | string | `"_"` | No bound on this endpoint |
| Evaluation-relative | string | `"now"` | Server's evaluation time |

Servers MUST support RFC 3339 strings. Servers MAY additionally support millisecond epoch numbers. Servers MUST advertise which time formats they accept in the manifest's `facts_profile`.

### 4.3 Temporal Operators (Informative)

Mangle's DatalogMTL defines four temporal operators that servers use internally during evaluation. These are not part of the wire format but are documented here so clients understand the server's temporal reasoning capabilities:

| Operator | Mangle Syntax | Meaning |
|----------|--------------|---------|
| Past open | `<-[D]` | "Within the last D time units (exclusive)" |
| Past closed | `[-[D]` | "Within the last D time units (inclusive)" |
| Future open | `<+[D]` | "Within the next D time units (exclusive)" |
| Future closed | `[+[D]` | "Within the next D time units (inclusive)" |

Duration bounds `D` are expressed in Mangle syntax as `Nd` (days), `Nh` (hours), `Nm` (minutes), `Ns` (seconds), or raw milliseconds.

Clients do not send temporal operator expressions over the wire. They send timestamped facts; the server's `.mg` rules contain the temporal operators.

### 4.4 Evaluation Time

Every intent evaluation occurs relative to an **evaluation time**:

- If the client supplies `eval_time` in the intent request, the server MUST use it as the reference point for all temporal operators.
- If the client omits `eval_time`, the server MUST use its own wall clock and MUST report the chosen time in the response's `eval_time_used` field.
- The evaluation time corresponds to Mangle's `now` keyword in temporal rules.

### 4.5 Temporal Fact Semantics

- An atemporal fact (no `t` field) is treated as **always true** -- equivalent to `@[_, _]` in Mangle.
- A temporal fact is true only during its annotated interval.
- When a server returns temporal facts in a `state_delta`, those facts carry their validity intervals. Clients feeding these facts back into subsequent intent requests preserve the temporal semantics.

### 4.6 Example: DatalogMTL Temporal Pattern

This pattern demonstrates how client-sent timestamped facts dictate server-side macro-tool selection using DatalogMTL temporal bounds:

**1. Client sends a timestamped fact:**

```json
{
  "pred": "console_event",
  "args": ["s1", "error"],
  "t": {"at": "2026-02-19T14:30:00Z"}
}
```

**2. Server injects into TemporalStore:**
The fact is stored natively with strict temporal bounds:

```mangle
console_event("s1", "error") @["2026-02-19T14:30:00Z", "2026-02-19T14:30:00Z"].
```

**3. Server rule evaluates relative to `eval_time` (`now`):**

```mangle
# "If there was a console error in the last 5 minutes, expose the diagnose tool"
macro_tool("diagnose_error", "full") :-
    <-[5m] console_event(_, "error").
```

If the client's `eval_time` is `14:34:00Z` (within 5 minutes), the `diagnose_error` tool is synthesized. If `eval_time` is `14:36:00Z`, it is excluded. This provides robust execution gating without custom Go logic.

---

## 5. Fact Provenance

Provenance tracks where a fact originated. This is essential for trust decisions, debugging, and rule coverage analysis.

### 5.1 Provenance Object

```json
{
  "source_type": "<string>",
  "source_id": "<string | null>",
  "asserted_at": "<Time | null>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source_type` | string | REQUIRED | Origin category. Standard values defined in Section 5.2. |
| `source_id` | string \| null | OPTIONAL | Identifier for the specific source (e.g., database name, scanner ID). |
| `asserted_at` | Time \| null | OPTIONAL | When the fact was asserted. Distinct from temporal validity (`t`). |

### 5.2 Standard Source Types

| Source Type | Meaning | Trust Implication |
|-------------|---------|-------------------|
| `"server"` | Asserted by the server's own logic or configuration | Highest trust |
| `"scan"` | Generated by automated scanning (codebase scan, DB introspection) | High trust |
| `"external"` | From an external data source (API, database, MCP server) | Medium trust |
| `"client"` | Asserted by the client in an intent request | Lowest trust (session-scoped) |
| `"derived"` | Computed by rule evaluation | Trust determined by input fact trust |

Servers MAY define additional source types. Custom source types MUST be prefixed with `x-` (e.g., `"x-arango_live"`).

### 5.3 Provenance in Wire Messages

- **Client -> Server (intent request):** Provenance is OPTIONAL on client-asserted facts. If omitted, the server MUST treat the fact as `source_type: "client"`.
- **Server -> Client (state_delta):** Provenance is RECOMMENDED. Servers SHOULD include provenance on all returned facts so clients can make trust-informed decisions.
- Provenance MUST NOT affect evaluation semantics. Two facts with the same predicate and arguments but different provenance are **identical** for rule evaluation purposes. Provenance is metadata, not part of the fact's logical identity.

---

## 6. Fact Categories

Fact categories determine lifecycle behavior across the protocol boundary.

### 6.1 Category Definitions

| Category | Wire Name | Lifecycle | Origin | Persistence |
|----------|-----------|-----------|--------|-------------|
| **Server** | `"server"` | Server-managed, persists across requests | Server schema, scans, configuration | Server-side |
| **Session** | `"session"` | Valid for one intent evaluation cycle | Client intent request | None (discarded after evaluation) |
| **Derived** | `"derived"` | Computed during evaluation, discarded after response | Rule derivation | None |

> **Note:** These wire names are intentionally engine-agnostic. Implementations using Mangle may map `"server"` to EDB (extensional database) and `"derived"` to IDB (intensional database) internally, but the wire format uses human-readable names to improve developer ergonomics.

### 6.2 Category Rules

- Client-asserted facts in intent requests are always `"session"` category.
- Facts in `state_delta` responses MUST include a `category` field.
- Clients MUST NOT assume `"derived"` facts from a previous response remain valid in a subsequent request. To use derived facts as input to a new evaluation, clients MUST re-assert them as `"session"` facts.
- Servers MUST NOT persist `"session"` facts beyond the evaluation that consumed them.

---

## 7. Structured Fact Format (Informative, Extension)

> **Status:** This section describes an OPTIONAL extension format. It is NOT part of the core wire protocol and SHOULD be treated as implementation guidance for servers that need structured rule representation. Future protocol versions may formalize this as a standard extension.

For contexts where facts carry complex structure (e.g., `facts_profile` schema declarations, rule submission in future protocol versions), MangleCP defines an extended structured format. This format is inspired by codeNERD's `synth/compile.go` pattern and is useful for servers that support rule submission or need to exchange structured clauses.

### 7.1 Clause Object (Informative, v2)

```json
{
  "head": {
    "predicate": "<string>",
    "args": [
      {"type": "string", "value": "read"},
      {"type": "variable", "name": "Target"},
      {"type": "wildcard"}
    ]
  },
  "body": [
    {
      "predicate": "file_exists",
      "args": [{"type": "variable", "name": "Target"}]
    }
  ]
}
```

This compiles to:

```mangle
safe_action("read", Target, _) :- file_exists(Target).
```

### 7.2 Argument Type Descriptors

| Type | JSON | Mangle Equivalent |
|------|------|--------------------|
| `"string"` | `{"type": "string", "value": "x"}` | `"x"` |
| `"number"` | `{"type": "number", "value": 42}` | `42` |
| `"variable"` | `{"type": "variable", "name": "X"}` | `X` |
| `"wildcard"` | `{"type": "wildcard"}` | `_` |
| `"list"` | `{"type": "list", "elements": [...]}` | `fn:list(...)` |
| `"map"` | `{"type": "map", "entries": {...}}` | `fn:map(...)` |

---

## 8. Predicate Schema Declarations

MangleCP servers advertise the predicates they understand through schema declarations in the manifest's `facts_profile` (Spec 04). This section defines the schema declaration format.

### 8.1 Predicate Schema Object

```json
{
  "predicate": "<string>",
  "arity": "<integer>",
  "arg_types": ["<type_name>", ...],
  "arg_names": ["<string>", ...],
  "temporal": "<boolean>",
  "description": "<string>",
  "direction": "<string>"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `predicate` | string | REQUIRED | Predicate name |
| `arity` | integer | REQUIRED | Number of arguments |
| `arg_types` | array of strings | RECOMMENDED | Type names for each argument position: `"string"`, `"number"`, `"boolean"`, `"any"` |
| `arg_names` | array of strings | OPTIONAL | Human-readable names for each argument position |
| `temporal` | boolean | OPTIONAL | Whether the predicate accepts temporal annotations. Default: `false`. |
| `description` | string | OPTIONAL | Human-readable description |
| `direction` | string | OPTIONAL | `"input"` (client provides), `"output"` (server derives), `"both"` (either). Default: `"both"`. |

### 8.2 Direction Semantics

- `"input"`: The server expects the client to provide facts with this predicate in intent requests. The server does not derive facts with this predicate.
- `"output"`: The server derives facts with this predicate during evaluation. The client should not assert facts with this predicate.
- `"both"`: The predicate can be both client-asserted and server-derived.

This enables client-side validation: clients can check that the facts they are sending match `"input"` or `"both"` predicates before transmitting.
