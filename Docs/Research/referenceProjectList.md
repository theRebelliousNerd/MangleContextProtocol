# MangleCP — Reference Project List

Projects studied for architecture, patterns, and lessons applicable to MangleCP protocol design and implementation.

---

## 1. BrowserNERD

| | |
|---|---|
| **Source** | `C:\CodeProjects\SybioGenv3\symbiogenBackEndV3\dev_tools\BrowserNERD` |
| **Public Repo** | [github.com/theRebelliousNerd/browserNerd](https://github.com/theRebelliousNerd/browserNerd) |
| **Language** | Go |
| **Stack** | Rod (CDP) · Mangle (logic engine) · mcp-go (MCP protocol) |
| **Case Study** | [BrowserNERD_CaseStudy.md](BrowserNERD_CaseStudy.md) |

**What it is:** A token-efficient browser automation MCP server that uses Mangle for declarative reasoning over browser state. 39 MCP tools (source-verified, not 37 as README claims), 990-line `.mg` schema with 13 vectors and ~100+ predicates, progressive disclosure via `browser-observe`/`browser-act`/`browser-reason`/`browser-mangle`.

**Why it matters for MangleCP:**

- First real-world system that embeds Mangle for runtime fact evaluation and rule-based tool synthesis
- Demonstrates the `execute-plan` macro-execution pattern that MangleCP's macro-tools generalize
- Progressive disclosure model is the direct ancestor of MangleCP's intent-scoped tool synthesis
- Shows practical Mangle integration patterns: fact stores, predicate schemas, custom rule submission, temporal queries
- Proves the token-efficiency thesis: 50-100x fewer tokens vs. raw HTML approaches

---

## 2. spec-implementer

| | |
|---|---|
| **Source** | `C:\CodeProjects\SybioGenv3\symbiogenBackEndV3\.claude\plugins\spec-implementer` |
| **Language** | Go |
| **Stack** | Mangle (logic engine) · mcp-go (MCP protocol) · SQLite (persistence) |
| **Case Study** | [SpecImplementer_CaseStudy.md](SpecImplementer_CaseStudy.md) |

**What it is:** A Claude Code MCP server plugin that orchestrates feature spec implementation through Mangle-powered task classification, dependency sequencing, agent assignment, and context-scoped prompt generation. 17 MCP tools, 474-line .mg schema with 30 predicates and 28 rules, 3 subagents (complex-implementer, mechanical-implementer, unit-test-grinder), 2 skills (execution-strategy, spec-parsing).

**Why it matters for MangleCP:**

- The `needs_file` predicate is the closest existing implementation to MangleCP's intent-scoped context injection: Mangle rules prove which context files a subagent needs, and the server includes only those
- Skill-Mangle parallel encoding pattern: human-readable skills and machine-queryable rules encode the same domain logic from different angles, showing how MangleCP skills and rules should co-exist
- `agent_for_step` rules demonstrate Mangle-driven capability selection: the server computes which agent type to assign based on task complexity facts
- Self-contained prompt generation via `get_agent_prompt` is the macro-tool pattern in action: one call returns everything needed, zero conversation history required
- Reactive fact-driven workflow: progress fact mutations trigger rule re-evaluation, dynamically updating which phases are ready
- Commented-out DatalogMTL rules (`stale_phase`) show explicit desire for temporal reasoning that MangleCP enables

---

## 3. symbiogen-spec-generator

| | |
|---|---|
| **Source** | `C:\CodeProjects\SybioGenv3\symbiogenBackEndV3\.claude\plugins\symbiogen-spec-generator` |
| **Language** | Go |
| **Stack** | Mangle (logic engine) · mcp-go (MCP protocol) · SQLite (persistence) · ArangoDB (live introspection) |
| **Case Study** | [SpecGenerator_CaseStudy.md](SpecGenerator_CaseStudy.md) |

**What it is:** A Claude Code MCP server plugin that scans a live codebase and ArangoDB database to build a Mangle knowledge base of cross-system relationships, then uses deductive rules for anti-duplication reasoning during spec generation. 21 MCP tools + 2 resource templates, 247-line .mg schema with 31 predicates and 20+ rules, 4 subagents, 2 skills (feature-alignment, spec-templates), 7-stage generation pipeline.

**Why it matters for MangleCP:**

- The `derived_surface` rule (5-hop cross-system data flow tracing: entity → endpoint → SDK → component → page) is the strongest single argument for why MangleCP uses Mangle over static tool lists
- Multi-source fact pipeline with provenance tracking: facts from codebase scans, live ArangoDB, and client pushes are distinguished by source type, showing how MangleCP facts should carry provenance
- Ghost/orphan collection detection via negation-as-failure demonstrates cross-system inconsistency detection by comparing facts from two independent sources
- `component_reuse_candidate` is a proactive recommendation rule: it finds components that SHOULD exist on a page but don't, anticipating MangleCP's `next.suggested_intents`
- `map_feature_surface` is a hand-coded composite tool that does what MangleCP automates: one call replacing ~10 atomic searches, validating the macro-tool paradigm
- Rule status self-documentation (WORKING / PARTIALLY ALIVE / DEAD) shows the need for MangleCP servers to report fact completeness and rule coverage

---

## 4. codeNERD

| | |
|---|---|
| **Source** | `C:\CodeProjects\codeNERD` |
| **Language** | Go |
| **Stack** | Mangle (logic engine) · MCP client (mcp-go) · SQLite + sqlite-vec · Go cross-compiler (Ouroboros) |
| **Case Studies** | [Part 1: Kernel Architecture](CodeNERD_CaseStudy_Part1_Kernel.md) · [Part 2: JIT, MCP, Autopoiesis](CodeNERD_CaseStudy_Part2_JIT_MCP.md) |

**What it is:** A ~220K LOC Go CLI coding agent with the deepest Mangle integration of any system studied. Unlike the other reference projects (which are MCP servers), codeNERD is an **MCP client** that connects to external MCP servers while using Mangle as its internal reasoning kernel. 907 Go files, 274 `.mg` files, 35 static tools + Ouroboros-generated dynamic tools. The Mangle kernel governs every action: file writes, bash commands, MCP tool calls, and subagent spawns are all proposed as Mangle facts, evaluated against constitutional safety rules, gated by permission predicates, and routed through a virtual predicate dispatch system.

**Why it matters for MangleCP:**

- The "Hollow Kernel" pattern — Go code as evaluation harness, all reasoning in `.mg` files — is the strongest validation of MangleCP's server architecture model
- Fresh-store evaluation per cycle ensures deterministic, stateless reasoning — the correct model for MangleCP's per-intent evaluation
- Stratified trust ordering (schema > policy > learned) provides a formal security hierarchy that MangleCP should adopt for rule source trust levels
- Three-tier fact categorization (persistent/ephemeral/derived) maps directly to MangleCP's EDB/session-scoped/IDB distinction
- The JIT Prompt Compiler's 6-phase selection pipeline (skeleton → exclusion → flesh → conflict → dependency → final) IS MangleCP's intent evaluation model
- Hybrid logic+vector scoring (70% Mangle rules + 30% embedding similarity) in `policy_mcp.mg` demonstrates combined Datalog+ML tool selection
- The Ouroboros autopoiesis loop proves Mangle-governed transactional state machines can manage runtime capability synthesis with COW snapshot safety
- Dual-layer constitutional safety (unforgeable Go checks + configurable Mangle policy) is the reference model for MangleCP's security architecture
- External predicates (10 FFI bridges to knowledge graph, embeddings, traces) show how MangleCP servers can connect Mangle evaluation to external data sources
- Transaction API (atomic retract+assert with single rebuild) defines the state mutation model for MangleCP's `state_delta` responses
- Proof tree tracer enables explainability in MangleCP's `proof_hints` response field
- Structured JSON → Mangle compilation (`synth/compile.go`) is a candidate wire-format encoder — typed JSON instead of raw Mangle strings
- Zero use of DatalogMTL/TemporalStore despite massive temporal semantics — strongest evidence that MangleCP should make temporal features optional rather than mandatory
