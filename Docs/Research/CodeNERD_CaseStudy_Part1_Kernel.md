# Case Study: codeNERD — Part 1: Kernel Architecture

> Source-verified lessons from codeNERD's Mangle kernel for MangleCP protocol design.
> Based on exhaustive Go source code inspection of ~220K LOC across 907 Go files and 274 `.mg` files.

---

## Overview

codeNERD is a Go CLI coding agent (~220K LOC) with the deepest Mangle integration of any system studied. Unlike BrowserNERD (MCP server), spec-implementer, and spec-generator (MCP server plugins), codeNERD is an **MCP client** — it connects TO external MCP servers while using Mangle as its internal reasoning kernel.

The Mangle kernel is not a bolt-on feature. It is the **central nervous system**: every action the agent takes — file writes, bash commands, MCP tool calls, subagent spawns — is proposed as a Mangle fact, evaluated against constitutional safety rules, gated by permission predicates, and routed through a virtual predicate dispatch system. The kernel manages 500K+ derived fact limits, cached stratification, external predicate bridges to Go runtime data, a transaction API for atomic state mutations, and a self-healing learned rule system.

This is the strongest evidence yet that MangleCP's thesis — **Mangle reasoning as the gatekeeper for AI tool access** — works at scale in production.

---

## Source-Verified Architecture

### Actual Codebase Stats

| Component | File(s) | Lines | Key Insight |
|-----------|---------|-------|-------------|
| Low-Level Engine | `internal/mangle/engine.go` | 1,041 | `ConcurrentFactStore`, gas limit via `DerivedFactsLimit` (100K default), cached stratification, type-aware coercion |
| Kernel Core | `internal/core/kernel_types.go` | 99 | "Hollow Kernel" — `RealKernel` struct with boot facts, cached atoms, virtual store, max 250K facts / 500K derived limit |
| Kernel Eval | `internal/core/kernel_eval.go` | 311 | Fresh store per evaluation, external predicates from VirtualStore, stratified trust ordering |
| Kernel Init | `internal/core/kernel_init.go` | 552 | 24 schema modules + policy directory + 8 core modules + user extensions from `.nerd/mangle/` |
| Kernel Query | `internal/core/kernel_query.go` | 578 | Pattern-matching queries with variables as wildcards, `UpdateSystemFacts()` for git state |
| Kernel Facts | `internal/core/kernel_facts.go` | 1,208 | Assert/Retract, `KernelTransaction` for atomic batches, `sanitizeFactForNumericPredicates` |
| Kernel Policy | `internal/core/kernel_policy.go` | 490 | Learned rules with sandbox validation, hot-load with repair interceptor, autopoiesis hooks |
| Virtual Store | `internal/core/virtual_store.go` | 1,729+ | FFI Router — 60+ action types, constitutional safety, permission cache, MCP client dispatch |
| External Preds | `internal/core/external_predicates.go` | 154 | 10 external predicates bridging Mangle to Go runtime (knowledge graph, embeddings, traces) |
| Fact Categories | `internal/core/fact_categories.go` | 184 | Three-tier: Persistent (survive sessions), Ephemeral (session-scoped), Derived (rule-computed) |
| Constitution | `internal/core/defaults/policy/constitution.mg` | 337 | Default-deny permission model, 60+ `safe_action` facts, appeal mechanism |
| Strategy | `internal/core/defaults/policy/strategy.mg` | 54 | Intent-to-strategy mapping rules |
| Activation | `internal/core/defaults/policy/activation.mg` | 28 | Spreading activation network for fact salience |

**Total kernel source: ~6,900+ lines of Go + ~420+ lines of .mg policy**

### The "Hollow Kernel" Pattern

The `RealKernel` struct (`kernel_types.go:18-55`) reveals a deliberate architectural inversion — the kernel holds no domain logic itself; it is a **shell** around Mangle evaluation:

```go
type RealKernel struct {
    mu              sync.RWMutex
    facts           []Fact              // EDB — ground truth
    cachedAtoms     []ast.Atom          // Pre-parsed atoms (optimization)
    factIndex       map[string]struct{} // Deduplication index
    bootFacts       []string            // Facts loaded at init
    bootIntents     []string            // Intents loaded at init
    bootPrompts     []string            // System prompts from boot
    schemas         string              // Concatenated .mg schema text
    policy          string              // Concatenated .mg policy text
    learned         string              // Hot-loaded rules (autopoiesis)
    strata          []ast.Stratum       // Cached stratification
    predToStratum   map[ast.PredicateSym]int
    virtualStore    *VirtualStore       // FFI router for actions
    derivedFactLimit int               // Default: 500,000
    maxFacts        int                 // Default: 250,000
    repairInterceptor func(...)         // Autopoiesis hook
    predicateCorpus   []string          // Known predicate names
}
```

The kernel is "hollow" because **all reasoning lives in `.mg` files** — loaded via `go:embed` from `defaults/*.mg`, `defaults/schema/*.mg`, and `defaults/policy/*.mg`. The Go code is purely the evaluation harness. This is the MangleCP model: the server is a harness, the protocol logic is declarative.

---

## Key Patterns Worth Adopting

### 1. Fresh-Store Evaluation with Cached EDB

Every `evaluate()` call (`kernel_eval.go:198-311`) creates a **fresh fact store**, populates it from the kernel's cached `[]ast.Atom`, and runs stratified evaluation:

```go
func (k *RealKernel) evaluate() error {
    store := factstore.NewSimpleInMemoryStore()
    // Populate from cached EDB atoms
    for _, atom := range k.cachedAtoms {
        store.Add(atom)
    }
    // Run full stratified evaluation with gas limit
    _, err := engine.EvalStratifiedProgramWithStats(
        k.programInfo, store, engine.DerivedFactsLimit(k.derivedFactLimit))
    return err
}
```

This means **derived facts never accumulate across evaluations**. Each evaluation is a clean slate with only EDB facts as input. Derived facts exist only for the duration of the evaluation and subsequent queries.

**MangleCP lesson:** This is the correct model for a stateless protocol. Each intent evaluation starts from the server's current EDB (facts it knows), not from stale derived facts from previous requests. MangleCP servers SHOULD use fresh-store evaluation per intent request.

### 2. Stratified Trust Ordering for Rule Sources

The `rebuildProgram()` function (`kernel_eval.go:19-82`) concatenates rule sources in a **specific trust order**:

```go
func (k *RealKernel) rebuildProgram() error {
    fullSource := k.schemas + "\n" + k.policy + "\n" + k.learned
    // Parse → Analyze → Stratify
    programInfo, err := analysis.AnalyzeAndCompile(parsed)
    k.strata = programInfo.Strata
    k.predToStratum = programInfo.PredToStratum
    return nil
}
```

Three layers:
1. **Schemas** — embedded via `go:embed`, immutable, define predicate declarations and base rules
2. **Policy** — embedded or loaded from workspace `.nerd/mangle/policy/`, defines permission and safety rules
3. **Learned** — hot-loaded at runtime via autopoiesis, validated in sandbox before acceptance

This ordering ensures learned rules (potentially LLM-generated) cannot override schema definitions or safety policy. Mangle's stratification algorithm naturally enforces that higher-stratum rules cannot contradict lower-stratum facts.

**MangleCP lesson:** MangleCP servers should distinguish rule sources by trust level. The protocol's `facts_profile` should support declaring which rule layers are server-intrinsic (immutable) vs. session-scoped (mutable). Client-pushed facts should be treated as highest-stratum (least trusted).

### 3. Three-Tier Fact Categorization

`fact_categories.go:18-184` defines three categories that determine fact lifecycle:

| Category | Examples | Lifecycle | MangleCP Mapping |
|----------|----------|-----------|------------------|
| **Persistent** | `user_preference`, `project_config`, `learned_rule` | Survive sessions, loaded from disk | Server-side EDB |
| **Ephemeral** | `user_intent`, `pending_action`, `next_action`, `active_shard` | Session-scoped, cleared on restart | Intent request facts |
| **Derived** | `permitted`, `safe_action`, `context_atom`, `selected_result` | Computed by rules, never persisted | IDB — evaluation output |

The `filterBootFacts()` function uses this categorization to prevent **stale action replay** — ephemeral facts from a crashed session are not reloaded on restart, preventing the agent from re-executing completed or failed actions.

**MangleCP lesson:** MangleCP MUST distinguish EDB (server facts) from IDB (derived) from session-scoped (client-pushed) facts. The protocol's `state_delta` responses should tag facts by category. Stale ephemeral facts from disconnected sessions must be garbage-collected.

### 4. Transaction API for Atomic State Mutations

`kernel_facts.go:980-1208` implements `KernelTransaction` — a transactional wrapper that batches retract+assert operations and commits atomically:

```go
type KernelTransaction struct {
    kernel     *RealKernel
    toRetract  []RetractOp
    toAssert   []string
}

func (tx *KernelTransaction) Retract(pred string) {
    tx.toRetract = append(tx.toRetract, RetractOp{Predicate: pred})
}

func (tx *KernelTransaction) Assert(fact string) {
    tx.toAssert = append(tx.toAssert, fact)
}

func (tx *KernelTransaction) Commit() error {
    k.mu.Lock()
    defer k.mu.Unlock()
    // 1. Execute all retractions
    for _, op := range tx.toRetract { ... }
    // 2. Execute all assertions
    for _, fact := range tx.toAssert { ... }
    // 3. Single rebuild + evaluation
    return k.rebuildAndEval()
}
```

The key insight: retract+assert+rebuild happens under a single lock acquisition with one evaluation pass. Without transactions, each retract or assert would trigger its own rebuild+eval, making N operations cost N evaluations instead of 1.

**MangleCP lesson:** MangleCP's `invoke` response includes `state_delta` (facts to add/remove). These deltas should be applied **transactionally** — all retractions and assertions in a single delta are committed atomically with one evaluation pass. The protocol should specify this.

### 5. External Predicates as FFI Bridges

`external_predicates.go:18-154` defines **10 external predicates** that bridge Mangle evaluation to Go runtime data:

| Predicate | Mode | What It Bridges |
|-----------|------|-----------------|
| `query_learned(+Pattern, -Rule)` | `bf` | Learned rules store |
| `query_session(+Pred, +Key, -Value)` | `bbf` | Session state |
| `recall_similar(+Query, +K, -Result)` | `bbf` | Embedding-based similarity search |
| `query_knowledge_graph(+Subject, +Relation, -Object)` | `bbf` | Knowledge graph |
| `query_strategic(+Category, +Key, -Value)` | `bbf` | Strategic planning state |
| `query_activations(+Concept, -Score)` | `bf` | Spreading activation network |
| `has_learned(+Pattern)` | `b` | Existence check for learned rules |
| `query_traces(+Pred, +Arg1, +Arg2, +Limit, -Result)` | `bbbbb` | Execution trace history |
| `query_trace_stats(+Pred, +Window, +Metric, -Value)` | `bbbf` | Trace aggregation |
| `string_contains(+Haystack, +Needle)` | `bb` | String matching utility |

Each predicate is registered as a `virtualExternalPredicate` implementing `ExternalPredicateCallback`. Mode declarations (`b` = bound input, `f` = free output) enforce binding patterns — Mangle will reject queries that don't bind the required input positions.

These predicates are injected into evaluation via `virtualStore.BuildExternalPredicates()` which is called during `evaluate()`:

```go
externalPreds := k.virtualStore.BuildExternalPredicates()
// Filter to only predicates declared as external in schema
for _, decl := range k.programInfo.Decls {
    if decl.IsExternal() {
        // Register the external predicate for this evaluation
    }
}
```

**MangleCP lesson:** MangleCP servers will need external predicates to bridge Mangle evaluation to databases, APIs, and runtime state. The protocol should define how servers declare their external predicate capabilities in the manifest, so clients know what data sources back the server's reasoning.

### 6. Constitutional Safety — Dual Layer Defense

Safety enforcement operates at **two independent layers**:

**Layer 1: Mangle policy rules** (`constitution.mg:1-337`)

Default-deny permission model:

```mangle
# Nothing is permitted unless explicitly safe AND not dangerous
permitted(Action, Target, Payload) :-
    safe_action(Action, Target, Payload),
    !dangerous_content(Action, Target, Payload).

# 60+ explicit safe_action facts
safe_action("read", Target, _) :- file_exists(Target).
safe_action("write", Target, _) :- in_workspace(Target).
safe_action("bash", Cmd, _) :- safe_command(Cmd).
```

Appeal mechanism with temporary overrides:

```mangle
# User can override a blocked action with justification
permitted(Action, Target, Payload) :-
    appeal_granted(Action, Target, Justification),
    !expired_appeal(Action, Target).
```

**Layer 2: Go-level constitutional checks** (`virtual_store.go`)

Hardcoded checks that **cannot be overridden by Mangle rules**:

- No destructive commands (`rm -rf /`, `mkfs`, `dd`, `:(){:|:&};:`)
- No secret exfiltration (blocks `curl` with env vars, `cat ~/.ssh/*`)
- Path traversal protection (resolves symlinks, rejects `../` escapes)
- No system file modification (`/etc/passwd`, `/etc/shadow`, crontab)

The `RouteAction` pipeline in `virtual_store.go` runs both layers sequentially:

1. Boot guard (reject actions during kernel initialization)
2. Parse action from `next_action` atom
3. **Go constitutional check** (hardcoded, unforgeable)
4. **Mangle permission gate** (query `permitted(Action, Target, Payload)`)
5. Execute action
6. Post-action validation
7. Fact injection (result facts back into kernel)

**MangleCP lesson:** MangleCP's security model should follow this dual-layer pattern. Protocol-level constraints (wire format validation, message size limits) are the Go layer — unforgeable, always enforced. Mangle-level policy rules are the declarative layer — configurable, auditable, explainable. Neither layer alone is sufficient.

### 7. Gas Limits and Evaluation Safety

Unlike BrowserNERD and spec-implementer (which have no evaluation limits), codeNERD implements multiple safety valves:

| Safety Valve | Default | Source |
|--------------|---------|--------|
| `DerivedFactsLimit` | 100,000 | `engine.go` — passed to `EvalStratifiedProgramWithStats` |
| `derivedFactLimit` | 500,000 | `kernel_types.go` — kernel-level override |
| `maxFacts` | 250,000 | `kernel_types.go` — caps EDB size |

The engine-level gas limit (`engine.go`) works by counting derived facts during evaluation and returning an error when the limit is exceeded:

```go
_, err := engine.EvalStratifiedProgramWithStats(
    k.programInfo, store, engine.DerivedFactsLimit(k.derivedFactLimit))
```

This prevents infinite derivation from pathological rules (e.g., unbounded recursion, cartesian products from poorly constrained joins).

**MangleCP lesson:** The protocol's `constraints.max_facts_created` maps directly to this gas limit. MangleCP servers MUST enforce derivation limits. The protocol should require servers to report their limits in the manifest and return `413 Derivation Limit Exceeded` when hit.

### 8. Quiescent Boot and Boot Guard

`kernel_init.go` implements a careful boot sequence:

1. Load 24 embedded schema modules via `go:embed`
2. Load policy directory (constitution, strategy, activation)
3. Load 8 core modules (taxonomy, inference, jit_compiler, etc.)
4. Load user extensions from `.nerd/mangle/`
5. Validate schema consistency
6. `filterBootFacts()` — remove ephemeral facts to prevent stale action replay
7. Assert boot facts
8. Initial evaluation
9. Consume boot intents and prompts

The **boot guard** in `virtual_store.go` prevents any action dispatch during steps 1-8. Actions proposed by stale facts from a previous session are blocked until the kernel reaches quiescent state.

**MangleCP lesson:** MangleCP servers need a boot/ready protocol. The manifest handshake should include a `status` field indicating whether the server is fully initialized. Clients should not send intent requests until the server reports ready. This prevents race conditions where intent evaluation runs against an incomplete rule set.

### 9. Cached Atom Optimization

The kernel maintains two parallel representations of EDB facts:

```go
facts       []Fact          // String representations for display/serialization
cachedAtoms []ast.Atom      // Pre-parsed AST atoms for fast store population
```

When `addFactIfNewLocked()` is called, it parses the fact string into an `ast.Atom` **once** and caches it. Subsequent `evaluate()` calls populate the fresh store directly from `cachedAtoms`, avoiding repeated string parsing.

The `factIndex map[string]struct{}` provides O(1) deduplication — identical fact strings are rejected before parsing.

**MangleCP lesson:** MangleCP servers handling high fact volumes should pre-parse client-pushed facts into AST atoms on receipt and cache them for repeated evaluations. The protocol's fact format (currently JSON) should be designed to minimize server-side parsing cost.

### 10. Self-Healing Learned Rules (Autopoiesis Foundation)

`kernel_policy.go:290-490` implements `HotLoadLearnedRule()` — the mechanism by which the agent learns new Mangle rules at runtime:

```
Pipeline: LLM generates rule → normalize → unsafe negation check →
          sandbox compile → schema validation → pathological pattern check →
          persist to learned.mg
```

The **repair interceptor** (`repairInterceptor` field on `RealKernel`) provides a hook for the autopoiesis system to attempt automatic repair when a learned rule fails validation:

1. Parse the rule
2. Check for unsafe negation (variables in negated subgoals not bound in positive subgoals)
3. **Sandbox compile**: create a throwaway copy of the kernel, add the rule, run full evaluation — if it crashes or exceeds gas limits, reject
4. Schema validation: ensure the rule's head predicate matches a declared schema
5. Pathological pattern check: reject rules with known-bad patterns (unbounded recursion, cartesian products)
6. If all pass: append to `learned` string, persist to `learned.mg` file

`GenerateValidatedRule()` goes further — it uses a **feedback loop with the LLM**: if validation fails, the error message is sent back to the LLM with instructions to fix the rule, up to a retry budget.

**MangleCP lesson:** MangleCP servers that accept client-pushed rules (a potential v2 feature) MUST implement this validation pipeline. The sandbox-compile pattern — try the rule in an isolated copy before accepting it — is essential. The protocol should define a `submit_rule` message type with validation semantics.

---

## Things to Improve On

### 1. No TemporalStore Despite Massive Temporal Semantics

Like all four studied systems, codeNERD uses `factstore.NewSimpleInMemoryStore()`. Zero use of `factstore.NewTemporalStore()` or DatalogMTL temporal operators (`@[T1, T2]`, `<-[Duration]`), despite numerous predicates with temporal semantics:

- `active_shard` — which context shard is currently active (time-bounded by nature)
- `pending_action` — actions awaiting execution (expire after completion)
- `appeal_granted` / `expired_appeal` — time-limited permission overrides
- `stagnation_detected` — detection of lack of progress (inherently temporal)

All temporal logic is implemented in Go code rather than Mangle rules.

**MangleCP improvement:** DatalogMTL would replace hundreds of lines of Go temporal logic:

```mangle
# codeNERD Go code: manually check if appeal has expired
# MangleCP Mangle rule: temporal annotation handles it
appeal_valid(Action, Target) @[T, T+3600000] :-
    appeal_granted(Action, Target, _) @[T, _].

# Stagnation detection (no progress in 5 minutes)
stagnation(Goal) :-
    active_goal(Goal),
    !progress_event(Goal) <-[300000].
```

### 2. ConcurrentFactStore Adds Complexity Without Temporal Benefit

`engine.go` wraps Mangle's `SimpleInMemoryStore` in a `ConcurrentFactStore` with mutex-based thread safety. This adds ~200 lines of locking code. The `TemporalStore` in upstream Mangle already handles concurrent access patterns through interval coalescing.

**MangleCP improvement:** Using `TemporalStore` would eliminate the need for a custom concurrent wrapper while adding temporal query capabilities.

### 3. VirtualStore Monolith (1,729+ Lines)

`virtual_store.go` is a 1,729+ line monolith that handles:
- Action routing for 60+ action types
- Constitutional safety checks
- Permission caching
- MCP client dispatch
- Tool registry management
- Dreamer (speculative execution)
- Post-action validation

This violates separation of concerns. A MangleCP server should decompose these into distinct layers with clean interfaces.

**MangleCP improvement:** The protocol should define distinct message types for each concern — action dispatch, safety evaluation, capability registry — rather than funneling everything through a single router. Each message type maps to a distinct server-side handler.

### 4. String-Based Fact Representation

Facts flow through the kernel as strings (`[]Fact` where `Fact` is a string type), requiring repeated parsing:

```go
type Fact = string  // e.g., "next_action(\"bash\", \"ls\", \"\")"
```

The `cachedAtoms` optimization partially mitigates this, but string facts still leak into APIs, logging, serialization, and display. Type errors from malformed strings are caught late (at parse time) rather than early (at construction time).

**MangleCP improvement:** MangleCP's wire format should use structured JSON for facts — `{"predicate": "next_action", "args": ["bash", "ls", ""]}` — so that parsing happens once at the wire boundary, not repeatedly inside the server.

### 5. Monolithic Schema Loading (24 Modules at Boot)

`kernel_init.go` loads 24 schema modules + policy + core modules at every boot, regardless of what the agent will be doing. There is no lazy loading or schema subsetting based on the current task.

**MangleCP improvement:** MangleCP servers could implement **schema sharding** — loading only the rule modules relevant to the declared intent. The manifest could advertise available schema modules, and the client's intent request could trigger selective loading. This aligns with the "smallest relevant set" principle.

---

## Architecture Decisions to Question

| codeNERD Pattern | Worth Keeping? | Why |
|---|---|---|
| Hollow Kernel (Go shell + .mg logic) | **Yes — canonical** | This IS MangleCP's server architecture model |
| Fresh-store evaluation per cycle | **Yes** | Prevents derived fact accumulation, deterministic |
| Stratified trust ordering (schema > policy > learned) | **Yes — formalize** | MangleCP should define trust levels in the protocol |
| Three-tier fact categories | **Yes — adopt** | EDB/IDB/session-scoped distinction is essential |
| Transaction API (atomic retract+assert) | **Yes — adopt** | `state_delta` should be transactional |
| External predicates as FFI | **Yes — declare in manifest** | Servers should advertise their FFI bridges |
| Dual-layer safety (Go + Mangle) | **Yes — both layers** | Protocol constraints + declarative policy |
| Gas limits (100K-500K derived facts) | **Yes — require in protocol** | Map to `max_facts_created` constraint |
| Boot guard / quiescent state | **Yes — add to manifest** | Server readiness signaling |
| Cached atom optimization | **Implementation detail** | Not protocol-relevant but good practice |
| Self-healing learned rules | **Future — v2** | `submit_rule` message type with sandbox validation |
| `SimpleInMemoryStore` (no temporal) | **No — use TemporalStore** | Temporal reasoning is MangleCP's differentiator |
| String-based fact representation | **No — use structured JSON** | Wire format should prevent late parsing errors |
| Monolithic VirtualStore | **No — decompose** | Separate concerns into distinct protocol message types |
| 24-module monolithic boot | **Reconsider** | Schema sharding could improve intent-scoped evaluation |

---

## Summary

codeNERD's Mangle kernel is the most sophisticated Mangle integration studied — a 6,900+ line Go harness around declarative `.mg` rules that govern every action the agent takes. The "Hollow Kernel" pattern — where Go code is purely the evaluation shell and all reasoning lives in Mangle files — is the strongest validation of MangleCP's server architecture model. The fresh-store evaluation pattern ensures deterministic, stateless reasoning per cycle. Stratified trust ordering (schema > policy > learned) provides a formal security hierarchy for rule sources. The three-tier fact categorization (persistent/ephemeral/derived) maps directly to MangleCP's need to distinguish server EDB, client-pushed facts, and derived IDB.

The most important patterns for MangleCP: **transactional state mutations** (atomic retract+assert with single evaluation), **external predicates as FFI** (bridging Mangle to runtime data sources), **dual-layer safety** (unforgeable Go checks + configurable Mangle policy), and **gas limits** (preventing unbounded derivation). The biggest gap remains temporal reasoning — codeNERD implements all temporal logic in Go code despite Mangle's DatalogMTL capabilities being available upstream. MangleCP should close this gap by building on `TemporalStore` from day one.
