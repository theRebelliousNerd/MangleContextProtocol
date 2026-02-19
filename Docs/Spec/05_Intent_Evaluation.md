# MangleCP Specification 05: Intent Evaluation

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), Spec 03 (Data Model), Spec 04 (Manifest)

---

## 1. Purpose

This document specifies the intent evaluation layer -- the core of MangleCP. It defines the intent request/response payloads, the behavioral requirements that evaluation results must satisfy, evaluation constraints, and progressive disclosure semantics.

This is the "Middle Doll" of the Matryoshka architecture: the client declares what it wants to accomplish, and the server returns the mathematically constrained set of capabilities required to accomplish it.

---

## 2. Intent Request

### 2.1 Intent Request Payload

```json
{
  "intent": {
    "name": "<string>",
    "params": {}
  },
  "facts": [ /* array of Fact objects (Spec 03) */ ],
  "eval_time": "<Time | null>",
  "constraints": {
    "max_facts_created": "<integer | null>",
    "max_intervals_per_atom": "<integer | null>",
    "max_compute_ms": "<integer | null>",
    "max_tools_returned": "<integer | null>",
    "max_tokens_budget": "<integer | null>"
  },
  "options": {
    "include_proof_hints": "<boolean>",
    "disclosure_preference": "<string>"
  }
}
```

### 2.2 Intent Request Fields

#### Intent Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `intent.name` | string | REQUIRED | The intent type. SHOULD match one of the intents declared in the manifest. |
| `intent.params` | object | OPTIONAL | High-level intent parameters. These are NOT atomic tool arguments; they describe the desired outcome. |

Intent params are domain-specific. Examples:

```json
// Browser automation
{"name": "diagnose_error", "params": {"severity": "critical"}}

// Codebase analysis
{"name": "find_data_surface", "params": {"entity": "UserProfile"}}

// Workflow orchestration
{"name": "get_ready_phases", "params": {"spec_id": "spec-2026-001"}}
```

#### Facts Array

An array of Fact objects (Spec 03 Section 2). These are the client's assertions about current state. All client-supplied facts are treated as `"session"` category and lowest trust level.

#### Evaluation Time

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `eval_time` | Time | RECOMMENDED | The reference time for temporal operator evaluation. Maps to Mangle's `now`. |

If omitted, the server uses its wall clock. The server MUST report the actual evaluation time used in the response.

#### Constraints

Client-supplied resource budgets that the server MUST NOT exceed:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_facts_created` | integer | Server's `limits.max_derived_facts` | Maximum derived facts during evaluation |
| `max_intervals_per_atom` | integer | Server's `limits.max_intervals_per_atom` | Maximum temporal intervals per atom |
| `max_compute_ms` | integer | Server's `limits.max_compute_ms` | Maximum evaluation time |
| `max_tools_returned` | integer | No limit | Maximum number of macro-tools in the response |
| `max_tokens_budget` | integer | No limit | Maximum estimated token cost for the returned macro-tools |

If a client-supplied constraint is more restrictive than the server's limit, the server MUST use the client's value. If less restrictive, the server MUST use its own limit.

#### Options

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `include_proof_hints` | boolean | `false` | Request derivation traces in the response |
| `disclosure_preference` | string | `"adaptive"` | Progressive disclosure strategy: `"full"`, `"condensed"`, `"minimal"`, `"adaptive"` |

---

## 3. Evaluation Behavioral Contracts

### 3.1 Overview

When a server receives an intent request, it MUST:

1. **Validate** the request (schema, constraints, authentication)
2. **Prepare** an isolated evaluation context (the server MUST NOT carry state from prior evaluations)
3. **Inject** client-supplied facts as session-scoped entries
4. **Evaluate** the intent against its rules and fact state
5. **Render** the evaluation results into MacroTool objects with disclosure levels
6. **Respond** with the intent response

The protocol specifies the **properties** the output must satisfy, not the internal algorithm. Servers MAY use any evaluation strategy (Mangle rule strata, procedural code, hybrid approaches) as long as the output contracts in Section 3.2 are met.

### 3.2 Output Contracts

The set of MacroTool objects returned by an intent evaluation MUST satisfy all of the following properties:

#### Contract 1: Intent Relevance

Every returned macro-tool MUST be relevant to the declared intent. A server MUST NOT return macro-tools for unrelated capabilities. The server's rules or logic define what "relevant" means for each intent type.

#### Contract 2: Prohibition Enforcement

If the server's policy rules prohibit a capability in the current context (safety rules, incompatibility, missing prerequisites, client restrictions), that capability MUST NOT appear in the response. Prohibition MUST override any positive relevance signal.

#### Contract 3: Conflict Freedom

The returned set MUST be conflict-free. If two capabilities serve the same purpose and conflict with each other, only one MUST appear. The server MUST resolve conflicts deterministically -- given the same input facts and rules, the same capability MUST be selected.

#### Contract 4: Dependency Completeness

If a returned macro-tool depends on another capability being available, that dependency MUST also be present in the response, OR the dependent macro-tool MUST be removed. The returned set MUST be self-consistent.

#### Contract 5: Constraint Compliance

The evaluation MUST NOT exceed the resource constraints specified in the request or the server's own limits (Section 3.3). If a constraint is hit before evaluation completes, the server MUST return an error rather than partial results.

#### Contract 6: Determinism

Given identical inputs (intent, facts, eval_time, constraints) and identical server state (rules, base facts), the server MUST produce identical output: same macro-tools, same disclosure levels, same scores. See Section 6 for detailed determinism requirements.

### 3.3 Evaluation Constraints Enforcement

During evaluation, the server MUST monitor resource usage to prevent denial-of-service or infinite loops (especially runaway temporal recursions, such as a rule that infinitely derives facts at `T+1`):

| Constraint | Enforcement | Error Code |
|------------|-------------|------------|
| `max_facts_created` | Stop derivation when limit reached | `"derivation_limit_exceeded"` |
| `max_intervals_per_atom` | Coalesce or reject when limit reached | `"interval_limit_exceeded"` |
| `max_compute_ms` | Abort evaluation on timeout | `"evaluation_timeout"` |

**Runaway Temporal Recursions:** Due to DatalogMTL's expressive power, poorly written temporal rules can generate infinite sequences (e.g., `A@[T+1] :- A@[T]`). The `max_intervals_per_atom` constraint MUST be enforced by the server's `TemporalStore` to detect and halt such escalations before memory exhaustion.

When a constraint is hit, the server MUST return an error response (Spec 09). The server SHOULD NOT return partial results from an incomplete evaluation.

---

## 4. Intent Response

### 4.1 Intent Response Payload

```json
{
  "eval_time_used": "<Time>",
  "eval_duration_ms": "<integer>",
  "macro_tools": [ /* array of MacroTool objects (Spec 06) */ ],
  "proof_hints": "<object | null>",
  "required_skills": [ /* array of Skill objects (Spec 06) */ ],
  "diagnostics": {
    "facts_evaluated": "<integer>",
    "facts_derived": "<integer>",
    "rules_fired": "<integer>",
    "phases_completed": "<integer | null>"
  }
}
```

### 4.2 Response Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `eval_time_used` | Time | REQUIRED | The evaluation time actually used (echoed or server-chosen) |
| `eval_duration_ms` | integer | RECOMMENDED | Wall-clock milliseconds the evaluation took |
| `macro_tools` | array | REQUIRED | MacroTool objects computed for this intent. MAY be empty. |
| `proof_hints` | object \| null | OPTIONAL | Present only if `include_proof_hints` was `true` in the request |
| `required_skills` | array | OPTIONAL | Skills proven necessary for the selected macro-tools |
| `diagnostics` | object | OPTIONAL | Evaluation statistics |

### 4.3 Proof Hints

When requested, proof hints provide a minimal explanation of why each macro-tool was selected. Proof hints MUST NOT expose server-internal chain-of-thought, raw Mangle source, or security-sensitive policy rules.

```json
{
  "derivations": [
    {
      "macro_id": "diag-console-001",
      "selected_by": "skeleton",
      "score": 100,
      "supporting_facts": [
        {"pred": "console_event", "args": ["s1", "error", "TypeError: null", 1771510200000]}
      ],
      "rule_chain": ["mandatory_for_intent", "has_console_errors"]
    }
  ]
}
```

The `rule_chain` field lists predicate names (not full rule bodies) that participated in the derivation. This is aligned with codeNERD's proof tree tracer pattern: showing which rules fired without exposing rule internals.

### 4.4 Diagnostics

Diagnostics are informational and MUST NOT affect protocol behavior:

| Field | Type | Description |
|-------|------|-------------|
| `facts_evaluated` | integer | Total facts in the store at evaluation start (server + session) |
| `facts_derived` | integer | Number of derived facts created during evaluation |
| `rules_fired` | integer | Number of rule activations during evaluation |
| `phases_completed` | integer \| null | Number of internal evaluation phases completed (implementation-defined) |

---

## 5. Progressive Disclosure

### 5.1 Disclosure Levels

Each MacroTool in the response carries a `disclosure_level` field (Spec 06) indicating how much detail the server included:

| Level | Content | Token Cost | When Used |
|-------|---------|------------|-----------|
| `"full"` | Complete name, description, input/output schemas, context injection, safety metadata | High (~200-500 tokens) | Highest-relevance tools |
| `"condensed"` | Name and one-line description | Low (~20 tokens) | Medium-relevance tools |
| `"minimal"` | Name only | Minimal (~5 tokens) | Low-relevance tools included for awareness |

### 5.2 Disclosure Strategy

The `disclosure_preference` option in the intent request controls the strategy:

| Preference | Server Behavior |
|------------|-----------------|
| `"full"` | All tools at full disclosure |
| `"condensed"` | All tools at condensed disclosure |
| `"minimal"` | All tools at minimal disclosure |
| `"adaptive"` (default) | Server assigns disclosure levels based on relevance scores from evaluation |

For `"adaptive"` mode, servers SHOULD use score thresholds derived from their relevance scoring:

- Score >= 70: `"full"` disclosure
- Score >= 40 and < 70: `"condensed"` disclosure
- Score >= 20 and < 40: `"minimal"` disclosure
- Score < 20: excluded from response

If `max_tokens_budget` is provided in the constraints, servers MUST perform **Token Budget Fitting**: sorting tools by relevance score and progressively demoting the lowest-scoring tools (e.g., from `"full"` to `"condensed"`, or `"condensed"` to `"minimal"`) until the estimated total token cost of all tools fits within the budget.

These thresholds and fitting logic are RECOMMENDED defaults. Servers MAY adjust them based on implementation specifics.

### 5.3 Disclosure Upgrade

A client receiving a `"condensed"` or `"minimal"` tool MAY request full details by sending a new intent request with the same intent but adding a fact:

```json
{"pred": "disclosure_upgrade", "args": ["<macro_id>"]}
```

The server SHOULD return the specified tool at `"full"` disclosure level.

---

## 6. Determinism Requirements

### 6.1 Deterministic Evaluation

Given identical inputs (intent, facts, eval_time, constraints) and identical server state (base facts, rules), a MangleCP server MUST produce identical output:

- The same set of macro-tools MUST be selected
- The same disclosure levels MUST be assigned
- The same scores MUST be computed

This is a natural consequence of Mangle's deterministic evaluation semantics and the fresh-store-per-evaluation model.

### 6.2 Non-Determinism Sources

The following are permitted sources of variation between evaluations:

- **Different `eval_time`**: Temporal rules may select different facts
- **Different server base facts**: The server's base facts may have changed between requests
- **Different external predicate results**: FFI bridges may return different data
- **Non-deterministic external predicates**: Servers MUST document which external predicates are non-deterministic in the manifest

---

## 7. Example Intent Evaluation

### Request

```json
{
  "type": "intent_request",
  "id": "req-001",
  "manglecp": "2026-02-draft",
  "payload": {
    "intent": {
      "name": "diagnose_error",
      "params": {"severity": "critical"}
    },
    "facts": [
      {"pred": "current_url", "args": ["https://app.example.com/dashboard"]},
      {"pred": "console_event", "args": ["s1", "error", "TypeError: Cannot read properties of null", 1771510200000], "t": {"at": "2026-02-19T14:30:00Z"}},
      {"pred": "net_request", "args": ["s1", "r42", "GET", "https://api.example.com/users", 1771510199000], "t": {"start": "2026-02-19T14:29:59Z", "end": "2026-02-19T14:30:01Z"}}
    ],
    "eval_time": "2026-02-19T14:30:05Z",
    "constraints": {
      "max_compute_ms": 5000,
      "max_tools_returned": 5
    },
    "options": {
      "include_proof_hints": true,
      "disclosure_preference": "adaptive"
    }
  }
}
```

### Response

```json
{
  "type": "intent_response",
  "id": "req-001",
  "manglecp": "2026-02-draft",
  "payload": {
    "eval_time_used": "2026-02-19T14:30:05Z",
    "eval_duration_ms": 47,
    "macro_tools": [
      {
        "macro_id": "diag-causal-chain-001",
        "name": "diagnose_causal_chain",
        "description": "Analyze the causal chain from the failed API request to the console TypeError, including network timing and DOM impact.",
        "disclosure_level": "full",
        "input_schema": {
          "type": "object",
          "properties": {
            "error_id": {"type": "string"},
            "include_network": {"type": "boolean", "default": true}
          },
          "required": ["error_id"]
        },
        "context_injection": {
          "instructions": "Correlate the GET /users 404 response with the subsequent TypeError. Check if the component rendering depends on the API response data.",
          "skills": []
        },
        "safety": {
          "requires_user_confirmation": false,
          "side_effects": []
        },
        "validity": {
          "not_before": "2026-02-19T14:30:05Z",
          "expires_at": "2026-02-19T14:35:05Z"
        }
      }
    ],
    "proof_hints": {
      "derivations": [
        {
          "macro_id": "diag-causal-chain-001",
          "selected_by": "skeleton",
          "score": 100,
          "supporting_facts": [
            {"pred": "console_event", "args": ["s1", "error", "TypeError: Cannot read properties of null", 1771510200000]},
            {"pred": "net_request", "args": ["s1", "r42", "GET", "https://api.example.com/users", 1771510199000]}
          ],
          "rule_chain": ["mandatory_for_intent", "has_console_errors", "has_failed_request", "causal_chain_candidate"]
        }
      ]
    },
    "required_skills": [],
    "diagnostics": {
      "facts_evaluated": 3,
      "facts_derived": 12,
      "rules_fired": 8,
      "phases_completed": 6
    }
  }
}
```
