# Mangle Context Protocol (MangleCP)

## Abstract

Mangle Context Protocol (MangleCP) is a server-to-client protocol for tool discovery and capability serving that replaces the "static list of atomic tools" model with **server-computed, context-validated macro-tools**. MangleCP's distinguishing feature is that the server embeds the Mangle runtime (an embeddable Go library) and uses DatalogMTL-style temporal reasoning and aggregation to synthesize the smallest, most relevant JSON tool payload(s) for a given client "Intent" and time-scoped state facts.

MangleCP is strictly a **stateless wire protocol**: it defines JSON message shapes, negotiation, and payload semantics. It does **not** define an agent loop, memory model, planner, or "autonomy runtime." Its only job is to specify how a server computes and transmits mathematically constrained (i.e., rule-validated) tool capability payloads over the wire. *(This boundary is a design constraint for this document.)*

---

## Terminology and Conventions

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as described in Internet Engineering Task Force documents [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) (BCP 14).

- Unless otherwise specified, all protocol payloads are JSON as defined by [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259).
- Time instants in MangleCP use [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) timestamps by default (an interoperable profile of ISO 8601 for Internet protocols).
- When MangleCP uses a "well-known" discovery path for HTTP transports, it aligns with [RFC 8615](https://www.rfc-editor.org/rfc/rfc8615) and the IANA Well-Known URIs registry model.

### Definitions

| Term | Definition |
|------|------------|
| **Client** | The host framework that requests capabilities from a server and invokes returned tools (often—but not necessarily—an LLM host). |
| **Server** | The remote service that computes and returns macro-tool schemas and executes macro-tools. |
| **Manifest** | The initial, lightweight capability advertisement broadcast on connect (conceptually similar to an A2A "AgentCard"). |
| **Intent** | A client-supplied label or structured object representing the client's desired outcome. MangleCP does not define a natural-language format; it transports an Intent payload. |
| **Fact** | A single assertion about state, optionally time-bounded (temporal). |
| **Temporal fact** | A fact with a validity interval (e.g., `@[start,end]` in Mangle), or a point interval (e.g., `@[T]`). |
| **Macro-tool** | A dynamically generated tool abstraction that bundles a validated chain of atomic actions behind a single high-level invocation schema (analogous to `execute-plan` in the browserNerd paradigm). |
| **Skill** | A reusable instruction/resource bundle that can be injected into a tool payload when a server-side predicate proves it to be contextually required (conceptually aligned with Agent Skills). |

---

## Grounded Research Findings

### Mangle's Implementation Shape and `.mg` Program Model

The [google/mangle](https://github.com/google/mangle) repository describes Mangle as a Datalog extension with practical additions such as aggregation, function calls, and optional type checking, and explicitly states that the repository provides an implementation *"as a Go library that can be easily embedded into applications."*

Mangle's repo includes a generated parser pipeline (ANTLR grammar and regeneration steps), indicating a first-class language frontend suitable for embedding into other systems that load and analyze program files.

The repo's recent DatalogMTL work also includes example files with the `.mg` extension (e.g., `temporal_graph_intervals.mg`, `temporal_graph_points.mg`, `temporal_sequence.mg`) in its changed-file list for the DatalogMTL integration commit—supporting the interpretation that Mangle programs are stored and exchanged as `.mg` sources in practice.

### Temporal Knowledge Graphs and DatalogMTL Alignment

Mangle's repository README includes a "Temporal Knowledge Graphs" example that marks a predicate as temporal and uses the `@[T]` annotation in recursive reachability rules to represent time-scoped reachability.

The Mangle temporal documentation specifies that any fact can be annotated with a time interval using `@[start,end]`, that `_` denotes an unbounded interval endpoint, and that `now` denotes the evaluation time. It also defines a point-interval shorthand `@[T]`.

Mangle defines **four temporal operators**—two past operators (`<-`, `[-`) and two future operators (`<+`, `[+`)—with bounded windows expressed using duration bounds (e.g., days `d`, hours `h`) relative to an evaluation time `now`. This is directly in the shape of metric temporal operators over Datalog-like rules, matching the common description of "DatalogMTL" as Datalog extended with metric temporal logic operators.

> The Feb 9, 2026 commit *"Implement full DatalogMTL support and fix temporal reasoning bugs"* states that temporal reasoning was integrated into the semi-naive evaluation engine, including correct derivation of temporal facts into a `TemporalStore`, preservation of temporal annotations on initial facts, improved parsing and interval handling, and integration of temporal recursion checks into static analysis. This indicates that temporal semantics are not merely surface syntax: they are **integrated into the core evaluation pipeline**.

### Temporal Stores, Evaluation Time, and Termination Controls

Mangle's temporal documentation provides a programmatic integration pattern: the server can create a temporal store (`factstore.NewTemporalStore()`), add facts with intervals, and evaluate programs with explicit temporal support and a chosen evaluation time.

The same documentation describes safeguards against non-termination and interval explosion, including:

- **Interval coalescing**
- A **default interval limit per atom** (configurable)
- **Created-fact limits** in the engine

It also provides examples of "safe" vs "dangerous" temporal recursion patterns and explicitly warns that some temporal reasoning patterns may not terminate without safeguards.

### Aggregation and Path-Finding Expressed in Core Datalog Constructs

Mangle supports aggregation through a pipeline (`|> do ...`) that can group results (`fn:group_by(...)`) and compute aggregations such as `fn:count()` and `fn:sum(...)`.

Mangle (as a Datalog family system) supports recursion and fixed-point computation. The docs' canonical `reachable` example illustrates transitive closure over a `knows` relation—i.e., graph reachability/path-finding with recursive rules.

Mangle's README also demonstrates constructing structured outputs (e.g., lists of route codes for a one- or two-leg trip) and computing derived values (e.g., total price via addition), showing that "path-ish" results can be represented as compositional structured values, not only boolean reachability.

### The browserNerd Execution Paradigm Relevant to Macro-Tools

The browserNerd repository README describes an approach to **token-efficient browser automation** where the system returns structured JSON "browser state," and emphasizes batching and progressive disclosure. In particular, it states that `execute-plan` runs multiple actions in one call, reducing round trips and token usage relative to many atomic calls.

The same README explicitly describes **"Progressive Disclosure" tools** (e.g., `browser-observe`, `browser-act`, `browser-reason`) that allow the client to request only the detail level it needs *"right detail at the right time,"* while the underlying system manages the operational complexity.

browserNerd also describes its internal use of Mangle for declarative reasoning over extracted browser/React facts (including time-windowed fact analysis), reinforcing the architecture pattern: **extract facts → evaluate rules → expose higher-level, token-efficient outputs**.

### Contrast Point: MCP's List-First Tool Model and Optional Auth

The MCP specification describes a tool model in which servers "expose tools" that can be invoked by models, with each tool identified by name and described with a schema.

- MCP's base protocol is JSON-RPC and includes lifecycle management (initialization and capability negotiation).
- MCP's authorization spec explicitly states that authorization is **OPTIONAL** for implementations and describes OAuth 2.1-based requirements for protected resource requests when auth is implemented.
- Public security commentary has highlighted that optional authorization combined with HTTP-exposed MCP endpoints can produce practical exposure risks, motivating a **"secure-by-default"** posture for any successor protocol.

---

## Design Goals and Constraints Derived from the Research

MangleCP's architecture in this whitepaper is constrained by two grounded observations:

1. **Mangle's strengths** lie in representing state as facts, deriving new facts via rules, performing recursion for path-finding, applying aggregation for summarization, and evaluating temporal logic relative to a chosen evaluation time—backed by a `TemporalStore`, coalescing, and explicit termination controls.

2. **browserNerd** operationalizes token efficiency by collapsing a multi-step action sequence into a single higher-level call (`execute-plan`) and uses progressive disclosure to avoid flooding the client with raw state.

From these, the **core MangleCP goals** are:

- **Manifest-first discovery** — A lightweight "business card" broadcast at connect time, aligning with the A2A AgentCard metaphor ("like a business card") rather than a full tool catalog.
- **Intent-scoped tool synthesis** — The server computes macro-tools in response to the client's intent+state, rather than exposing a single static tool list. *(This is the primary departure from MCP's list-first tooling posture.)*
- **Temporal correctness** — The protocol must carry enough time/interval information for the server to evaluate DatalogMTL-style rules deterministically relative to an explicit evaluation time.
- **Safety against non-termination** — The protocol must support server-declared evaluation/derivation limits and error surfaces for "dangerous temporal patterns," reflecting Mangle's documented termination risks and mitigation knobs.
- **Secure-by-default** — Because "optional auth" combined with HTTP exposure is a demonstrated operational hazard, MangleCP should normatively require authentication for network transports unless the server is explicitly in an "open demo" mode.

---

## Protocol Architecture

### Overview

MangleCP defines **three conceptual layers**:

```
┌─────────────────────────────────────────────────┐
│  Outer Doll: Handshake Manifest                 │
│  Server sends a lightweight "business card"     │
├─────────────────────────────────────────────────┤
│  Middle Doll: Intent Evaluation                 │
│  Client sends (Intent, Facts, Temporal context) │
│  Server evaluates .mg programs → MacroTools     │
├─────────────────────────────────────────────────┤
│  Inner Doll: Macro-Tool Invocation              │
│  Client invokes macro-tool with high-level args │
│  Server executes atomic chain → results + delta │
└─────────────────────────────────────────────────┘
```

**Outer Doll — Handshake Manifest:**
On connection establishment, the server sends a single manifest describing its domain and how to query it. This mirrors the A2A AgentCard idea: metadata that allows others to discover "what you can do."

**Middle Doll — Intent Evaluation:**
The client sends `(Intent, Facts, Temporal context)` to a single evaluation endpoint. The server loads its `.mg` programs and evaluates them using Mangle's evaluation pipeline, including temporal reasoning (DatalogMTL-style operators), recursion, and aggregation, producing a set of validated **MacroTool** proposals.

**Inner Doll — Macro-Tool Invocation:**
The client invokes a chosen macro-tool using only a few high-level arguments. The server executes the underlying atomic chain "natively," returning results and (optionally) a delta of derived facts that the client can feed into the next Intent call. This mirrors the browserNerd *"LLM doesn't micromanage atomic actions; it invokes a batch plan"* paradigm.

### What MangleCP Does *Not* Do

MangleCP does **not** define:

- Step-by-step planning heuristics
- A reasoning loop
- Client or server memory semantics
- Agent autonomy

Those are explicitly out of scope; MangleCP is **only** the wire protocol and JSON payload semantics.

---

## Wire Protocol Specification

### Transport and Discovery

MangleCP is transport-agnostic but defines normative bindings for two common classes:

**Session transports** (e.g., WebSocket, stdio):

- The server **MUST** send a `manifest` message as the first application message after transport establishment.

**HTTP request/response transports:**

- The server **MUST** expose its manifest at a well-known path (recommended: `/.well-known/manglecp/manifest.json`) using the RFC 8615 "well-known URI" pattern.
- All payloads **MUST** be valid JSON ([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)).

### Message Envelope

MangleCP defines a minimal envelope compatible with JSON-RPC-style correlation while allowing server-initiated messages:

```json
{
  "type": "<manifest | intent_request | intent_response | invoke_request | invoke_response | error>",
  "id": "<optional string — MUST be present for request/response correlation>",
  "manglecp": "<protocol version string, e.g. '2026-02-draft'>",
  "payload": { }
}
```

> This design is intentionally simpler than MCP's full lifecycle state model while preserving a request/response correlation primitive. MCP's base protocol is JSON-RPC and includes lifecycle management; MangleCP intentionally does not replicate lifecycle state beyond a manifest handshake.

### Manifest Payload

The manifest is intentionally "ultra-lightweight" and **MUST NOT** enumerate a full set of atomic tools. It **SHOULD** provide enough information for the client to form Intent evaluation requests.

#### Manifest Fields

```json
{
  "server_name": "<string>",
  "server_version": "<string>",
  "protocol": { "manglecp": "<version>" },
  "domain": {
    "id": "<string>",
    "description": "<string>"
  },
  "endpoints": {
    "intent_eval": "<URI (absolute or relative)>",
    "macro_invoke": "<URI (absolute or relative)>"
  },
  "facts_profile": "<describes how the server expects facts to be encoded>",
  "auth": "<authentication requirements (see Security)>"
}
```

> **Rationale:** This parallels the A2A AgentCard's role as a "business card" metadata document to describe agent capabilities (without embedding all behaviors).

---

### Fact Encoding

MangleCP needs a JSON representation that is faithful to Mangle's fact and temporal semantics.

#### Fact Object

```json
{
  "pred": "<string — predicate symbol>",
  "args": ["<JSON values: strings / numbers / booleans / objects / arrays>"],
  "t": "<optional temporal annotation>"
}
```

The temporal annotation `t` supports two forms:

| Form | JSON Shape |
|------|------------|
| **Interval** | `{ "start": <Time>, "end": <Time \| "_"> }` |
| **Point** | `{ "at": <Time> }` |

#### Time

- By default, `<Time>` **MUST** be an RFC 3339 timestamp string.
- Servers **MAY** additionally support `{ "nanos": <int64> }` as an optimization. If nanos are used, servers **SHOULD** provide bridge functions or conversion guidance aligned with Mangle's documented "time bridge" functions between column timestamps and temporal reasoning.

#### Temporal Semantics Obligations

- A server that receives temporal facts **MUST** evaluate them relative to an explicit evaluation time if the client supplies it, because Mangle's temporal operators over relative durations depend on a reference point called `now`.
- If the client omits evaluation time, the server **MAY** use its own clock, but **MUST** report the chosen evaluation time in the response for determinism auditing.

---

### Intent Evaluation Request

#### Intent Request Payload

```json
{
  "intent": {
    "name": "<string — e.g. 'checkout', 'diagnose_error', 'extract_summary'>",
    "params": { "<high-level intent parameters; NOT atomic steps>" }
  },
  "facts": [ "<array of Fact objects>" ],
  "eval_time": "<optional Time (recommended)>",
  "constraints": {
    "max_facts_created": "<integer — maps to Mangle-style created-fact limit controls>",
    "max_intervals_per_atom": "<integer — maps to Mangle temporal store interval limit controls>",
    "max_compute_ms": "<integer — server-side budget; exact mechanism is implementation-defined>"
  }
}
```

#### Server Behavior

- The server **MUST** load and analyze its `.mg` program(s), apply temporal validation checks, and reject "dangerous temporal recursion patterns" when configured to do so—consistent with Mangle's integration of temporal recursion checks and its documentation of dangerous patterns.
- The server **SHOULD** use coalescing and interval/fact limits by default for safety, reflecting Mangle's documented safeguards and the explicit termination risk of temporal reasoning.

### Intent Evaluation Response

The response returns a list of **MacroTool** objects computed for the provided intent and facts.

#### Intent Response Payload

```json
{
  "eval_time_used": "<Time>",
  "macro_tools": [ "<array of MacroTool objects>" ],
  "proof_hints": "<optional — server-defined minimal explanation>",
  "required_skills": [ "<optional array of Skill objects>" ]
}
```

> `proof_hints` (e.g., which predicates triggered a macro) **MUST NOT** expose internal chain-of-thought. `required_skills` are included only when proven necessary.

#### MacroTool Object

```json
{
  "macro_id": "<string — stable identifier for this synthesized tool instance>",
  "name": "<string — tool name presented to the client>",
  "description": "<string>",
  "input_schema": "<JSON Schema (recommended draft 2020-12)>",
  "output_schema": "<optional JSON Schema>",
  "context_injection": {
    "instructions": "<string (concise)>",
    "skills": [ "<array of Skill references or embedded skills>" ]
  },
  "safety": {
    "requires_user_confirmation": "<boolean>",
    "side_effects": [ "<e.g. 'network', 'filesystem', 'payments'>" ]
  },
  "validity": {
    "not_before": "<Time>",
    "expires_at": "<Time>"
  }
}
```

> **Why JSON Schema:** MCP's tool model is schema-described, and JSON Schema 2020-12 is a common interoperable target for describing structured JSON inputs/outputs.

---

### Skill Objects and Conditional Injection

MangleCP treats workflows/instructions as **first-class payload components** only when the server proves they are required.

A **Skill** in MangleCP is a JSON representation aligned with the "Agent Skills" concept: a bundle of instructions/resources that can be loaded dynamically to improve task performance.

#### Skill Object

```json
{
  "skill_id": "<string>",
  "name": "<string>",
  "description": "<string>",
  "content": "<string — minimal instructions>",
  "resources": [
    { "uri": "<string>", "type": "<string>" }
  ]
}
```

> The server **MUST NOT** inject skills indiscriminately; it **SHOULD** inject only the minimal subset required for the selected macro-tool, mirroring the browserNerd and Agent Skills emphasis on efficient, contextual disclosure.

---

### Macro-Tool Invocation

The macro invocation is where the client passes high-level arguments and the server executes the atomic chain.

#### Invoke Request Payload

```json
{
  "macro_id": "<string>",
  "args": "<object — must validate against input_schema>",
  "eval_time": "<optional Time (if macro execution depends on temporal rules)>"
}
```

#### Invoke Response Payload

```json
{
  "result": "<object — must validate against output schema if provided>",
  "state_delta": [ "<optional array of Facts — newly observed or derived facts>" ],
  "observability": {
    "summary": "<string>",
    "events": [ "<array of small structured events (implementation-defined)>" ]
  },
  "next": {
    "suggested_intents": [
      { "name": "<string>", "params": { } }
    ]
  }
}
```

> This is the MangleCP analog to the browserNerd model where a single call (e.g., `execute-plan`) executes multiple steps and returns only what changed or what is relevant.

---

### Error Model

Errors are **first-class protocol messages**.

#### Error Payload

```json
{
  "code": "<string — e.g. 'invalid_facts', 'invalid_temporal_pattern', 'schema_validation_failed', 'auth_required', 'budget_exceeded'>",
  "message": "<string>",
  "details": {
    "violations": [ "<array of structured validation issues>" ]
  }
}
```

> Servers **SHOULD** surface temporal non-termination hazards explicitly when they detect "dangerous patterns," reflecting Mangle's documented categorization and safeguards.

---

## Security, Privacy, and Registries

### Authentication and Authorization

For any network-accessible transport (including HTTP), MangleCP servers **MUST** require authentication by default.

This stance is motivated by two grounded facts:

1. MCP's authorization is explicitly **OPTIONAL** per its spec.
2. Real-world scanning and incident-style analysis has shown that optional authorization combined with exposed endpoints leads to tool enumeration and potential pivoting risk.

MangleCP does not mandate a particular auth mechanism in this draft, but for HTTP it **SHOULD** support OAuth 2.1-style bearer tokens in the `Authorization` header, aligning with the well-established pattern used by MCP's authorization guidance when auth is enabled.

### Prompt Injection and Tool Misuse Surface Reduction

Because MangleCP's server computes macro-tools and can mark them with side-effects metadata and confirmation requirements, it can **reduce client-side prompt injection surface** by limiting what the model can invoke at any moment to only those macro-tools proven relevant by rules. This is consistent with the "progressive disclosure" motivation demonstrated by browserNerd's design (request only necessary detail, batch actions, reduce uncontrolled exploration).

### Data Minimization and Temporal Privacy

Temporal facts can encode sensitive behavioral histories (e.g., events and durations). Because Mangle can evaluate relative-time queries based on a provided evaluation time and bounded windows:

- **Clients SHOULD** send only the minimal time-bounded facts required for a given intent.
- **Servers SHOULD** return only minimal derived facts needed to support follow-on intents.

---

## IANA-Style Considerations for a Future Standardization Track

If MangleCP is progressed as an Internet-Draft, it **SHOULD** define:

1. A **Well-Known URI registration** for `manglecp` (e.g., `/.well-known/manglecp/manifest.json`) per RFC 8615 and IANA well-known URI procedures.
2. **Media types** such as `application/manglecp+json` (registration policy and change control would follow the general IANA considerations guidance framework).
3. A **small registry** for standard `error.code` values and `facts_profile` identifiers.

> *These are forward-looking registry notes; the present document is a protocol whitepaper draft rather than a completed IANA registration request.*
