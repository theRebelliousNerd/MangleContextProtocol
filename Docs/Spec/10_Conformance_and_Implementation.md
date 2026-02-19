# MangleCP Specification 10: Conformance and Implementation Guide

**Status:** Draft
**Version:** 2026-02-draft
**References:** All previous specifications (01-09)

---

## 1. Purpose

This document defines conformance levels for MangleCP implementations, provides implementation guidance for server and client developers, and addresses IANA-style considerations for future standardization.

---

## 2. Conformance Levels

MangleCP defines three conformance levels. Implementations MUST declare which level they target.

### 2.1 Level 1: Core (Minimum Viable)

A Level 1 conformant implementation MUST support:

| Requirement | Spec Reference |
|-------------|----------------|
| Message envelope with all required fields | Spec 02 Section 2 |
| At least one transport binding (HTTP, WebSocket, or stdio) | Spec 02 Section 4 |
| Manifest delivery on connection/request | Spec 04 Section 2 |
| Manifest with all REQUIRED fields | Spec 04 Section 3 |
| Fact encoding (atemporal facts only) | Spec 03 Section 2 |
| Intent request/response cycle | Spec 05 Section 2-4 |
| At least phases 1, 2, and 6 of the evaluation pipeline (or equivalent behavioral contracts) | Spec 05 Section 3.2 |
| MacroTool objects at `"full"` disclosure level | Spec 06 Section 2 |
| Invoke request/response cycle | Spec 07 Section 2-3 |
| All standard error codes | Spec 09 Section 3 |
| Authentication support (RECOMMENDED for network transports) | Spec 08 Section 3 |
| Message size limits enforcement | Spec 02 Section 3.2 |
| Derivation gas limit enforcement | Spec 05 Section 3.3 |

A Level 1 implementation MAY omit:
- Temporal fact support
- Progressive disclosure (condensed/minimal levels)
- Proof hints
- Rule coverage diagnostics
- Side-effect confirmation workflow
- Suggested intents in invoke responses
- Fact provenance
- External predicates

### 2.2 Level 2: Standard

A Level 2 conformant implementation MUST support everything in Level 1 plus:

| Requirement | Spec Reference |
|-------------|----------------|
| All behavioral contracts (Contracts 1-6) for evaluation | Spec 05 Section 3.2 |
| Progressive disclosure (full/condensed/minimal) | Spec 05 Section 5 |
| Temporal fact encoding (interval and point forms) | Spec 03 Section 4 |
| Evaluation time handling (client-supplied and server-chosen) | Spec 03 Section 4.4 |
| Fact provenance on state_delta responses | Spec 03 Section 5 |
| Fact categories (server/session/derived) on state_delta | Spec 03 Section 6 |
| Transactional state delta (assert + retract) | Spec 07 Section 4 |
| Side-effect classification on all macro-tools | Spec 06 Section 2.3 |
| Confirmation workflow for high-risk tools | Spec 08 Section 4.2 |
| Suggested intents in invoke responses | Spec 07 Section 6 |
| Skill objects (inline and referenced) | Spec 06 Section 4 |
| Context injection on full-disclosure macro-tools | Spec 06 Section 5 |
| Structured error details for fact validation | Spec 09 Section 4.1 |
| Server readiness signaling (status field) | Spec 04 Section 4 |
| Cancellation support on session transports | Spec 02 Section 6.2 |

### 2.3 Level 3: Full

A Level 3 conformant implementation MUST support everything in Level 2 plus:

| Requirement | Spec Reference |
|-------------|----------------|
| Temporal evaluation with past and future reasoning capabilities | Spec 03 Section 4 |
| Temporal validity windows on macro-tools | Spec 06 Section 2.2 |
| Proof hints (derivation traces) | Spec 09 Section 5 |
| Rule coverage diagnostics (optional extension) | Spec 09 Section 6 |
| External predicate declarations in manifest | Spec 04 Section 3.2.8 |
| Predicate schema declarations in facts_profile | Spec 04 Section 3.2.7 |
| Structured fact format (extension) | Spec 03 Section 7 |
| Disclosure upgrade requests | Spec 05 Section 5.3 |
| All three transport bindings (HTTP, WebSocket, stdio) | Spec 02 Section 4 |

---

## 3. Conformance Declaration

### 3.1 In Manifest

Servers MUST declare their conformance level in the manifest's `protocol` object:

```json
{
  "protocol": {
    "manglecp": "2026-02-draft",
    "conformance_level": 2,
    "supported_versions": ["2026-02-draft"]
  }
}
```

### 3.2 In Documentation

Implementations SHOULD declare conformance in their documentation using the following format:

> This implementation conforms to MangleCP Level [1|2|3], version 2026-02-draft.

---

## 4. Server Implementation Guide

### 4.1 Recommended Architecture

Based on patterns validated across four production Mangle systems (BrowserNERD, spec-implementer, spec-generator, codeNERD):

```
MangleCP Server
+--------------------------------------------------+
|  Transport Layer                                  |
|  (HTTP handlers / WebSocket / stdio reader)       |
+--------------------------------------------------+
|  Protocol Layer                                   |
|  (Message parsing, validation, auth, routing)     |
+--------------------------------------------------+
|  Evaluation Layer                                 |
|  (Mangle runtime, fact store, pipeline phases)    |
+--------------------------------------------------+
|  Domain Layer                                     |
|  (.mg files, external predicate bridges, skills)  |
+--------------------------------------------------+
```

### 4.2 The Hollow Kernel Pattern

Servers SHOULD adopt the "Hollow Kernel" pattern validated by codeNERD:

- **All domain logic in `.mg` files**: Tool selection rules, context injection rules, safety policy, intent routing -- all declared as Mangle predicates and rules.
- **Host language as evaluation harness**: Go/Rust/Python code handles transport, serialization, Mangle runtime lifecycle, and external predicate bridges.
- **No domain logic in host code**: If a capability decision requires an `if` statement in Go code, it SHOULD be a Mangle rule instead.

Benefits:
- Rules are auditable and declarative
- Rules can be updated without recompilation
- Rules benefit from Mangle's stratification and termination guarantees

### 4.3 Fact Store Lifecycle

Per-request evaluation MUST use a fresh fact store:

```go
func (s *Server) handleIntentRequest(req IntentRequest) IntentResponse {
    // 1. Create fresh store
    store := factstore.NewSimpleInMemoryStore()  // or NewTemporalStore() for Level 3

    // 2. Load server facts (cached atoms for performance)
    for _, atom := range s.cachedServerAtoms {
        store.Add(atom)
    }

    // 3. Inject client session facts
    for _, fact := range req.Facts {
        atom := compileFact(fact)
        store.Add(atom)
    }

    // 4. Evaluate with gas limit
    _, err := engine.EvalStratifiedProgramWithStats(
        s.programInfo, store,
        engine.DerivedFactsLimit(min(req.Constraints.MaxFactsCreated, s.limits.MaxDerivedFacts)),
    )

    // 5. Query for final_selected tools
    results := queryAll(store, "final_selected(Tool, Score, Source)")

    // 6. Render to MacroTool objects
    return renderResponse(results, req.Options)
}
```

### 4.4 Rule Organization

Servers SHOULD organize their `.mg` files in trust-stratified layers:

```
rules/
  schema/           # Level 1 trust (immutable)
    predicates.mg   # Predicate declarations
    types.mg        # Type schemas
  policy/           # Level 2 trust (server-configurable)
    safety.mg       # Permission and safety rules
    constitution.mg # Constitutional constraints
  domain/           # Level 3 trust (domain rules)
    intents.mg      # Intent routing rules
    tools.mg        # Tool selection rules
    context.mg      # Context injection rules (needs_file patterns)
    skills.mg       # Skill injection rules
  pipeline/         # Pipeline phases
    01_skeleton.mg
    02_exclusion.mg
    03_flesh.mg
    04_conflict.mg
    05_dependency.mg
    06_output.mg
```

### 4.5 RECOMMENDED: The 6-Phase Evaluation Pipeline

The following 6-phase pipeline is a RECOMMENDED implementation strategy for satisfying the behavioral contracts in Spec 05 Section 3.2. It is derived from codeNERD's JIT Prompt Compiler and has been validated across four production Mangle systems.

Servers are NOT required to implement these exact phases. Any evaluation strategy that produces output satisfying Contracts 1-6 (Spec 05 Section 3.2) is conformant. However, the 6-phase model is a proven pattern that naturally addresses all contracts.

```
Phase 1: Skeleton     -- Mandatory capabilities for this intent type
Phase 2: Exclusion    -- Capabilities blocked by context or policy
Phase 3: Flesh        -- Additional relevant capabilities (scoring)
Phase 4: Conflict     -- Resolve competing capabilities
Phase 5: Dependency   -- Ensure prerequisite capabilities are included
Phase 6: Final Output -- Validated, conflict-free capability set
```

#### Phase 1: Skeleton (Mandatory Selection)

The skeleton phase identifies **mandatory** capabilities that MUST be included for the given intent type, regardless of other context.

Conceptual Mangle pattern:
```mangle
selected(Tool, 100, "skeleton") :-
    intent_type(Intent, Type),
    mandatory_for_intent(Type, Tool).

selected(Tool, 90, "skeleton") :-
    tool_constraint(Tool, Key, Value),
    satisfied_constraint(Key, Value),
    !blocked_by_context(Tool).
```

Skeleton tools represent the **minimum viable capability set** for the intent. A `diagnose_error` intent always needs diagnostic tools. A `find_data_surface` intent always needs search capabilities.

If no skeleton tools match the intent, the server SHOULD return an empty `macro_tools` array rather than an error (the intent may be valid but outside the server's current rule coverage).

**Satisfies:** Contract 1 (Intent Relevance)

#### Phase 2: Exclusion (Firewall)

The exclusion phase removes capabilities that are **prohibited** in the current context. Exclusion overrides any positive selection from any phase.

Conceptual Mangle pattern:
```mangle
excluded(Tool) :- prohibited(Tool, Reason).
excluded(Tool) :- incompatible(Tool, OtherTool), selected(OtherTool, _, _).
excluded(Tool) :- requires_capability(Tool, Cap), !server_has_capability(Cap).
```

Common exclusion reasons:
- Policy prohibition (safety rules block a tool in this context)
- Incompatibility (two tools conflict; the lower-scored one is excluded)
- Missing prerequisite (the tool requires a server capability that is unavailable)
- Client restriction (the client's intent params exclude certain tool categories)

**Satisfies:** Contract 2 (Prohibition Enforcement)

#### Phase 3: Flesh (Relevance Scoring)

The flesh phase identifies **additional** capabilities beyond the skeleton that are relevant to the current intent and facts. This is where hybrid logic+vector scoring (if the server supports it) produces relevance rankings.

Conceptual Mangle pattern:
```mangle
selected(Tool, Score, "relevance") :-
    tool_relevant_to_intent(Tool, Intent, BaseScore),
    !excluded(Tool),
    Score = BaseScore.

selected(Tool, Score, "vector") :-
    vector_similarity(Tool, Intent, VScore),
    VScore > 30,
    !excluded(Tool),
    Score = VScore.
```

Servers MAY use pure Mangle rule-based scoring, hybrid logic+vector scoring (as in codeNERD's 70/30 model), or any other relevance mechanism.

**Satisfies:** Contract 1 (Intent Relevance), provides scoring data for progressive disclosure

#### Phase 4: Conflict Resolution

When multiple capabilities serve the same purpose, the conflict resolution phase selects the best one.

Conceptual Mangle pattern:
```mangle
invalid(Tool1) :-
    conflicts_with(Tool1, Tool2),
    selected(Tool1, S1, _),
    selected(Tool2, S2, _),
    S1 < S2.

# Deterministic tie-breaking on name
invalid(Tool1) :-
    conflicts_with(Tool1, Tool2),
    selected(Tool1, S, _),
    selected(Tool2, S, _),
    Tool1 > Tool2.
```

**Satisfies:** Contract 3 (Conflict Freedom), Contract 6 (Determinism)

#### Phase 5: Dependency Resolution

Capabilities may declare dependencies on other capabilities. If a selected capability requires a prerequisite that was not selected, the prerequisite MUST be added or the dependent capability MUST be removed.

Conceptual Mangle pattern:
```mangle
missing_dependency(Tool, Dep) :-
    tool_requires(Tool, Dep),
    selected(Tool, _, _),
    !selected(Dep, _, _).

# Option A: Cascade add the dependency
selected(Dep, 80, "dependency") :-
    missing_dependency(_, Dep),
    !excluded(Dep).

# Option B: Cascade invalidate the dependent
invalid(Tool) :- missing_dependency(Tool, Dep), excluded(Dep).
```

Servers SHOULD prefer adding missing dependencies over removing dependents, unless the dependency is excluded.

**Satisfies:** Contract 4 (Dependency Completeness)

#### Phase 6: Final Output

The final phase produces the validated, conflict-free set of capabilities:

Conceptual Mangle pattern:
```mangle
final_selected(Tool, Priority, Source) :-
    selected(Tool, Priority, Source),
    !excluded(Tool),
    !invalid(Tool).
```

The output of Phase 6 is the set of `final_selected` facts, which are rendered into MacroTool objects for the response.

**Satisfies:** All contracts (final validation gate)

### 4.6 External Predicate Implementation

For servers that bridge Mangle evaluation to runtime data:

```go
// Register an external predicate
externalPreds := []engine.ExternalPredicate{
    {
        Name:     "recall_similar",
        Arity:    3,  // query(bound), k(bound), result(free)
        Callback: func(args []ast.Constant) ([][]ast.Constant, error) {
            query := args[0].StringValue()
            k := args[1].NumberValue()
            results := embeddingStore.Search(query, int(k))
            return formatResults(results), nil
        },
    },
}
```

External predicates MUST be declared in the manifest's `capabilities.external_predicates` array.

### 4.7 Performance Recommendations

| Concern | Recommendation | Validated By |
|---------|---------------|--------------|
| EDB/server fact parsing | Pre-parse server facts to AST atoms once; reuse across evaluations | codeNERD `cachedAtoms` |
| Stratification | Cache stratification results; rebuild only when rules change | codeNERD `rebuildProgram()` |
| Fact deduplication | Use a hash set to reject duplicate facts before parsing | codeNERD `factIndex` |
| Bulk fact loading | Suspend evaluation during initial fact load; single eval at end | spec-implementer `SuspendEval/ResumeEval` |
| Adaptive sampling | Under load, sample low-value facts while preserving high-value ones | BrowserNERD 5-tier sampling |

---

## 5. Client Implementation Guide

### 5.1 Minimal Client Flow

```
1. Connect to server (transport-dependent)
2. Receive manifest
3. Parse manifest to understand:
   - Available intent types
   - Required/optional facts per intent
   - Server limits
   - Authentication requirements
4. Authenticate (if required)
5. Construct intent request:
   - Choose intent name from manifest
   - Build fact array using manifest's facts_profile for validation
   - Set evaluation constraints
6. Send intent_request, receive intent_response
7. Present macro_tools to user/LLM
8. User/LLM selects a macro-tool
9. Validate args against input_schema
10. Send invoke_request, receive invoke_response
11. Process result, apply state_delta
12. Follow suggested_intents for next step (optional)
```

### 5.2 Client-Side Fact Validation

Before sending facts, clients SHOULD validate against the manifest's `facts_profile.predicates`:

```python
def validate_fact(fact, predicates):
    schema = find_predicate(predicates, fact.pred)
    if not schema:
        warn(f"Unknown predicate: {fact.pred}")
        return

    if schema.direction == "output":
        error(f"Predicate {fact.pred} is output-only; don't send it")
        return

    if len(fact.args) != schema.arity:
        error(f"Arity mismatch: {fact.pred} expects {schema.arity}, got {len(fact.args)}")
        return

    for i, (arg, expected_type) in enumerate(zip(fact.args, schema.arg_types)):
        if not type_matches(arg, expected_type):
            error(f"Type mismatch at arg {i}: expected {expected_type}")
```

### 5.3 Multi-Server Clients

Clients connecting to multiple MangleCP servers (like codeNERD connects to multiple MCP servers) face a tool selection problem across servers. Recommendations:

- **Per-server evaluation**: Send intent requests to relevant servers based on their `domain` field.
- **Cross-server deduplication**: If multiple servers return similar macro-tools, use `metadata.score` to select the best.
- **Domain routing**: Map intents to servers based on `domain.categories` in manifests.

### 5.4 State Management Across Requests

Since MangleCP is stateless, clients maintain their own working state:

```python
class MangleCPClientState:
    def __init__(self):
        self.facts = []  # Current working facts

    def apply_state_delta(self, delta):
        # 1. Apply retractions
        for retract in delta.get("retract", []):
            self.facts = [f for f in self.facts if not matches_pattern(f, retract)]

        # 2. Apply assertions
        for assertion in delta.get("assert", []):
            if assertion not in self.facts:
                self.facts.append(assertion)

    def build_intent_request(self, intent_name, params, extra_facts=None):
        all_facts = self.facts + (extra_facts or [])
        return {
            "intent": {"name": intent_name, "params": params},
            "facts": all_facts
        }
```

---

## 6. Testing Conformance

### 6.1 Conformance Test Categories

| Category | What It Tests |
|----------|--------------|
| **Manifest** | Manifest delivery timing, required fields, valid JSON |
| **Fact encoding** | All argument types, temporal annotations, provenance |
| **Intent evaluation** | Pipeline phases, constraint enforcement, determinism |
| **Macro-tools** | Disclosure levels, schema validity, safety metadata |
| **Invocation** | Arg validation, state delta, observability, suggested intents |
| **Errors** | All standard error codes, structured details, recoverability |
| **Security** | Auth enforcement, reserved predicate rejection, side-effect gating |
| **Transport** | Each transport binding's specific requirements |

### 6.2 Determinism Test

Servers MUST pass the determinism test:

1. Send identical intent requests (same intent, facts, eval_time, constraints) 100 times.
2. All 100 responses MUST return the same set of `macro_id` values.
3. All 100 responses MUST assign the same `disclosure_level` to each macro-tool.
4. All 100 responses MUST compute the same `metadata.score` for each macro-tool.

Variation in `eval_duration_ms` is expected and acceptable.

### 6.3 Gas Limit Test

1. Send an intent request with `max_facts_created: 10`.
2. If the server's rules would normally derive more than 10 facts, the server MUST return error code `"derivation_limit_exceeded"`.
3. The error MUST include `details.budget` with the limit and consumed values.

---

## 7. Versioning Policy

### 7.1 Version String Format

MangleCP version strings follow the format: `YYYY-MM-<qualifier>`

- `YYYY`: Year
- `MM`: Month
- `<qualifier>`: `draft`, `rc1`, `rc2`, `final`

Examples: `2026-02-draft`, `2026-06-rc1`, `2026-09-final`

### 7.2 Backward Compatibility

- **Draft** versions: No backward compatibility guarantees. Breaking changes allowed.
- **Release Candidate** versions: Backward compatible within the same RC series.
- **Final** versions: Backward compatible within the same major year. Servers supporting `2026-09-final` SHOULD also support `2026-06-final`.

### 7.3 Deprecation

When a version is deprecated:
- Servers MUST continue supporting it for at least 6 months after deprecation announcement.
- Servers SHOULD return a warning in the manifest's `extensions` when a deprecated version is in use.
- Clients SHOULD upgrade to the latest supported version when notified of deprecation.

---

## 8. IANA-Style Considerations

If MangleCP progresses to an Internet-Draft or standards track, the following registrations would be needed:

### 8.1 Well-Known URI Registration

| Field | Value |
|-------|-------|
| URI suffix | `manglecp/manifest.json` |
| Change controller | MangleCP Working Group |
| Reference | This specification, Spec 02 Section 5.1 |

### 8.2 Media Type Registration

| Field | Value |
|-------|-------|
| Type name | `application` |
| Subtype name | `manglecp+json` |
| Required parameters | None |
| Optional parameters | `version` (protocol version string) |
| Encoding | UTF-8 |
| Interoperability | All MangleCP messages |
| Reference | This specification, Spec 02 |

### 8.3 Error Code Registry

A registry for `error.code` values as defined in Spec 09 Section 3.1. Custom codes MUST use the `x-` prefix.

### 8.4 Side Effect Category Registry

A registry for `safety.side_effects` values as defined in Spec 06 Section 2.3. Custom categories MUST use the `x-` prefix.

### 8.5 Domain Category Registry

A registry for `domain.categories` values as defined in Spec 04 Section 5. Custom categories MUST use the `x-` prefix.

---

## 9. Future Extensions (Informative)

The following features are anticipated for future MangleCP versions:

| Feature | Version Target | Description |
|---------|---------------|-------------|
| Rule submission | v2 | Clients submit Mangle rules with sandbox validation |
| Pub/sub subscriptions | v2 | Server-initiated notifications when derived facts change |
| Streaming invocation | v2 | SSE/WebSocket streaming for long-running macro-tool execution |
| Multi-server federation | v2 | Protocol for cross-server intent routing |
| Schema sharding | v2 | Lazy loading of rule modules based on intent type |
| Fact compression | v2 | Binary encoding for high-volume fact transfer |
| Capability negotiation | v2 | Bidirectional capability exchange (client declares what it can do) |

These features are NOT part of the current draft and MUST NOT be assumed by implementations.
