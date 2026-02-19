# MangleCP Specification 09: Error Handling and Diagnostics

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), Spec 02 (Wire Format), Spec 05 (Intent Evaluation)

---

## 1. Purpose

This document specifies MangleCP's error model, including error message structure, standard error codes, validation error reporting, proof hints for debugging, and rule coverage diagnostics. Errors are first-class protocol messages, not HTTP-level artifacts.

---

## 2. Error Message

### 2.1 Error Message Structure

Errors are delivered in the standard MangleCP message envelope:

```json
{
  "type": "error",
  "id": "<string | null>",
  "manglecp": "2026-02-draft",
  "payload": {
    "code": "<string>",
    "message": "<string>",
    "details": {},
    "recoverable": "<boolean>",
    "retry_after_ms": "<integer | null>"
  }
}
```

### 2.2 Error Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | REQUIRED | Machine-readable error code from the standard registry (Section 3) |
| `message` | string | REQUIRED | Human-readable error description |
| `details` | object | OPTIONAL | Structured error details specific to the error code |
| `recoverable` | boolean | REQUIRED | Whether the client can retry or modify the request to succeed |
| `retry_after_ms` | integer \| null | OPTIONAL | If recoverable, suggested wait time before retry |

### 2.3 Error Correlation

- If the error corresponds to a specific request, the envelope's `id` MUST match the request's `id`.
- If the error is server-initiated (not correlated to a request), `id` MUST be `null`.

---

## 3. Standard Error Codes

### 3.1 Error Code Registry

#### Protocol Errors (1xx range conceptually)

| Code | HTTP Status | Recoverable | Description |
|------|-------------|-------------|-------------|
| `"unsupported_version"` | 400 | Yes | Client's protocol version not supported |
| `"malformed_message"` | 400 | No | Message does not conform to envelope schema |
| `"message_too_large"` | 413 | Yes | Message exceeds `limits.max_message_bytes` |
| `"invalid_type"` | 400 | No | Unknown message type |

#### Authentication Errors (2xx range conceptually)

| Code | HTTP Status | Recoverable | Description |
|------|-------------|-------------|-------------|
| `"auth_required"` | 401 | Yes | Authentication required but not provided |
| `"auth_invalid"` | 401 | Yes | Authentication token is invalid or expired |
| `"auth_insufficient"` | 403 | No | Authenticated but lacks required permissions |

#### Fact Validation Errors (3xx range conceptually)

| Code | HTTP Status | Recoverable | Description |
|------|-------------|-------------|-------------|
| `"invalid_facts"` | 400 | Yes | One or more facts fail validation |
| `"unknown_predicate"` | 400 | Yes | Fact uses a predicate not declared in the facts profile |
| `"arity_mismatch"` | 400 | Yes | Fact has wrong number of arguments for its predicate |
| `"type_mismatch"` | 400 | Yes | Fact argument type does not match predicate schema |
| `"reserved_predicate"` | 400 | No | Fact uses the reserved `_manglecp_` prefix |
| `"too_many_facts"` | 400 | Yes | Exceeds `limits.max_facts_per_request` |

#### Evaluation Errors (4xx range conceptually)

| Code | HTTP Status | Recoverable | Description |
|------|-------------|-------------|-------------|
| `"evaluation_timeout"` | 408 | Yes | Evaluation exceeded `max_compute_ms` |
| `"derivation_limit_exceeded"` | 413 | Yes | Exceeded `max_facts_created` during evaluation |
| `"interval_limit_exceeded"` | 413 | Yes | Exceeded `max_intervals_per_atom` |
| `"invalid_temporal_pattern"` | 400 | No | Dangerous temporal recursion detected in rules |
| `"evaluation_failed"` | 500 | No | Internal evaluation error (rule compilation, stratification failure) |

#### Invocation Errors (5xx range conceptually)

| Code | HTTP Status | Recoverable | Description |
|------|-------------|-------------|-------------|
| `"macro_not_found"` | 404 | Yes | `macro_id` not found (evicted or never existed) |
| `"macro_expired"` | 410 | Yes | Macro-tool's validity window has passed |
| `"schema_validation_failed"` | 400 | Yes | Invoke args don't match input schema |
| `"confirmation_required"` | 403 | Yes | Tool requires user confirmation; no token provided |
| `"confirmation_invalid"` | 403 | Yes | Confirmation token invalid or expired |
| `"execution_failed"` | 500 | No | Macro-tool execution failed |

#### Server State Errors (6xx range conceptually)

| Code | HTTP Status | Recoverable | Description |
|------|-------------|-------------|-------------|
| `"server_not_ready"` | 503 | Yes | Server is in `"booting"` or `"draining"` status |
| `"rate_limited"` | 429 | Yes | Too many requests; retry after `retry_after_ms` |
| `"internal_error"` | 500 | No | Unexpected server error |
| `"cancelled"` | 499 | No | Request was cancelled by client |

### 3.2 Custom Error Codes

Servers MAY define custom error codes prefixed with `x-` (e.g., `"x-browser_not_launched"`, `"x-database_unavailable"`). Custom codes MUST include a descriptive `message` field.

---

## 4. Structured Error Details

### 4.1 Fact Validation Details

When `code` is `"invalid_facts"`, `"unknown_predicate"`, `"arity_mismatch"`, or `"type_mismatch"`, the `details` object SHOULD include:

```json
{
  "violations": [
    {
      "fact_index": 0,
      "predicate": "console_event",
      "issue": "arity_mismatch",
      "expected_arity": 4,
      "actual_arity": 3,
      "message": "Predicate 'console_event' expects 4 arguments, got 3"
    },
    {
      "fact_index": 2,
      "predicate": "unknown_pred",
      "issue": "unknown_predicate",
      "message": "Predicate 'unknown_pred' is not declared in the server's facts profile",
      "suggestion": "Did you mean 'user_intent'?"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `violations[].fact_index` | integer | REQUIRED | 0-based index of the offending fact in the request's `facts` array |
| `violations[].predicate` | string | REQUIRED | The predicate that failed validation |
| `violations[].issue` | string | REQUIRED | Specific validation issue type |
| `violations[].message` | string | REQUIRED | Human-readable description |
| `violations[].expected_arity` | integer | OPTIONAL | Expected argument count |
| `violations[].actual_arity` | integer | OPTIONAL | Actual argument count |
| `violations[].expected_type` | string | OPTIONAL | Expected argument type |
| `violations[].actual_type` | string | OPTIONAL | Actual argument type |
| `violations[].argument_index` | integer | OPTIONAL | 0-based index of the problematic argument |
| `violations[].suggestion` | string | OPTIONAL | Suggested correction |

### 4.2 Schema Validation Details

When `code` is `"schema_validation_failed"`, the `details` object SHOULD include JSON Schema validation errors:

```json
{
  "schema_errors": [
    {
      "path": "/phase_id",
      "message": "Required property 'phase_id' is missing",
      "keyword": "required"
    },
    {
      "path": "/dry_run",
      "message": "Expected boolean, got string",
      "keyword": "type"
    }
  ]
}
```

### 4.3 Evaluation Budget Details

When `code` is `"derivation_limit_exceeded"`, `"interval_limit_exceeded"`, or `"evaluation_timeout"`, the `details` object SHOULD include:

```json
{
  "budget": {
    "limit": 100000,
    "consumed": 100000,
    "unit": "derived_facts"
  },
  "partial_results_available": false,
  "suggestion": "Reduce the number of facts in the request, or increase max_facts_created in constraints."
}
```

### 4.4 Version Negotiation Details

When `code` is `"unsupported_version"`, the `details` object MUST include:

```json
{
  "requested_version": "2025-01-draft",
  "supported_versions": ["2026-02-draft"]
}
```

---

## 5. Proof Hints for Debugging

### 5.1 Purpose

Proof hints help clients and developers understand **why** a particular set of macro-tools was returned (or why a request produced unexpected results). They are the protocol's explainability mechanism, derived from codeNERD's proof tree tracer.

### 5.2 Requesting Proof Hints

Clients request proof hints by setting `options.include_proof_hints: true` in the intent request. Proof hints are OPTIONAL -- servers MAY support them but are not required to.

### 5.3 Proof Hint Structure

```json
{
  "proof_hints": {
    "derivations": [
      {
        "macro_id": "<string>",
        "selected_by": "<phase name>",
        "score": "<number>",
        "supporting_facts": [ /* relevant input facts */ ],
        "rule_chain": ["<predicate>", ...],
        "tree": {
          "fact": "<string representation>",
           "type": "<server | derived>",
          "children": [ /* recursive */ ]
        }
      }
    ],
    "excluded": [
      {
        "candidate": "<tool name or id>",
        "excluded_by": "<phase name>",
        "reason": "<string>"
      }
    ],
    "unmatched_facts": [
      {
        "pred": "<string>",
        "reason": "No rules reference this predicate"
      }
    ]
  }
}
```

### 5.4 Proof Hint Fields

#### Derivations

Each derivation explains why a macro-tool was selected:

| Field | Type | Description |
|-------|------|-------------|
| `macro_id` | string | The macro-tool this derivation explains |
| `selected_by` | string | Pipeline phase that selected it: `"skeleton"`, `"flesh"`, `"dependency"` |
| `score` | number | Relevance score |
| `supporting_facts` | array of Facts | Client-supplied facts that contributed to selection |
| `rule_chain` | array of strings | Predicate names in the derivation chain (NOT full rule bodies) |
| `tree` | object | OPTIONAL: Full derivation tree with server/derived classification at each node |

#### Exclusions

Each exclusion explains why a candidate capability was NOT included:

| Field | Type | Description |
|-------|------|-------------|
| `candidate` | string | Name or ID of the excluded capability |
| `excluded_by` | string | Pipeline phase that excluded it |
| `reason` | string | Human-readable reason |

#### Unmatched Facts

Facts supplied by the client that were not used by any rule during evaluation:

| Field | Type | Description |
|-------|------|-------------|
| `pred` | string | Predicate name of the unused fact |
| `reason` | string | Why it was unused (no matching rules, wrong arity, etc.) |

### 5.5 Security Constraints on Proof Hints

Proof hints MUST NOT expose:
- Raw Mangle rule source code
- Internal server file paths
- Security policy rule details
- Credentials or sensitive configuration

Proof hints SHOULD expose:
- Predicate names (already visible in the facts profile)
- Phase names (standard protocol phases)
- Relevance scores
- Derivation chains at the predicate level

---

## 6. Rule Coverage Diagnostics (Optional Extension)

### 6.1 Purpose

Rule coverage reporting enables server operators to understand how well the server's rules cover the current fact state. This formalizes spec-generator's rule status self-documentation pattern (WORKING / PARTIALLY ALIVE / DEAD).

> **Status:** Rule coverage diagnostics are an OPTIONAL extension. They are NOT part of the core protocol. Servers MAY implement them for development and debugging purposes. Clients MUST NOT depend on rule coverage data for protocol correctness.

### 6.2 Rule Coverage in Diagnostics

The `diagnostics` object in intent responses (Spec 05 Section 4.4) MAY be extended with rule coverage information:

```json
{
  "diagnostics": {
    "facts_evaluated": 42,
    "facts_derived": 120,
    "rules_fired": 15,
    "phases_completed": 6,
    "rule_coverage": {
      "total_rules": 30,
      "rules_fired": 15,
      "rules_dormant": 10,
      "rules_inapplicable": 5,
      "dormant_details": [
        {
          "rule": "derived_surface",
          "missing_predicates": ["data_entity", "data_served_by"],
          "status": "partially_alive"
        },
        {
          "rule": "pipeline_dependency",
          "missing_predicates": ["pipeline_layer"],
          "status": "dead"
        }
      ]
    }
  }
}
```

### 6.3 Rule Status Classification

| Status | Meaning |
|--------|---------|
| `"fired"` | The rule matched at least one set of facts and produced derived facts |
| `"partially_alive"` | Some (but not all) of the rule's body predicates have matching facts |
| `"dead"` | None of the rule's body predicates have matching facts |
| `"inapplicable"` | The rule is structurally valid but not relevant to the current intent type |

### 6.4 Coverage Reporting as Optional

Rule coverage diagnostics are OPTIONAL. Servers SHOULD support them in development/debug modes. Servers MAY omit them in production for performance reasons.

---

## 7. Error Recovery Patterns

### 7.1 Recoverable Errors

For errors with `recoverable: true`, clients SHOULD:

| Error Code | Recovery Action |
|------------|-----------------|
| `"unsupported_version"` | Re-send with a supported version from `details.supported_versions` |
| `"invalid_facts"` | Fix facts per `details.violations` and re-send |
| `"too_many_facts"` | Reduce fact count and re-send |
| `"evaluation_timeout"` | Reduce fact count or increase `max_compute_ms` and re-send |
| `"derivation_limit_exceeded"` | Reduce fact count or increase `max_facts_created` and re-send |
| `"macro_not_found"` | Re-evaluate the intent to get fresh macro-tools |
| `"macro_expired"` | Re-evaluate the intent to get fresh macro-tools |
| `"schema_validation_failed"` | Fix args per `details.schema_errors` and re-invoke |
| `"confirmation_required"` | Obtain user confirmation and re-invoke with token |
| `"server_not_ready"` | Wait for `retry_after_ms` then retry |
| `"rate_limited"` | Wait for `retry_after_ms` then retry |

### 7.2 Non-Recoverable Errors

For errors with `recoverable: false`, clients SHOULD:

- Log the error for debugging
- Display the error message to the user
- Not retry the same request without modification

### 7.3 Retry Semantics

When `retry_after_ms` is present:
- Clients MUST wait at least `retry_after_ms` before retrying
- Clients SHOULD implement exponential backoff for repeated failures
- Clients MUST NOT retry more than 5 times for the same request

---

## 8. Example: Fact Validation Error

### Request with invalid facts

```json
{
  "type": "intent_request",
  "id": "req-bad",
  "manglecp": "2026-02-draft",
  "payload": {
    "intent": {"name": "observe"},
    "facts": [
      {"pred": "current_url", "args": ["https://example.com"]},
      {"pred": "console_event", "args": ["s1", "error", "TypeError"]},
      {"pred": "_manglecp_internal", "args": ["hack"]}
    ]
  }
}
```

### Error Response

```json
{
  "type": "error",
  "id": "req-bad",
  "manglecp": "2026-02-draft",
  "payload": {
    "code": "invalid_facts",
    "message": "2 fact validation errors",
    "details": {
      "violations": [
        {
          "fact_index": 1,
          "predicate": "console_event",
          "issue": "arity_mismatch",
          "expected_arity": 4,
          "actual_arity": 3,
          "message": "Predicate 'console_event' expects 4 arguments (session_id, level, message, timestamp), got 3"
        },
        {
          "fact_index": 2,
          "predicate": "_manglecp_internal",
          "issue": "reserved_predicate",
          "message": "Predicates starting with '_manglecp_' are reserved for protocol use"
        }
      ]
    },
    "recoverable": true,
    "retry_after_ms": null
  }
}
```
