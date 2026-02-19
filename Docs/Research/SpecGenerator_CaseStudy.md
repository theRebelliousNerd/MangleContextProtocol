# Case Study: symbiogen-spec-generator

> Source-verified lessons from the symbiogen-spec-generator Claude Code plugin for MangleCP protocol design.
> Based on actual Go source code, .mg schema, skill files, agent definitions, and command pipeline.

---

## Overview

symbiogen-spec-generator is a Claude Code MCP server plugin that scans a live codebase and ArangoDB database, builds a Mangle knowledge base of cross-system relationships, and uses deductive rules to reason about data surfaces, duplication risks, component reuse opportunities, and extension candidates. It orchestrates a 7-stage spec generation pipeline through subagents (philosophical-aligner, requirements-interrogator, spec-writer, data-sourcing-analyst, ingestion-layer-analyst).

Where spec-implementer consumes specs and reasons about **workflow state**, spec-generator produces specs and reasons about **codebase topology**. This is the more ambitious Mangle use case: deductive rules operate over a **live representation of an entire software system**, deriving cross-system insights that no single grep or search could find.

---

## Source-Verified Architecture

### Actual Codebase Stats

| Component | File(s) | Lines | Key Insight |
|-----------|---------|-------|-------------|
| Mangle Engine | `mcp-server/internal/mangle/engine.go` | 516 | `SimpleInMemoryStore`, provenance tracking, SQLite persistence |
| Schema | `mcp-server/data/mangle-rules/symbiogen.mg` | 247 | 15 base predicates, 16 relationship predicates, 20+ deductive rules |
| MCP Server | `mcp-server/internal/mcp/server.go` | 1,005 | 21 tools + 2 resource templates |
| Codebase Scanner | `mcp-server/internal/scanner/scanner.go` | 526 | Multi-language entity extraction |
| Scanner Index | `mcp-server/internal/scanner/index.go` | 743 | SHA256 change detection, bulk indexing |
| ArangoDB Inspector | `mcp-server/internal/arango/inspector.go` | 194 | Live DB introspection via HTTP REST |
| SQLite Store | `mcp-server/internal/store/*.go` | ~700 | Entities, facts, files, sources, arango stats |
| Config | `mcp-server/internal/config/config.go` | 160 | YAML + env vars + path auto-detection |
| Skills | `skills/*/SKILL.md` | 478 | feature-alignment (111), spec-templates (366) |
| Subagents | `agents/*.md` | ~1,130 | philosophical-aligner (281), spec-writer (345), data-sourcing-analyst (211), ingestion-layer-analyst (291) |
| Command | `commands/spec.md` | 521 | 7-stage sequential pipeline |

**Total source lines: ~6,500+**

### Actual Tool Count from `server.go`

The `registerTools()` method registers **21 MCP tools** plus **2 resource templates**:

**Search tools (9):**
- `map_feature_surface` -- composite cross-system surface mapper
- `search_agents`, `search_deepagents`, `search_graphs`
- `search_collections`, `search_models`, `search_frontend_components`
- `search_sdk_services`, `search_tools`, `search_routers`

**Subsystem tools (2):**
- `get_subsystem_docs` -- reads `deepClaude.md` per subsystem
- `grep_codebase` -- regex search across entire codebase

**Mangle tools (3):**
- `query_cross_system` -- arbitrary Mangle queries with limit
- `push_facts` -- JSON array of facts into the deductive DB
- `get_mangle_status` -- fact count, predicates, schema status

**Index tools (3):**
- `reindex` -- force full codebase rescan with SHA256 change detection
- `index_status` -- entity counts, fact counts, last scan time
- `clear_index` -- destructive wipe (requires `confirm=true`)

**ArangoDB tools (2):**
- `arango_inspect` -- live collection/graph sync with cross-reference
- `arango_status` -- health check, version, collection counts

**Resource templates (2):**
- `symbiogen://framework-docs/{name}` -- LangGraph, LangChain, DeepAgents docs
- `symbiogen://subsystem/{name}` -- per-subsystem deepClaude docs

---

## Key Patterns Worth Adopting

### 1. Deductive Cross-System Reasoning

The `.mg` schema's most powerful pattern is **multi-hop deductive reasoning** over cross-system relationships. Example: determining where a data entity surfaces across the UI:

```mangle
derived_surface(Entity, Page) :-
    data_entity(Entity, _),
    data_served_by(Entity, Endpoint),
    sdk_calls_endpoint(SDK, Endpoint),
    component_uses_sdk(Component, SDK),
    component_on_page(Component, Page).
```

This rule chains through 5 relationship hops: entity → endpoint → SDK service → component → page. No single search query could derive this. The deductive engine traces the data flow path automatically.

**MangleCP lesson:** This is the **definitive argument** for Mangle over static tool lists. An MCP server listing `search_agents` and `search_components` as separate tools forces the client to manually chain searches. A MangleCP server receiving the intent "where does this data appear?" can run `derived_surface(Entity, Page)` and return the complete answer in one round trip.

### 2. Ghost and Orphan Collection Detection

The schema uses negation-as-failure to detect infrastructure drift:

```mangle
# Referenced in code but doesn't exist in ArangoDB
ghost_collection(Name) :- has_code_collection(Name), !has_arango_collection(Name).

# Exists in ArangoDB but no code references it
orphan_arango_collection(Name, Type, Count) :-
    arango_collection(Name, Type, Count), !has_code_collection(Name).
```

These rules compare facts from **two different sources** (codebase scan vs. live ArangoDB) and derive discrepancies. This is infrastructure-as-facts reasoning.

**MangleCP lesson:** MangleCP servers can ingest facts from multiple systems and use rules to detect cross-system inconsistencies. The `facts_profile` in the manifest should support declaring multi-source fact origins so clients know what the server can cross-reference.

### 3. Component Reuse Candidate Detection

The most sophisticated deductive rule finds components that **should** exist on a page but don't:

```mangle
component_reuse_candidate(Comp, CurrentPage, CandidatePage) :-
    component_on_page(Comp, CurrentPage),
    component_uses_sdk(Comp, SDK),
    component_uses_sdk(OtherComp, SDK),
    component_on_page(OtherComp, CandidatePage),
    CurrentPage != CandidatePage,
    !has_component_on_page(Comp, CandidatePage).
```

Logic: if component A uses SDK service S, and a different component B on a different page also uses S, then component A is a candidate for reuse on that other page. With negation, it only fires when A is NOT already there.

**MangleCP lesson:** This is a proactive recommendation derived from structural analysis. MangleCP's `next.suggested_intents` in invoke responses could carry similar rule-derived suggestions: "based on the current state, you should also consider..."

### 4. Multi-Source Fact Generation Pipeline

The server generates Mangle facts from **three sources**, each with explicit provenance:

**Source 1: Codebase scan** (`server.go:651-731`, `generateFactsFromIndex()`):
```go
s.engine.SuspendEval()
defer s.engine.ResumeEval()

for entityType, predicate := range types {
    entities, err := s.store.ListEntitiesByType(entityType)
    for _, e := range entities {
        s.engine.PushFactWithSource(predicate, "codebase_scan", nil, args...)
    }
}
```

Generates 9 entity type predicates + 5 relationship predicates from indexed codebase entities.

**Source 2: ArangoDB live introspection** (`server.go:733-778`, `generateFactsFromArango()`):
```go
s.engine.PushFactWithSource("arango_collection", "arango_live", nil, c.Name, c.CollectionType, c.DocCount)
s.engine.PushFactWithSource("collection_in_subsystem", "arango_live", nil, c.Name, sub)
s.engine.PushFactWithSource("arango_graph", "arango_live", nil, g.Name, ed.Collection, from, to)
```

**Source 3: Client-pushed facts** (`handlePushFacts`, `server.go:463-484`):
```go
for _, f := range facts {
    s.engine.PushFact(f.Predicate, f.Args...)
}
```

The engine tracks provenance via `PushFactWithSource(predicate, sourceType, sourceID, args...)`, persisting the source type ("codebase_scan", "arango_live", "manual") to SQLite alongside the fact.

**MangleCP lesson:** Facts in MangleCP should carry provenance metadata. The `Fact` object in the protocol could include an optional `source` field. This enables servers to distinguish client-asserted facts from server-derived facts and to prioritize or filter by source.

### 5. `map_feature_surface` -- The Composite Tool Pattern

The most powerful search tool is a **composite** that replaces ~10 individual searches:

```go
func (s *Server) handleMapFeatureSurface(ctx context.Context, req mcp.CallToolRequest) {
    entityTypes := []string{"agent", "deepagent", "graph", "model", "router",
                            "tool", "component", "sdk_service", "collection"}
    // Search ALL entity types for the keyword
    for _, et := range entityTypes {
        entities, _ := s.store.SearchEntities(et, keyword)
        // ...
    }
    // Also search relationship types
    // graph_composes_agent, component_uses_sdk, sdk_calls_endpoint,
    // component_on_page, page_uses_sdk
}
```

One tool call returns entities across 9 types + 5 relationship types, with a summary count and full details. This is explicitly the MangleCP macro-tool pattern: **one high-level call replacing many atomic searches**.

**MangleCP lesson:** `map_feature_surface` is a manual implementation of what MangleCP automates. In MangleCP, the server would evaluate the intent against rules and synthesize this composite response dynamically. The fact that it had to be hand-coded here validates MangleCP's design.

### 6. Rule Status Self-Documentation

The `.mg` schema includes a remarkable self-assessment section at the bottom (`lines 222-246`):

```mangle
# WORKING (fire with current scanner-populated facts):
#   agent_candidate       -- agent existence facts
#   ghost_collection      -- collection + arango_collection facts
#   component_reuse_candidate -- component_on_page + component_uses_sdk facts
#
# PARTIALLY ALIVE (some inputs populated):
#   duplication_risk      -- needs: agent_uses_collection
#   derived_surface       -- needs: data_entity, data_served_by
#
# DEAD (relationship inputs never populated by scanner):
#   pipeline_dependency   -- needs: pipeline_layer
```

This documents which rules actually fire based on which facts are populated by the scanner.

**MangleCP lesson:** MangleCP servers should be able to report rule coverage -- which rules are live vs. dormant based on current fact state. This could be part of `proof_hints` or a dedicated diagnostic endpoint.

---

## The 7-Stage Pipeline and Subagent Architecture

### Pipeline Flow (`commands/spec.md`)

```
Prior-Spec Discovery (registry-based or legacy fallback)
    |
Stage 1: Philosophical Alignment (philosophical-aligner, model: opus)
    |
Stage 2: Requirements Interrogation (requirements-interrogator, max 3 rounds)
    |  → If Reject verdict (0-11): PIPELINE STOPS
    |
Stage 3: Spec Generation (spec-writer, model: opus)
    |
Stage 4: Data Sourcing Analysis (data-sourcing-analyst)
    |
Stage 5: Ingestion Pipeline Analysis (ingestion-layer-analyst)
    |
Stage 6: Cross-System Journal + Registry Update (orchestrator directly)
    |
Stage 7: Summary
```

### Subagent-MCP Tool Access

Subagents use MCP tools **extensively** to ground their reasoning in codebase reality:

The `/spec` command instructs subagents to:
1. Call `map_feature_surface(keyword)` **FIRST** for a compact cross-system view
2. Call `query_cross_system` for deductive reasoning:
   - `pages_share_component(P1, P2, Comp)` -- sharing detection
   - `component_reuse_candidate(Comp, P1, P2)` -- reuse opportunities
   - `pages_share_sdk(P1, P2, SDK)` -- SDK dependency sharing
   - `shared_data(C1, C2, Endpoint)` -- data flow sharing
3. Use `search_*` tools for targeted follow-up

### Prior-Spec Discovery

Before generating a new spec, the pipeline searches for existing specs to avoid duplicate recommendations. Two modes:

**Registry-based** (preferred): Reads `_spec-registry.md`, matches keywords against a domain taxonomy, scores relevance in 3 tiers (HIGH/MEDIUM/LOW), reads spec files proportional to relevance.

**Legacy fallback**: Globs for `docs/specs/*/README.md`, reads last 5 specs by date.

This produces a `PRIOR_SPEC_CONTEXT` block that is injected into subagent prompts.

**MangleCP lesson:** Prior context discovery is another form of intent-scoped context selection. A MangleCP server could store facts from previous intent evaluations and use them to enrich future responses -- not as memory (that's out of scope), but as pre-loaded facts the client can optionally include.

---

## Things to Improve On

### 1. No Temporal Store

Same as BrowserNERD and spec-implementer: `factstore.NewSimpleInMemoryStore()`. No temporal operators used despite the schema modeling time-varying infrastructure state (collections grow, agents get added).

### 2. `map_feature_surface` Is Hand-Coded, Not Rule-Derived

The composite tool manually iterates over entity types and relationship types in Go code (`server.go:780-934`). The Mangle engine could derive which entities are relevant to a keyword via rules, but instead the search is done at the SQLite layer with string matching.

**MangleCP improvement:** A MangleCP server would express "entities relevant to keyword K" as a Mangle query, letting rules determine relevance rather than hardcoding entity types.

### 3. Scanner-Rule Gap

The `.mg` schema self-documents that several rules are "DEAD" or "PARTIALLY ALIVE" because the scanner doesn't populate all the relationship facts they need:

- `pipeline_dependency` needs `pipeline_layer` facts (never populated)
- `duplication_risk` needs `agent_uses_collection` facts (partially)
- `derived_surface` needs `data_entity`, `data_served_by` facts (partially)

**MangleCP consideration:** MangleCP servers need a way to communicate fact completeness. If rules exist but the facts they depend on aren't populated, the server should report this in the manifest or in query responses, so clients know the reasoning is limited.

### 4. No Evaluation Safety Limits

Same as the other projects. No constraints on `EvalProgram`.

### 5. 21 Tools Is Still a Large Surface

Despite being more focused than BrowserNERD's 39 tools, 21 tools is still a significant cognitive load for an AI client. The progressive disclosure problem persists: the client must understand which search tool to use for which entity type.

**MangleCP improvement:** MangleCP eliminates this by not exposing search tools at all. The client sends an intent; the server returns what's relevant. The 9 separate `search_*` tools would collapse into a single intent evaluation.

---

## Architecture Decisions to Question

| Pattern | Worth Keeping? | Why |
|---|---|---|
| `SimpleInMemoryStore` | **No** | Infrastructure state is inherently temporal |
| Multi-source fact generation | **Yes** | Codebase + live DB cross-referencing is powerful |
| Provenance tracking on facts | **Yes** | Essential for trust and debugging |
| `map_feature_surface` composite | **Evolved** | MangleCP automates this via intent evaluation |
| Ghost/orphan collection detection | **Yes** | Cross-system inconsistency detection via negation |
| Component reuse candidate rules | **Yes** | Proactive recommendations from structural analysis |
| Rule status self-documentation | **Adopt** | Servers should report which rules are live |
| Prior-spec discovery | **Yes** | Contextual enrichment from prior evaluations |
| 7-stage subagent pipeline | **Out of scope** | Orchestration belongs in clients, not in MangleCP |

---

## Summary

symbiogen-spec-generator is the most ambitious Mangle-powered MCP server in this study series. It builds a deductive knowledge base from **two live data sources** (codebase scan + ArangoDB introspection), reasons about cross-system relationships via multi-hop rules, detects infrastructure drift via negation, and identifies proactive optimization opportunities (component reuse, shared data). The `derived_surface` rule -- chaining through 5 relationship hops to determine where data appears in the UI -- is the strongest single argument for why MangleCP uses Mangle over static tool lists. The multi-source fact pipeline with provenance tracking shows how MangleCP's fact encoding should work in practice. The gap between the schema's aspirations (20+ rules) and the scanner's reality (many rules dormant) highlights the need for MangleCP servers to communicate fact completeness and rule coverage to clients.
