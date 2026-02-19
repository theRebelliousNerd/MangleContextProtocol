# Case Study: codeNERD — Part 2: JIT Compilation, MCP Client, and Autopoiesis

> Source-verified lessons from codeNERD's JIT tool/prompt compilers, MCP client integration, intent routing, and autopoietic tool synthesis for MangleCP protocol design.
> Based on exhaustive Go and .mg source code inspection.

---

## Overview

Part 1 covered codeNERD's Mangle kernel — the evaluation harness. This Part 2 covers the **systems built on top of that kernel**: the JIT Tool Compiler (which selects MCP tools per-context using hybrid Mangle+vector scoring), the JIT Prompt Compiler (which selects context atoms using 6-phase Mangle evaluation), the intent routing rules (which map user intents to personas, strategies, and tool sets), and the Ouroboros autopoiesis loop (which synthesizes new tools at runtime via Mangle-governed state machines).

These systems collectively demonstrate that **Mangle can govern the full lifecycle of tool selection, context assembly, and capability synthesis** — the exact workflow MangleCP aims to standardize as a wire protocol.

---

## JIT Tool Compiler: Hybrid Logic+Vector Tool Selection

### Source-Verified Architecture

| Component | File | Lines | Key Insight |
|-----------|------|-------|-------------|
| Compiler | `internal/mcp/compiler.go` | 361 | 5-phase pipeline: discover → embed → score → select → fit |
| Analyzer | `internal/mcp/analyzer.go` | 524 | LLM-powered semantic metadata extraction with heuristic fallback |
| Renderer | `internal/mcp/renderer.go` | 228 | Three-tier rendering: full schema, condensed one-line, minimal name-only |
| Store | `internal/mcp/store.go` | 627 | SQLite + `sqlite-vec` for ANN search, usage stats tracking |
| Policy | `internal/mcp/policy_mcp.mg` | 178 | Pure Mangle rules for tool selection scoring and render mode assignment |

### The 5-Phase Pipeline (`compiler.go`)

The `JITToolCompiler.Compile()` function (`compiler.go:45-220`) implements a 5-phase tool selection pipeline:

**Phase 1: Discovery** — Get all registered MCP tools from the MCPClientManager. This includes both static tools (from connected MCP servers) and Ouroboros-generated tools (synthesized at runtime).

**Phase 2: Vector Scoring** — For each tool, compute embedding similarity against the current context using the embedding engine. This produces a `float64` score per tool.

**Phase 3: Mangle Fact Injection** — Assert temporary vector scores as Mangle facts:
```mangle
mcp_tool_vector_score("browsernerd__browser-observe", 0.87).
mcp_tool_vector_score("spec-impl__get_execution_plan", 0.23).
```

**Phase 4: Mangle Selection** — Query the kernel for `mcp_tool_selected(ShardType, ToolID, RenderMode)`. The Mangle rules in `policy_mcp.mg` combine logic-based relevance with vector scores to produce the final selection. If the Mangle query fails, fallback to pure affinity scoring (70% logic + 30% vector).

**Phase 5: Token Budget Fitting** — Sort selected tools by combined score, then progressively demote tools to fit within the token budget:
- High-scoring tools → `RenderModeFull` (full JSON schema block)
- Medium-scoring tools → `RenderModeCondensed` (one-line description)
- Low-scoring tools → `RenderModeMinimal` (name only)
- Below threshold → excluded entirely

### The Selection Policy (`policy_mcp.mg`)

The 178-line policy file defines pure Mangle rules for tool selection:

**Section 50.1 — Availability:**
```mangle
mcp_tool_available(ToolID) :-
    mcp_server_status(ServerID, "connected"),
    mcp_tool_server(ToolID, ServerID).
```

**Section 50.2 — Base Relevance (shard affinity):**
```mangle
mcp_tool_base_relevant(ToolID, Affinity) :-
    mcp_tool_available(ToolID),
    active_shard(ShardType),
    mcp_tool_shard_affinity(ToolID, ShardType, Affinity),
    Affinity >= 30.
```

**Section 50.3 — Intent Boost (+30):**
```mangle
mcp_tool_intent_boost(ToolID, 30) :-
    user_intent(_, IntentType, _),
    mcp_intent_capability(IntentType, Capability),
    mcp_tool_capability(ToolID, Capability).
```

**Section 50.4 — Domain Boost (+20/+10):**
```mangle
mcp_tool_domain_boost(ToolID, 20) :-
    active_domain(Domain),
    mcp_tool_domain(ToolID, Domain).
```

**Section 50.5 — Combined Score:**
```mangle
# Combined score = logic_score * 0.7 + vector_score * 0.3
mcp_tool_combined_score(ToolID, Score) :-
    mcp_tool_logic_score(ToolID, LogicScore),
    mcp_tool_vector_score(ToolID, VectorScore),
    Score = LogicScore * 0.7 + VectorScore * 0.3.
```

**Section 50.6 — Render Mode Assignment:**
```mangle
mcp_tool_render_mode(ToolID, "full") :- mcp_tool_combined_score(ToolID, S), S >= 70.
mcp_tool_render_mode(ToolID, "condensed") :- mcp_tool_combined_score(ToolID, S), S >= 40, S < 70.
mcp_tool_render_mode(ToolID, "minimal") :- mcp_tool_combined_score(ToolID, S), S >= 20, S < 40.
```

**Section 50.7 — Skeleton Tools (always full):**
```mangle
mcp_tool_render_mode(ToolID, "full") :-
    mcp_tool_skeleton(ToolID).
# Skeleton: filesystem, search — always needed
mcp_tool_skeleton("filesystem__read_file").
mcp_tool_skeleton("filesystem__write_file").
mcp_tool_skeleton("search__grep").
```

**Section 50.8 — Intent-Capability Mapping EDB:**
```mangle
mcp_intent_capability("fix", "diagnostics").
mcp_intent_capability("fix", "code_modification").
mcp_intent_capability("test", "test_execution").
mcp_intent_capability("explore", "code_search").
mcp_intent_capability("explore", "documentation").
```

**MangleCP lesson:** This is the closest existing implementation to MangleCP's intent-scoped tool synthesis. The entire pipeline — discover → score → select → render — is governed by Mangle rules. MangleCP should formalize this as the server-side evaluation model: intent facts + tool capability facts → selection rules → ranked tool list with progressive disclosure levels.

### Three-Tier Progressive Disclosure (`renderer.go`)

The `ToolRenderer` (`renderer.go:18-228`) converts a `CompiledToolSet` into client-facing output with three disclosure tiers:

| Tier | Render Mode | Content | Token Cost |
|------|-------------|---------|------------|
| **Primary** | `RenderModeFull` | Full tool name, description, JSON input schema, parameter details | High (~200-500 tokens/tool) |
| **Secondary** | `RenderModeCondensed` | One-line: `- tool_name: condensed description` | Low (~20 tokens/tool) |
| **Additional** | `RenderModeMinimal` | Comma-separated names: `tool_a, tool_b, tool_c` | Minimal (~5 tokens/tool) |

Output format options: markdown (primary), compact single-line, JSON, or invocation format.

**MangleCP lesson:** MangleCP's `evaluate` response should support this three-tier model. The `macro_tools` array could include a `disclosure_level` field (`full`, `condensed`, `minimal`) that tells the client how much detail the server chose to include for each tool, based on relevance scoring. This is token-efficient by design — only the most relevant tools get full schemas.

---

## JIT Prompt Compiler: 6-Phase Context Atom Selection

### Source: `internal/core/defaults/jit_compiler.mg` (176 lines)

The JIT Prompt Compiler selects which **context atoms** (pieces of knowledge, instructions, examples) to include in the agent's system prompt. It operates entirely in Mangle:

**Phase 1 — Skeleton (Mandatory):**
```mangle
is_selected(Atom, 100, "skeleton") :-
    is_mandatory(Atom).

is_selected(Atom, 90, "skeleton") :-
    has_constraint(Atom, Key, Value),
    satisfied_constraint(Key, Value),
    !blocked_by_context(Atom).
```

Atoms marked `is_mandatory` are always included. Atoms with constraints are included only when all constraints are satisfied AND no context explicitly blocks them. The `blocked_by_context` pattern is important: an atom is blocked only when the context **explicitly has a different value** for the constraint key, not merely when the constraint is absent.

**Phase 2 — Exclusion (Firewall):**
```mangle
is_excluded(Atom) :- prohibited(Atom, Reason).
is_excluded(Atom) :- incompatible(Atom, OtherAtom), is_selected(OtherAtom, _, _).
```

Explicit exclusion overrides any selection. Incompatibility rules prevent conflicting atoms from being co-selected.

**Phase 3 — Flesh (Probabilistic):**
```mangle
is_selected(Atom, Score, "vector") :-
    vector_hit(Atom, Score),
    Score > 30,
    !is_excluded(Atom).
```

Vector similarity hits above a threshold are included as "flesh" — non-mandatory but contextually relevant atoms.

**Phase 4 — Conflict Resolution:**
```mangle
# When two atoms conflict, the higher-scoring one wins
# Lexicographic tie-breaking on atom name for determinism
invalid(Atom1) :-
    conflicts_with(Atom1, Atom2),
    is_selected(Atom1, S1, _),
    is_selected(Atom2, S2, _),
    S1 < S2.
```

**Phase 5 — Dependency Resolution:**
```mangle
# Atoms can declare dependencies
missing_dep(Atom, Dep) :-
    atom_requires(Atom, Dep),
    !is_selected(Dep, _, _).

# Missing dependency cascades to invalidation
invalid(Atom) :- missing_dep(Atom, _).
```

**Phase 6 — Final Output:**
```mangle
selected_result(Atom, Priority, Source) :-
    is_selected(Atom, Priority, Source),
    !is_excluded(Atom),
    !invalid(Atom).
```

**MangleCP lesson:** This 6-phase pattern IS MangleCP's intent evaluation model. Replace "context atoms" with "macro-tools" and the phases map directly:
1. **Skeleton** → mandatory capabilities for the intent type
2. **Exclusion** → capabilities blocked by context or policy
3. **Flesh** → additional relevant capabilities found by similarity
4. **Conflict resolution** → when multiple tools serve the same purpose, pick the best
5. **Dependency resolution** → tools that require other tools as prerequisites
6. **Final output** → the validated, conflict-free set returned to the client

---

## Intent Routing: From User Intent to Persona, Strategy, and Tools

### Source: `internal/mangle/intent_routing.mg` (401 lines)

The intent routing rules form a comprehensive pipeline from raw user intent to actionable configuration:

### Action Type Derivation (Section 1)

Maps verb keywords to action categories:

```mangle
action_type(Intent, "create") :- intent_verb(Intent, Verb), create_verb(Verb).
action_type(Intent, "modify") :- intent_verb(Intent, Verb), modify_verb(Verb).
action_type(Intent, "delete") :- intent_verb(Intent, Verb), delete_verb(Verb).
action_type(Intent, "query")  :- intent_verb(Intent, Verb), query_verb(Verb).

create_verb("add"). create_verb("create"). create_verb("scaffold"). create_verb("init").
modify_verb("fix"). modify_verb("refactor"). modify_verb("update"). modify_verb("change").
query_verb("find"). query_verb("search"). query_verb("explain"). query_verb("explore").
```

### Persona Selection (Section 2)

Maps intent patterns to agent personas:

```mangle
persona(Intent, "/coder") :- action_type(Intent, "create").
persona(Intent, "/coder") :- action_type(Intent, "modify").
persona(Intent, "/tester") :- intent_verb(Intent, "test").
persona(Intent, "/reviewer") :- intent_verb(Intent, "review").
persona(Intent, "/researcher") :- action_type(Intent, "query").
# Default fallback
persona(Intent, "/coder") :- user_intent(Intent, _, _), !has_persona(Intent).
```

### Test Framework Detection (Section 3)

Derives the appropriate test framework from project configuration files:

```mangle
test_framework("go_test") :- file_exists("go.mod").
test_framework("jest") :- file_exists("jest.config.js").
test_framework("vitest") :- file_exists("vitest.config.ts").
test_framework("pytest") :- file_exists("pytest.ini").
test_framework("cargo_test") :- file_exists("Cargo.toml").
test_framework("rspec") :- file_exists("Gemfile"), file_contains("Gemfile", "rspec").
```

### Tool Selection Per Persona (Section 4)

Each persona gets a different tool set:

```mangle
tool_for_persona("/coder", "file_write").
tool_for_persona("/coder", "bash").
tool_for_persona("/coder", "search").
tool_for_persona("/tester", "bash").
tool_for_persona("/tester", "file_read").
tool_for_persona("/reviewer", "file_read").
tool_for_persona("/reviewer", "search").
tool_for_persona("/researcher", "search").
tool_for_persona("/researcher", "web_fetch").
```

### Context Priority Assignment (Section 7)

Assigns relevance scores to context files based on their relationship to the intent:

```mangle
context_priority(File, 100) :- intent_target(_, File).          # Direct reference
context_priority(File, 90)  :- failing_test(File).              # Failing test
context_priority(File, 70)  :- same_package(File, Target),      # Same package
                               intent_target(_, Target).
context_priority(File, 50)  :- imports(Target, File),           # Import dependency
                               intent_target(_, Target).
```

### TDD State Machine (Section 8)

A 3-state machine for test-driven development:

```mangle
tdd_state("red") :- failing_test(_), !all_tests_pass.
tdd_state("green") :- all_tests_pass, recent_fix(_).
tdd_state("refactor") :- all_tests_pass, !recent_fix(_).
```

### Strategy Selection (`strategy.mg`, 54 lines)

Maps intent+context combinations to execution strategies:

```mangle
strategy("/tdd_repair_loop") :- persona(_, "/fix"), has_diagnostics.
strategy("/breadth_first_survey") :- persona(_, "/explore").
strategy("/project_init") :- persona(_, "/scaffold").
strategy("/refactor_guard") :- persona(_, "/refactor").
strategy("/campaign_planning") :- persona(_, "/campaign").
```

### Spreading Activation (`activation.mg`, 28 lines)

A neural-network-inspired activation spreading model:

```mangle
activation(Concept, 100) :- new_fact(Concept).
activation(Tool, 80) :- active_goal(Goal), goal_capability(Goal, Cap), tool_capability(Tool, Cap).
activation(File, 90) :- intent_target(_, File).
activation(File, 70) :- file_depends(Target, File), intent_target(_, Target).
context_atom(X) :- activation(X, Score), Score > 30.
```

Facts with activation scores above 30 become `context_atom` — included in the agent's working context. This creates a dynamically expanding/contracting attention window based on relevance to the current goal.

**MangleCP lesson:** The intent routing pipeline (verb → action type → persona → tools → context priorities → strategy) is a complete reference implementation for MangleCP's intent evaluation layer. The protocol should support all these derivation stages. Particularly important: **context priority assignment** (which files/resources matter for this intent) and **strategy selection** (how to execute) are exactly what MangleCP's `evaluate` response should carry.

---

## MCP Client Integration

### Source: `internal/mcp/client.go` (469 lines)

codeNERD is an **MCP client**, not a server. It connects to external MCP servers (BrowserNERD, spec-implementer, etc.) and integrates their tools into its Mangle-governed tool selection:

**MCPClientManager** manages multiple server connections:
- Supports HTTP, Stdio, and SSE transports
- Auto-connects on startup via configuration
- `DiscoverTools()` fetches available tools from each server
- `processToolSchema()` extracts tool metadata (name, description, input schema)
- `CallTool()` invokes a tool on the appropriate server with usage recording

**Tool Analysis Pipeline** (`analyzer.go`, 524 lines):

When tools are discovered from an MCP server, the `ToolAnalyzer` extracts rich semantic metadata:

1. **LLM Analysis** (primary path): Sends the tool's JSON schema to an LLM with a structured extraction prompt. Extracts: categories, capabilities, domain, shard affinities (0-100 per shard type), use cases, condensed description.

2. **Heuristic Inference** (fallback): When LLM is unavailable, keyword matching against tool name and description to infer categories and basic affinities.

3. **Embedding Generation**: Computes vector embeddings for the tool's description + capabilities, stored in SQLite for future similarity search.

**Tool Storage** (`store.go`, 627 lines):

SQLite-backed with `sqlite-vec` extension for ANN (approximate nearest neighbor) search:

```sql
CREATE TABLE mcp_tools (
    id TEXT PRIMARY KEY,
    server_id TEXT,
    name TEXT,
    description TEXT,
    input_schema TEXT,
    embedding BLOB,        -- Vector embedding for similarity search
    categories TEXT,
    capabilities TEXT,
    domain TEXT,
    shard_affinities TEXT, -- JSON: {"coding": 80, "testing": 60, ...}
    usage_count INTEGER,
    success_rate REAL,
    avg_latency_ms REAL
);
```

`SemanticSearch()` uses the `sqlite-vec` extension for fast ANN search when available, falling back to brute-force cosine similarity computation over all stored embeddings.

**MangleCP lesson:** A MangleCP client connecting to multiple servers faces the same challenge: how to select relevant tools across servers for a given intent. codeNERD's approach — analyze tool schemas, generate embeddings, store metadata, use Mangle rules for selection — is a reference architecture for MangleCP clients. The protocol should define how servers expose tool metadata (beyond just JSON Schema) to enable this kind of client-side reasoning.

---

## Autopoiesis: The Ouroboros Loop

### Source: `internal/autopoiesis/ouroboros.go` (1,527 lines)

The Ouroboros loop is codeNERD's self-modification system — it **synthesizes new tools at runtime** through a Mangle-governed transactional state machine. This is the most advanced MangleCP-relevant pattern in any studied system.

### 4-Phase Transactional State Machine

```
Proposal → Audit → Simulation → Commit
    ↑                              |
    └──────── (loop) ──────────────┘
```

**Phase 1 — Proposal:**
- The agent identifies a capability gap (no existing tool handles the current need)
- Generates a tool specification: name, description, input/output schema, Go implementation
- Asserts `proposed_tool(ToolID, Spec)` into the Mangle kernel

**Phase 2 — Audit:**
- Mangle rules evaluate the proposal against safety policy
- Queries: `should_halt(Reason)` — checks for termination conditions
- Queries: `valid_transition(CurrentState, "audit")` — validates state machine transition
- Constitutional checks: does the proposed tool violate any safety rules?
- If audit fails, the proposal is rejected with a reason fact

**Phase 3 — Simulation:**
- **COW (Copy-On-Write) snapshots** via the differential engine
- Creates a throwaway copy of the kernel state
- Simulates tool execution in the sandbox
- PanicMaker: intentionally triggers edge cases to test robustness
- Thunderdome: adversarial testing — tries to break the tool
- Queries: `converged(ToolID)` — has the simulation stabilized?
- Queries: `stagnation_detected(ToolID)` — is the loop stuck?

**Phase 4 — Commit:**
- Compiles the tool implementation using Go cross-compiler
- Registers the tool in the runtime registry
- Asserts `tool_registered(ToolID)` and `has_capability(ToolID, Cap)` facts
- The tool is now available for selection by the JIT Tool Compiler
- Hot-reloads the tool without agent restart

### Mangle Governance Throughout

Every state transition is gated by Mangle queries:

```go
// Check if loop should halt
results, _ := kernel.Query("should_halt(Reason)")
if len(results) > 0 { return halt(results[0].Reason) }

// Validate state transition
results, _ = kernel.Query(fmt.Sprintf("valid_transition(%q, %q)", currentState, nextState))
if len(results) == 0 { return error("invalid transition") }

// Check convergence
results, _ = kernel.Query(fmt.Sprintf("converged(%q)", toolID))
if len(results) > 0 { return commit(tool) }

// Check stagnation
results, _ = kernel.Query(fmt.Sprintf("stagnation_detected(%q)", toolID))
if len(results) > 0 { return abort("stagnation") }
```

### Tool Compilation and Registry

Synthesized tools are compiled as Go plugins via cross-compilation, producing standalone binaries. The runtime registry manages them as subprocesses communicating via JSON over stdin/stdout — the same pattern as MCP stdio transport.

Each tool's capabilities are advertised as Mangle facts:

```mangle
tool_registered("ouroboros_csv_parser").
has_capability("ouroboros_csv_parser", "csv_parsing").
has_capability("ouroboros_csv_parser", "data_transformation").
tool_version("ouroboros_csv_parser", 3).
```

These facts feed back into the JIT Tool Compiler's selection pipeline — Ouroboros-generated tools compete with static tools and MCP server tools on equal footing.

**MangleCP lesson:** The Ouroboros loop is the ultimate validation of MangleCP's thesis — **Mangle reasoning governs not just tool selection but tool creation**. For MangleCP:

1. **Fact-driven capability advertisement** (`tool_registered`, `has_capability`) is the wire-format model for server capability negotiation
2. **COW snapshot simulation** before committing state changes aligns with MangleCP's need for "what-if" evaluation
3. **Transactional state machine governed by Mangle queries** is the reference architecture for stateful MangleCP interactions
4. **Hot-reload without restart** shows that MangleCP servers can dynamically update their capabilities between requests

---

## Supporting Systems

### Grammar-Constrained Decoding (`internal/mangle/grammar.go`, 860 lines)

The `AtomValidator` validates LLM-generated Mangle atoms against predicate schemas:

- 18 core predicates hardcoded with full schema definitions
- `UpdateFromProgramInfo()` extracts schemas dynamically from Mangle AST
- Layered validation: syntax → arity → type → name constants
- `RepairLoop()` for auto-fix with LLM feedback prompts

When the LLM generates a fact like `next_action("bash", "ls -la", "")`, the validator checks:
1. Is `next_action` a known predicate? (syntax)
2. Does it have 3 arguments? (arity)
3. Are all arguments strings? (type)
4. Is `"bash"` a valid action type constant? (name constants)

**MangleCP lesson:** MangleCP's wire format should include schema metadata that enables client-side validation before sending facts to the server. The `facts_profile` in the manifest should carry enough type information for clients to validate facts locally.

### Differential Engine (`internal/mangle/differential.go`, 631 lines)

Stratum-aware incremental evaluation:

- `ChainedFactStore`: layered read/write store where reads cascade through layers but writes go to the top layer
- COW `Snapshot()`: creates a copy-on-write snapshot for speculative evaluation without modifying the base store
- `RegisterVirtualPredicate`: demand-driven loading — predicates are evaluated lazily when first queried, not eagerly materialized
- Stratum tracking: knows which stratum each fact belongs to, enabling targeted re-evaluation

**MangleCP lesson:** This is the core server evaluation architecture for MangleCP. Delta updates (only re-evaluate what changed) + snapshot branches (try before commit) = efficient, safe intent evaluation. MangleCP servers should implement differential evaluation rather than full re-evaluation on every request.

### Proof Tree Tracer (`internal/mangle/proof_tree.go`, 483 lines)

Post-hoc derivation reconstruction:

- `DerivationNode` trees with EDB/IDB classification at each node
- Shows exactly which base facts and which rules produced each derived fact
- Materializes as `derivation_trace` and `proof_tree_node` facts
- ASCII and JSON rendering for debugging/display

Example output:
```
permitted("write", "src/main.go", "") 
├── safe_action("write", "src/main.go", "")
│   └── in_workspace("src/main.go") [EDB]
└── !dangerous_content("write", "src/main.go", "")
    └── (negation: no matching dangerous_content facts)
```

**MangleCP lesson:** The `proof_hints` field in MangleCP's `evaluate` response should carry derivation traces in this format. This enables clients to understand **why** a particular set of macro-tools was returned — essential for debugging, auditing, and trust.

### Mangle Synthesis Compiler (`internal/mangle/synth/compile.go`, 425 lines)

Converts structured JSON specifications into validated Mangle AST:

```json
{
  "clauses": [{
    "head": {"predicate": "safe_action", "args": [
      {"type": "string", "value": "read"},
      {"type": "variable", "name": "Target"},
      {"type": "wildcard"}
    ]},
    "body": [{"predicate": "file_exists", "args": [
      {"type": "variable", "name": "Target"}
    ]}]
  }]
}
```

Compiles to:
```mangle
safe_action("read", Target, _) :- file_exists(Target).
```

Full Mangle coverage: packages, declarations, clauses, transforms, all term types (constants, variables, wildcards, lists, maps, structs). Parse+Analyze validation ensures the output is semantically valid, not just syntactically correct.

**MangleCP lesson:** This is a candidate wire-format encoder for MangleCP. Instead of sending raw Mangle text over the wire (which requires the client to know Mangle syntax), MangleCP could use structured JSON fact format — typed, validatable, language-agnostic. The synthesis compiler proves this JSON-to-Mangle compilation is reliable.

### Feedback Loop (`internal/mangle/feedback/loop.go`, 466 lines)

4-phase validation for LLM-generated Mangle code:

1. **Pre-validate**: syntax check, safety scan (no forbidden predicates)
2. **Auto-repair**: fix common errors (missing quotes, wrong arity, type mismatches)
3. **Sandbox compile**: compile in isolated environment, check for evaluation errors
4. **Schema validate**: verify all predicates match declared schemas

Budget-controlled retries: each repair attempt costs from a retry budget. Dual output protocol — accepts both raw Mangle rules and MangleSynth JSON specifications. JIT predicate selection sends only relevant schema predicates to the LLM for token efficiency.

**MangleCP lesson:** MangleCP servers accepting client-pushed facts or rules should implement this validation pipeline. The protocol should define error responses for each validation stage, so clients know exactly what failed and how to fix it. Budget semantics (limited retries, cost tracking) could be a protocol-level concept.

---

## Cross-Cutting Patterns for MangleCP

### Pattern Summary Table

| Pattern | codeNERD Source | MangleCP Protocol Implication |
|---------|-----------------|-------------------------------|
| Hybrid logic+vector scoring | `compiler.go` + `policy_mcp.mg` | Server evaluation can combine Datalog rules with embedding similarity — protocol should support both |
| Three-tier progressive disclosure | `renderer.go` + `jit_compiler.mg` | `macro_tools` response should carry `disclosure_level` per tool |
| 6-phase atom selection | `jit_compiler.mg` | Skeleton → Exclusion → Flesh → Conflict → Dependency → Final = MangleCP's evaluation pipeline |
| Intent routing pipeline | `intent_routing.mg` | Verb → action type → persona → tools → context priorities → strategy |
| Spreading activation | `activation.mg` | Dynamic attention window based on fact relevance scoring |
| Fact-driven capability negotiation | `ouroboros.go` (`tool_registered`, `has_capability`) | Wire format for capability advertisement in manifest |
| COW snapshot simulation | `differential.go` | "What-if" evaluation without state commitment |
| Proof trees as response metadata | `proof_tree.go` | `proof_hints` field for explainability |
| Structured JSON → Mangle compilation | `synth/compile.go` | Wire-format: typed JSON instead of raw Mangle strings |
| Budget-controlled validation | `feedback/loop.go` | Rate-limiting and cost semantics for the protocol |
| LLM-powered tool analysis | `analyzer.go` | Clients can use LLMs to extract semantic metadata from server-provided tool schemas |
| Usage-weighted selection | `store.go` (usage_count, success_rate, avg_latency) | Tool selection should factor in historical performance |

---

## Things to Improve On

### 1. LLM Dependency for Tool Analysis

The `ToolAnalyzer` (`analyzer.go`) depends on an LLM to extract semantic metadata from tool schemas. This creates a bootstrapping problem: you need an LLM to analyze tools, but you need tools to use an LLM.

**MangleCP improvement:** MangleCP servers should expose structured capability metadata in their manifests — categories, domains, shard affinities — so clients can perform tool selection without needing a separate LLM call for metadata extraction.

### 2. Affinity Scores Are Opaque

Shard affinities (0-100 per shard type) are extracted by the LLM but have no standardized semantics. A score of 60 from one server might mean something different from 60 on another.

**MangleCP improvement:** MangleCP should define standardized capability categories and relevance semantics. If the protocol specifies what "coding", "testing", "debugging" capabilities mean, scores become comparable across servers.

### 3. Ouroboros Complexity

The 1,527-line Ouroboros system is powerful but extremely complex. The 4-phase state machine with COW snapshots, adversarial testing, Go cross-compilation, and hot-reload is beyond what most MangleCP implementations would need.

**MangleCP improvement:** MangleCP should offer the capability advertisement pattern (fact-driven, `tool_registered`/`has_capability`) without requiring the full autopoietic synthesis loop. Dynamic capability updates should be a protocol feature; how the server creates new capabilities is an implementation detail.

### 4. No Temporal Reasoning in Tool Selection

`policy_mcp.mg` has no temporal rules. Tool usage history (count, success rate, latency) is tracked in SQLite but not available as temporal Mangle facts. There is no concept of "tools that worked well in the last hour" vs. "tools that have been unreliable recently."

**MangleCP improvement:** DatalogMTL would enable temporal tool selection:
```mangle
recently_reliable(ToolID) :-
    tool_success(ToolID) <-[3600000],
    !tool_failure(ToolID) <-[1800000].
```

### 5. Tight Coupling Between JIT Compilers

The JIT Tool Compiler and JIT Prompt Compiler are separate systems with separate evaluation pipelines, but they reason about the same context. There is no shared evaluation — each runs its own Mangle evaluation cycle independently.

**MangleCP improvement:** MangleCP's single `evaluate` request should trigger ONE evaluation cycle that produces both the tool selection and the context assembly in a single derivation pass. This is more efficient and ensures consistency between the tools returned and the context provided.

---

## Architecture Decisions to Question

| codeNERD Pattern | Worth Keeping? | Why |
|---|---|---|
| Hybrid logic+vector scoring (70/30) | **Yes — formalize** | Combines semantic understanding with structural reasoning |
| Three-tier rendering (full/condensed/minimal) | **Yes — adopt** | Token-efficient progressive disclosure |
| 6-phase selection pipeline | **Yes — canonical** | This IS MangleCP's evaluation model |
| Intent routing via Mangle rules | **Yes — adopt** | Structured intent → action derivation |
| Spreading activation for context | **Yes — consider** | Dynamic attention is powerful but adds complexity |
| Ouroboros autopoiesis | **Simplify** | Capability advertisement yes; full synthesis is implementation detail |
| LLM-dependent tool analysis | **No — replace** | Server manifests should carry structured metadata |
| Separate JIT compilers | **No — unify** | Single evaluation cycle for tool selection + context assembly |
| COW snapshot simulation | **Yes — for stateful servers** | Essential for "what-if" evaluation |
| Proof tree explainability | **Yes — optional** | `proof_hints` as opt-in response metadata |
| Grammar-constrained decoding | **Yes — for fact validation** | Wire-format validation at the protocol boundary |
| Feedback loop with budget | **Yes — for rule submission** | `submit_rule` message type with validation semantics |

---

## Summary

codeNERD's JIT compilation, MCP client integration, and autopoiesis systems collectively demonstrate that **Mangle can govern the full lifecycle of AI tool interaction** — from intent interpretation (`intent_routing.mg`) through tool selection (`policy_mcp.mg`, `compiler.go`) to context assembly (`jit_compiler.mg`) to capability synthesis (`ouroboros.go`). The 6-phase selection pipeline in `jit_compiler.mg` is the closest existing reference implementation for MangleCP's intent evaluation model. The hybrid logic+vector scoring in `policy_mcp.mg` shows how Datalog rules and embedding similarity can complement each other for relevance ranking. The Ouroboros loop proves that Mangle-governed state machines can manage transactional capability creation with safety guarantees via COW snapshots and adversarial testing.

The most important takeaway for MangleCP: **tool selection, context assembly, and capability negotiation should happen in a SINGLE Mangle evaluation cycle**, not in separate systems with separate evaluation passes. codeNERD's architecture, while powerful, suffers from fragmentation across JIT compilers, MCP client managers, and autopoiesis loops. MangleCP has the opportunity to unify these concerns in a single protocol evaluation model — one intent request triggers one evaluation that produces a complete, context-validated, token-efficient response.
