# MangleCP Specification 06: Macro-Tools and Skills

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), Spec 03 (Data Model), Spec 05 (Intent Evaluation)

---

## 1. Purpose

This document specifies the MacroTool and Skill objects -- the primary output of MangleCP's intent evaluation. A MacroTool is a dynamically synthesized tool abstraction that bundles a validated chain of atomic actions behind a single high-level invocation schema. A Skill is a reusable instruction/resource bundle conditionally injected when proven necessary by rule evaluation.

---

## 2. MacroTool Object

### 2.1 Full Schema

```json
{
  "macro_id": "<string>",
  "name": "<string>",
  "description": "<string>",
  "disclosure_level": "<string>",
  "input_schema": {},
  "output_schema": {},
  "context_injection": {
    "instructions": "<string>",
    "skills": [ /* Skill references */ ],
    "context_resources": [ /* Context resource references */ ]
  },
  "safety": {
    "requires_user_confirmation": "<boolean>",
    "side_effects": ["<string>", ...],
    "reversible": "<boolean>",
    "idempotent": "<boolean>"
  },
  "validity": {
    "not_before": "<Time>",
    "expires_at": "<Time>"
  },
  "metadata": {
    "selection_phase": "<string>",
    "score": "<number>",
    "source_rules": ["<string>", ...]
  }
}
```

### 2.2 Field Definitions

#### Identity

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `macro_id` | string | REQUIRED | Stable identifier for this synthesized tool instance. MUST be unique within a single intent response. SHOULD be deterministic given the same evaluation inputs. |
| `name` | string | REQUIRED | Human-readable tool name presented to the client. SHOULD be descriptive and concise (max 64 characters). |
| `description` | string | CONDITIONAL | Human-readable description. REQUIRED for `"full"` disclosure. OPTIONAL for `"condensed"` (may be a one-line summary). MUST be omitted for `"minimal"`. |

#### Disclosure Level

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `disclosure_level` | string | REQUIRED | One of `"full"`, `"condensed"`, `"minimal"`. Determines which fields are present. |

**Field presence by disclosure level:**

| Field | `"full"` | `"condensed"` | `"minimal"` |
|-------|----------|---------------|-------------|
| `macro_id` | REQUIRED | REQUIRED | REQUIRED |
| `name` | REQUIRED | REQUIRED | REQUIRED |
| `description` | REQUIRED (full) | REQUIRED (one-line) | OMITTED |
| `input_schema` | REQUIRED | OMITTED | OMITTED |
| `output_schema` | OPTIONAL | OMITTED | OMITTED |
| `context_injection` | OPTIONAL | OMITTED | OMITTED |
| `safety` | REQUIRED | OMITTED | OMITTED |
| `validity` | OPTIONAL | OMITTED | OMITTED |
| `metadata` | OPTIONAL | OPTIONAL | OMITTED |

#### Input Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `input_schema` | JSON Schema | REQUIRED for `"full"` | JSON Schema (RECOMMENDED draft 2020-12) describing the invocation arguments. |

The input schema defines what the client must provide when invoking this macro-tool. It is NOT the schema of the underlying atomic operations -- those are hidden from the client.

Servers SHOULD generate focused, minimal input schemas. A macro-tool that internally chains 10 atomic operations might expose only 2-3 high-level parameters.

#### Output Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `output_schema` | JSON Schema | OPTIONAL | JSON Schema describing the expected result structure. |

When provided, clients can validate invoke responses against this schema.

#### Context Injection

Context injection bundles everything the client needs to effectively use the macro-tool. This formalizes spec-implementer's `needs_file` pattern and codeNERD's JIT context assembly.

```json
{
  "instructions": "<string>",
  "skills": [
    {"skill_id": "<string>", "inline": false},
    {"skill_id": "<string>", "inline": true, "content": "<string>"}
  ],
  "context_resources": [
    {"uri": "<string>", "relevance": "<string>"}
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `context_injection.instructions` | string | OPTIONAL | Concise, targeted instructions for using this tool effectively in the current context. |
| `context_injection.skills` | array | OPTIONAL | Skill references or inline skills proven necessary by rule evaluation. |
| `context_injection.context_resources` | array | OPTIONAL | Resource URIs that the server determined are relevant (derived from `needs_file`-style rules). Supports any URI scheme. |

##### Context Resource Object

```json
{
  "uri": "file://src/components/Dashboard.tsx",
  "relevance": "This component renders the user data that the failing API call was fetching.",
  "priority": 90
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uri` | string | REQUIRED | Resource URI. Standard schemes include `file://` for file paths, `http://`/`https://` for web resources, and custom schemes (e.g., `arangodb://`, `s3://`) for domain-specific resources. For backward compatibility, bare file paths (no scheme) SHOULD be interpreted as `file://` URIs. |
| `relevance` | string | OPTIONAL | Why this resource is relevant to the macro-tool |
| `priority` | number | OPTIONAL | Relevance score (0-100). Clients MAY use this to prioritize which resources to read first. |

#### Safety Metadata

Safety metadata enables clients to implement confirmation workflows and inform users about side effects.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `safety.requires_user_confirmation` | boolean | REQUIRED for `"full"` | Whether the client SHOULD prompt the user before invoking this tool. |
| `safety.side_effects` | array of strings | REQUIRED for `"full"` | Categories of side effects. Standard values defined in Section 2.3. |
| `safety.reversible` | boolean | OPTIONAL | Whether the action can be undone. Default: `false`. |
| `safety.idempotent` | boolean | OPTIONAL | Whether repeated invocations produce the same result. Default: `false`. |

### 2.3 Standard Side Effect Categories

| Category | Description | Examples |
|----------|-------------|----------|
| `"none"` | No side effects (pure query/observation) | Reading DOM state, querying facts |
| `"filesystem"` | Reads or writes files on the server | File creation, modification, deletion |
| `"network"` | Makes network requests | HTTP calls, WebSocket connections |
| `"database"` | Reads or writes database state | SQL queries, document mutations |
| `"browser"` | Modifies browser state | Navigation, form filling, clicking |
| `"process"` | Starts, stops, or signals processes | Subprocess execution, service restarts |
| `"payments"` | Initiates financial transactions | Payment processing, subscription changes |
| `"authentication"` | Modifies authentication state | Login, logout, token refresh |
| `"destructive"` | Irreversible data loss possible | Deletion, truncation, format |

Servers MAY use multiple categories for a single tool. Servers MAY define custom categories prefixed with `x-`.

#### Validity Window

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `validity.not_before` | Time | OPTIONAL | Earliest time this macro-tool can be invoked. Default: evaluation time. |
| `validity.expires_at` | Time | OPTIONAL | Latest time this macro-tool remains valid. After this, the client SHOULD re-evaluate the intent. |

Validity windows are derived from temporal rule evaluation. A tool synthesized from facts with a 5-minute temporal window should expire when those facts do.

If a client invokes a macro-tool after its `expires_at`, the server SHOULD return error code `"macro_expired"`.

#### Metadata

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `metadata.selection_phase` | string | OPTIONAL | Which evaluation phase selected this tool (implementation-defined, e.g. `"skeleton"`, `"flesh"`, `"dependency"`) |
| `metadata.score` | number | OPTIONAL | Numeric relevance score from evaluation |
| `metadata.source_rules` | array of strings | OPTIONAL | Predicate names that contributed to selection |

Metadata is informational. Clients MUST NOT depend on metadata fields for protocol correctness.

---

## 3. Macro-Tool Synthesis

### 3.1 How Servers Synthesize Macro-Tools

Macro-tools are NOT pre-defined tool registrations. They are dynamically assembled from the evaluation pipeline results. A server's `.mg` rules determine:

1. **What capabilities are relevant** (evaluation against behavioral contracts)
2. **How to compose them** (rule-derived action chains)
3. **What parameters to expose** (input schema generation)
4. **What context to inject** (skill and resource selection)
5. **What safety metadata to attach** (side effect derivation)

### 3.2 Composition Patterns

Servers MAY synthesize macro-tools using any of these patterns:

**Single-capability wrapper:** One atomic capability exposed with a simplified schema.

```mangle
macro_tool("observe_console", "full") :-
    intent_type(_, "diagnose_error"),
    has_console_errors.
```

**Multi-capability chain:** Multiple atomic capabilities bundled into a sequential execution plan.

```mangle
macro_tool("diagnose_causal_chain", "full") :-
    intent_type(_, "diagnose_error"),
    has_console_errors,
    has_failed_request,
    causal_chain_candidate(_, _).
```

**Conditional composition:** The chain varies based on available facts.

```mangle
macro_tool("full_stack_diagnosis", "full") :-
    intent_type(_, "diagnose_error"),
    has_console_errors,
    has_failed_request,
    has_docker_logs.  # Only include backend diagnosis if docker logs are available

macro_tool("frontend_diagnosis", "full") :-
    intent_type(_, "diagnose_error"),
    has_console_errors,
    !has_docker_logs.  # Fallback to frontend-only diagnosis
```

### 3.3 Macro-Tool Identity Stability

The `macro_id` SHOULD be deterministic: given the same intent, facts, and evaluation time, the same macro-tools SHOULD have the same IDs. This enables clients to track tool identity across re-evaluations.

Servers SHOULD generate IDs using a hash of the tool's composition (e.g., the set of capabilities it bundles) rather than random generation.

---

## 4. Skill Object

### 4.1 Skill Schema

```json
{
  "skill_id": "<string>",
  "name": "<string>",
  "description": "<string>",
  "content": "<string>",
  "content_type": "<string>",
  "resources": [
    {
      "uri": "<string>",
      "type": "<string>",
      "description": "<string>"
    }
  ]
}
```

### 4.2 Skill Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `skill_id` | string | REQUIRED | Unique skill identifier |
| `name` | string | REQUIRED | Human-readable skill name |
| `description` | string | REQUIRED | What this skill teaches/enables |
| `content` | string | REQUIRED | The skill's instruction content. For inline skills, this is the full text. For referenced skills, this MAY be a summary. |
| `content_type` | string | OPTIONAL | MIME type of the content. Default: `"text/markdown"`. |
| `resources` | array | OPTIONAL | Additional resources the skill references |

### 4.3 Skill Injection Rules

Skills are the protocol's mechanism for conditional instruction delivery. They formalize spec-implementer's skill-Mangle parallel encoding pattern:

- A server MUST NOT inject skills indiscriminately. Skills are included ONLY when a Mangle rule proves them necessary for the selected macro-tools.
- The relationship between skills and rules SHOULD be bidirectional: each skill corresponds to one or more Mangle predicates that gate its injection.

Conceptual Mangle pattern:

```mangle
# Skill injection rule
needs_skill(Intent, "react-best-practices") :-
    selected_tool(_, Tool),
    tool_touches_frontend(Tool),
    project_uses_react.

needs_skill(Intent, "api-contract-validation") :-
    selected_tool(_, Tool),
    tool_modifies_endpoint(Tool).
```

### 4.4 Inline vs. Referenced Skills

Skills may be delivered in two modes:

**Inline** (`"inline": true`): The full skill content is embedded in the intent response. Use for small, focused instructions (< 2000 tokens).

**Referenced** (`"inline": false`): Only the `skill_id` and `description` are included. The client can fetch the full content from the server via a separate resource request (implementation-defined). Use for large skills or skills shared across many macro-tools.

### 4.5 Skill-Rule Binding

MangleCP formalizes the relationship between human-readable skills and machine-queryable rules:

| Concept | Skill (Human) | Rule (Machine) |
|---------|---------------|----------------|
| Purpose | Instructions for the AI/user | Evaluation logic for the server |
| Encoding | Markdown text | Mangle predicates |
| Trigger | Included when rule fires | Fires when facts match |
| Audience | Client/LLM | Server evaluation engine |

Servers SHOULD maintain skills and their corresponding rules in parallel. When a rule is updated, the corresponding skill SHOULD be reviewed for consistency.

---

## 5. Context Injection Semantics

### 5.1 Purpose

Context injection is the mechanism by which a MacroTool carries **everything needed** for effective use. A well-formed macro-tool with context injection should be self-contained -- the client does not need prior conversation history or external knowledge to invoke it.

This formalizes spec-implementer's `get_agent_prompt` pattern: "Agents receiving this prompt need NO additional context from conversation history."

### 5.2 Context Completeness

Servers SHOULD ensure that context injection includes:

1. **Instructions** specific to the current intent and fact state
2. **Skills** proven necessary by rule evaluation
3. **Context resources** identified by `needs_file`-style rules with relevance explanations

Servers SHOULD NOT include:

- Generic instructions that don't change per-intent
- Skills that are always needed (embed those in the server's base behavior instead)
- Context resources with no clear relevance to the macro-tool

### 5.3 Token Budget Awareness

Context injection content contributes significantly to token usage. If the client provides a `max_tokens_budget` constraint (Spec 05), the server MUST attempt to fit the context injection within the budget.

Servers SHOULD apply the following demotion strategies to fit the token budget:

1. **Instructions**: Keep concise (< 200 tokens) regardless of budget.
2. **Skills**: Downgrade skills from `inline: true` to `inline: false` (Referenced) to save tokens.
3. **Context resources**: Recompute the selection priority and omit or truncate low-relevance resources if token pressure is high.

While the server does attempt to fit the budget, it SHOULD provide enough information (priority scores, relevance descriptions) for clients to make further intelligent trimming decisions if client-side limits are stricter.

---

## 6. Example: Full Disclosure MacroTool

```json
{
  "macro_id": "impl-phase2-backend-001",
  "name": "implement_api_endpoints",
  "description": "Implement the REST API endpoints defined in the spec's API contracts section. Creates route handlers, request validation, and database queries for the UserProfile CRUD operations.",
  "disclosure_level": "full",
  "input_schema": {
    "type": "object",
    "properties": {
      "phase_id": {
        "type": "integer",
        "description": "The implementation phase number"
      },
      "dry_run": {
        "type": "boolean",
        "default": false,
        "description": "If true, validate the implementation plan without writing files"
      }
    },
    "required": ["phase_id"]
  },
  "output_schema": {
    "type": "object",
    "properties": {
      "files_created": {
        "type": "array",
        "items": {"type": "string"}
      },
      "tests_generated": {
        "type": "array",
        "items": {"type": "string"}
      },
      "status": {"type": "string"}
    }
  },
  "context_injection": {
    "instructions": "Implement Phase 2 backend endpoints. The UserProfile entity is served by the /api/v1/users endpoint. The existing SDK service UserService already handles authentication. Wire the new endpoints through the existing router pattern in src/routes/.",
    "skills": [
      {"skill_id": "api-contract-validation", "inline": true, "content": "All endpoints MUST validate request bodies against the JSON Schema in 04-api-contracts.md. Use the shared validator middleware at src/middleware/validate.go."}
    ],
    "context_resources": [
      {"uri": "file://docs/specs/feature-001/04-api-contracts.md", "relevance": "Contains the endpoint specifications and request/response schemas", "priority": 100},
      {"uri": "file://src/routes/router.go", "relevance": "Existing router pattern to follow for new endpoint registration", "priority": 90},
      {"uri": "file://src/services/user_service.go", "relevance": "Existing service that handles UserProfile CRUD; new endpoints will delegate to this", "priority": 80},
      {"uri": "file://docs/specs/feature-001/02-cross-system-map.md", "relevance": "Shows how UserProfile flows from API to SDK to frontend components", "priority": 50}
    ]
  },
  "safety": {
    "requires_user_confirmation": true,
    "side_effects": ["filesystem"],
    "reversible": true,
    "idempotent": false
  },
  "validity": {
    "not_before": "2026-02-19T14:30:05Z",
    "expires_at": "2026-02-19T15:30:05Z"
  },
  "metadata": {
    "selection_phase": "skeleton",
    "score": 100,
    "source_rules": ["mandatory_for_intent", "has_backend_step", "needs_file"]
  }
}
```
