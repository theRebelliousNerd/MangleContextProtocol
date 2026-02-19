# MangleCP Specification 01: Overview and Architecture

**Status:** Draft
**Version:** 2026-02-draft
**Supersedes:** `Docs/Research/gpt5.2MangleContextProtocol.md` (whitepaper)

---

## 1. Purpose

This document is the normative overview of the Mangle Context Protocol (MangleCP). It defines the protocol's scope, architectural model, design constraints, and the relationships between the remaining specification documents.

MangleCP is a wire protocol that replaces the "static list of atomic tools" paradigm with **server-computed, context-validated macro-tools** powered by Mangle's temporal Datalog reasoning. A MangleCP server embeds the Mangle runtime, evaluates DatalogMTL-style rules against client-supplied intent and time-scoped facts, and returns the smallest, most relevant tool payloads over the wire.

---

## 2. Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document series are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) (BCP 14).

All protocol payloads are JSON as defined by [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) unless otherwise specified.

Time instants use [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) timestamps by default.

---

## 3. Scope and Design Boundaries

### 3.1 What MangleCP Is

MangleCP is a stateless wire protocol. It specifies:

- **JSON message shapes** for capability negotiation, intent evaluation, tool invocation, and error reporting
- **Fact encoding** for both atemporal and temporal (DatalogMTL) assertions
- **Macro-tool semantics** for server-synthesized, intent-scoped tool payloads
- **Evaluation constraints** for termination safety and resource budgeting
- **Security posture** for authentication, authorization, and side-effect gating
- **Conformance requirements** for interoperable server and client implementations

### 3.2 What MangleCP Is Not

MangleCP does **not** define:

| Excluded Concern | Rationale |
|------------------|-----------|
| Agent loop or planner | Autonomy and reasoning belong to client implementations |
| Memory model | State persistence is an implementation concern, not a wire protocol concern |
| Natural language processing | Intent is a structured payload, never a prompt |
| Orchestration | Multi-step workflows, subagent spawning, and pipeline sequencing are client concerns |
| Mangle runtime specification | MangleCP depends on Mangle but does not redefine its semantics; see [google/mangle](https://github.com/google/mangle) |

If a feature requires crossing these boundaries, it belongs in a client or server implementation, not in the protocol specification.

---

## 4. Architectural Model

### 4.1 Three-Layer Protocol ("Matryoshka Model")

MangleCP defines three conceptual layers, each nested within the next:

```
+----------------------------------------------------------+
|  Layer 1: Manifest (Capability Advertisement)            |
|  Server sends a lightweight "business card" on connect   |
+----------------------------------------------------------+
|  Layer 2: Intent Evaluation                              |
|  Client sends (Intent, Facts, Constraints)               |
|  Server evaluates .mg programs -> MacroTool set          |
+----------------------------------------------------------+
|  Layer 3: Macro-Tool Invocation                          |
|  Client invokes macro-tool with high-level args          |
|  Server executes atomic chain -> Result + State Delta    |
+----------------------------------------------------------+
```

**Layer 1 -- Manifest:** On connection, the server broadcasts a manifest describing its domain, capabilities, fact expectations, evaluation limits, and authentication requirements. The manifest MUST NOT enumerate a full set of atomic tools. It provides enough information for the client to form intent evaluation requests. *(Spec 04)*

**Layer 2 -- Intent Evaluation:** The client sends an intent request containing a structured intent label, client-asserted facts, optional temporal context, and evaluation constraints. The server loads its `.mg` programs, injects the client facts, runs a **6-phase evaluation pipeline** (skeleton, exclusion, flesh, conflict resolution, dependency resolution, final output), and returns a set of validated MacroTool objects with progressive disclosure levels. *(Spec 05)*

**Layer 3 -- Macro-Tool Invocation:** The client selects and invokes a macro-tool using high-level arguments validated against the tool's input schema. The server executes the underlying atomic action chain and returns results, a transactional state delta (facts to add/remove), observability events, and suggested follow-on intents. *(Spec 07)*

### 4.2 Server Architecture: The "Hollow Kernel" Model

A MangleCP server is an **evaluation harness** around declarative Mangle programs. All domain reasoning, tool selection logic, context injection rules, and safety policy live in `.mg` files. The server's host language code (Go, or any language with a Mangle binding) provides:

- Transport handling (WebSocket, HTTP, stdio)
- Mangle runtime embedding and evaluation lifecycle
- External predicate bridges to runtime data sources (databases, APIs, embeddings)
- Wire format serialization/deserialization
- Protocol-level security enforcement (the "unforgeable layer")

This pattern is validated by codeNERD's 6,900-line Go harness around 274 `.mg` files, where every action is proposed as a Mangle fact and gated by rule evaluation before execution.

### 4.3 Evaluation Model: Fresh Store Per Intent

Each intent evaluation request MUST be processed against a **fresh fact store** populated from:

1. **Server base facts** -- the server's persistent facts (schema, policy, capability declarations)
2. **Client-asserted facts** -- facts supplied in the intent request (session-scoped, highest stratum)
3. **External predicate results** -- facts materialized on-demand from FFI bridges

Derived facts exist only for the duration of the evaluation. They MUST NOT accumulate across evaluations. This ensures deterministic, stateless reasoning per request.

### 4.4 Trust Hierarchy for Rule Sources

MangleCP servers MUST evaluate rules from sources with a stratified trust ordering:

| Trust Level | Source | Mutability | Examples |
|-------------|--------|------------|----------|
| **1 (Highest)** | Schema | Immutable at runtime | Predicate declarations, type schemas |
| **2** | Policy | Server-configurable, not client-modifiable | Safety rules, permission predicates, constitutional checks |
| **3** | Domain Rules | Server-managed, may update between requests | Capability rules, intent routing, context injection |
| **4 (Lowest)** | Client-Pushed | Session-scoped, from intent request | Client-asserted facts and (future v2) client-submitted rules |

Higher-trust rules MUST NOT be overridable by lower-trust sources. Mangle's stratification algorithm naturally enforces this when rule sources are concatenated in trust order.

### 4.5 Fact Categories

MangleCP distinguishes three fact categories that determine lifecycle and wire format handling:

| Category | Wire Name | Lifecycle | Origin | Example Predicates |
|----------|-----------|-----------|--------|--------------------|
| **Persistent** | `"server"` | Survive across requests; server-managed | Server | `tool_registered`, `has_capability`, `server_config` |
| **Session-Scoped** | `"session"` | Live for the duration of one intent evaluation | Client | `user_intent`, `active_context`, `current_file` |
| **Derived** | `"derived"` | Computed by rule evaluation; never persisted | Rules | `permitted`, `selected_tool`, `context_priority` |

Servers MUST tag facts in `state_delta` responses with their category. Clients MUST NOT assume derived facts persist across requests.

---

## 5. Relationship to Prior Art

### 5.1 Relationship to MCP

MangleCP departs from MCP in three fundamental ways:

| Dimension | MCP | MangleCP |
|-----------|-----|----------|
| Tool discovery | Client enumerates all tools at connect time | Server synthesizes relevant tools per intent |
| Tool granularity | Atomic tools with static schemas | Macro-tools with dynamically generated schemas |
| Authorization | Optional | Required for network transports |

MangleCP is not a fork of MCP. It is a new protocol that addresses the tool explosion and token inefficiency problems that arise when static tool lists scale beyond a few dozen entries.

### 5.2 Relationship to Mangle

MangleCP depends on Mangle's evaluation semantics but does not redefine them. The protocol specifies:

- How Mangle facts are encoded in JSON for wire transport *(Spec 03)*
- How evaluation constraints map to Mangle's termination controls *(Spec 05)*
- How temporal annotations map to DatalogMTL's `@[start,end]` syntax *(Spec 03)*

MangleCP servers MAY use any conforming Mangle implementation. The protocol does not mandate a specific runtime version, but servers MUST report their temporal reasoning capabilities in the manifest.

### 5.3 Relationship to A2A

MangleCP's manifest is conceptually aligned with A2A's AgentCard -- both serve as lightweight capability advertisements. MangleCP does not adopt A2A's task lifecycle or multi-agent communication semantics.

---

## 6. Specification Document Index

| Document | Title | Scope |
|----------|-------|-------|
| **Spec 01** (this document) | Overview and Architecture | Protocol scope, layers, server model, trust hierarchy |
| **Spec 02** | Wire Format and Transport Bindings | Message envelope, transport bindings, discovery |
| **Spec 03** | Data Model | Fact encoding, temporal annotations, provenance, type system |
| **Spec 04** | Manifest and Capability Negotiation | Manifest payload, capability advertisement, boot/ready signaling |
| **Spec 05** | Intent Evaluation | 6-phase pipeline, evaluation semantics, constraints, progressive disclosure |
| **Spec 06** | Macro-Tools and Skills | MacroTool object, Skill object, context injection, disclosure levels |
| **Spec 07** | Invocation Protocol and State Management | Invoke request/response, transactional state delta, suggested intents |
| **Spec 08** | Security Model | Dual-layer safety, authentication, authorization, side-effect gating |
| **Spec 09** | Error Handling and Diagnostics | Error codes, validation errors, proof hints, rule coverage reporting |
| **Spec 10** | Conformance and Implementation Guide | Conformance levels, implementation guidance, IANA considerations |

---

## 7. Design Rationale Summary

The following table traces key design decisions to their source evidence in the research corpus:

| Decision | Evidence Source | Document |
|----------|----------------|----------|
| Intent-scoped evaluation over static tool lists | `map_feature_surface` hand-coding, `derived_surface` 5-hop rule | SpecGenerator, BrowserNERD |
| 6-phase evaluation pipeline | `jit_compiler.mg` 6-phase atom selection | codeNERD Part 2 |
| Fresh-store per evaluation | `kernel_eval.go` fresh store creation per cycle | codeNERD Part 1 |
| Stratified trust ordering | `rebuildProgram()` schema > policy > learned concatenation | codeNERD Part 1 |
| Three-tier fact categories | `fact_categories.go` persistent/ephemeral/derived | codeNERD Part 1 |
| Transactional state delta | `KernelTransaction` atomic retract+assert | codeNERD Part 1 |
| Dual-layer security | Go constitutional checks + Mangle policy rules | codeNERD Part 1 |
| Progressive disclosure levels | `renderer.go` full/condensed/minimal tiers | codeNERD Part 2 |
| Gas limits as protocol constraint | `DerivedFactsLimit` engine parameter | codeNERD Part 1 |
| Fact provenance | `PushFactWithSource` provenance tracking | SpecGenerator |
| DatalogMTL temporal as optional | Zero temporal store usage across 4 production systems | All case studies |
| Boot/ready signaling | Boot guard preventing actions during init | codeNERD Part 1 |
