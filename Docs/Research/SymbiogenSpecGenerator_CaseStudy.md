# Case Study: SymbioGen Spec Generator Plugin

> Source-verified analysis of the `symbiogen-spec-generator` plugin, analyzing how it uses Mangle to provide architectural intelligence and cross-system reasoning.

---

## Overview

The `symbiogen-spec-generator` plugin acts as a **"Living Architecture Engine"**. Unlike `spec-implementer` which drives *process*, this plugin drives *knowledge*. It indexes the entire codebase (frontend, backend, AI agents, database) into a local Mangle instance to answer complex questions about system surface area, dependency chains, and architectural violations.

Its primary goal is to support the "Extend Before Create" mandate by proving whether a requested feature can be built by composing existing components.

---

## Source-Verified Architecture

### 1. The Knowledge Server (The Cortex)

The heart of the system is the **SymbioGen Knowledge Server** (`mcp-server/cmd/server/main.go`). It is a standalone MCP server that:

1. **Scans:** Walks the filesystem to identify "Entities" (Agents, Components, Routers, Models).
2. **Indexes:** Persists these entities to SQLite.
3. **Connects:** Talks to a live ArangoDB instance to map the actual data schema.
4. **Deduces:** Loads all this into Mangle to run logic rules.

### 2. The God Tool: `map_feature_surface`

This is the standout pattern. Instead of making the Agent call 10 different search tools (`search_files`, `grep`, `find_tables`), the server exposes a single high-level cognitive tool:

**Tool:** `map_feature_surface(keyword="bid")`
**Internal Logic:**

1. Finds all entities matching "bid" (SQLite FTS).
2. Traces relationships (Graph traversal).
3. Queries Mangle for derived insights.
4. Returns a **Composite object** containing:
    * `agents`: Agents that handle bids.
    * `collections`: DB tables storing bids.
    * `components`: UI widgets displaying bids.
    * `sdk_services`: API clients fetching bids.
    * `gaps`: "Page X needs bid data but has no component for it." (Derived by Mangle)

### 3. Mangle Schema as Architectural Linter

The `symbiogen.mg` file (247 lines) defines the architectural rules of the platform. It doesn't just describe what *is*, it reasons about what *should be*.

**Key Patterns Identified:**

* **The "Ghost Collection" Detector:**

    ```mangle
    ghost_collection(Name) :-
        has_code_collection(Name),
        !has_arango_collection(Name).
    ```

    *Insight:* Finds code that references DB tables that don't exist in production.

* **The "Component Reuse" Recommender:**

    ```mangle
    component_reuse_candidate(Comp, CurrentPage, CandidatePage) :-
        component_on_page(Comp, CurrentPage),
        component_uses_sdk(Comp, SDK),
        component_uses_sdk(OtherComp, SDK),       // Both components use same data source
        component_on_page(OtherComp, CandidatePage),
        !has_component_on_page(Comp, CandidatePage).
    ```

    *Insight:* "Page A and Page B both fetch User data. Page A has a UserCard. Page B doesn't. You should put UserCard on Page B."

* **Duplication Risk Analysis:**

    ```mangle
    duplication_risk(A1, A2, Coll) :-
        agent_uses_collection(A1, Coll),
        agent_uses_collection(A2, Coll),
        A1 != A2,
        !is_composed(A1),
        !is_composed(A2).
    ```

    *Insight:* Two raw agents are touching the same table. They should probably be composed into a graph or check with each other.

---

## The Interplay Loop

1. **Ingest:** The scanner runs (triggered by file changes or `reindex` tool). It pushes base facts:
    * `component_on_page("BidPanel", "/bids")`
    * `component_uses_sdk("BidPanel", "BidService")`
    * `sdk_calls_endpoint("BidService", "/api/bids")`

2. **Deduce:** Mangle runs the forward-chaining rules.
    * It infers `derived_surface("BidData", "/bids")`.

3. **Query:** User asks "Where do we use Bids?".
    * Agent calls `map_feature_surface("bid")`.
    * Server returns the deduced map.

4. **Refine:** User asks "Can I reuse the BidPanel on the Dashboard?".
    * Agent calls `query_cross_system("component_reuse_candidate(C, _, '/dashboard').")`.
    * Mangle confirms if the dependencies (SDK services) are available on the dashboard.

---

## Implementation Insights for MangleCP

1. **Codebase-as-Database:** This proves Mangle is excellent for static analysis. MangleCP should support "scanners" that feed facts from AST parsing.
2. **Hybrid Querying:** The server combines SQLite (fast text search) with Mangle (logic). Mangle doesn't need to do fuzzy text matching; it delegates that to the host.
3. **Live Infrastructure Verification:** The `arango_inspect` tool feeds "live" facts (`arango_collection`) which are cross-referenced with "code" facts (`collection`). This allows **Runtime-Static Consistency Checking** via Mangle rules.

## Conclusion

`symbiogen-spec-generator` demonstrates that Mangle can be the **"Brain" of the IDE**. By lifting code structure into a logic programming domain, it allows the Agent to reason about system architecture (reuse, consistency, gaps) in milliseconds, a task that would take a human developer hours of Grep/Find-All-References.
