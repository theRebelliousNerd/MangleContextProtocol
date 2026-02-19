# Case Study: BrowserNERD

> Source-verified lessons from BrowserNERD for MangleCP protocol design.
> Based on actual Go source code inspection, not just README/docs.

---

## Overview

BrowserNERD is a Go-based MCP server for browser automation that embeds Google's Mangle logic engine. It transforms raw browser state (DOM, network, console, React Fiber trees) into structured Mangle facts and uses declarative rules to derive higher-level insights — causal error chains, slow API detection, login success detection — before exposing them to the AI client.

It is the closest existing system to what MangleCP envisions: **a server that uses Mangle reasoning to determine what to expose to the client**.

---

## Source-Verified Architecture

### Actual Codebase Stats

| Component | File(s) | Lines | Key Insight |
|-----------|---------|-------|-------------|
| Mangle Engine | `internal/mangle/engine.go` | 839 | Uses `factstore.SimpleInMemoryStore`, NOT `TemporalStore` |
| Schema | `schemas/browser.mg` | 990 | **13 vectors**, ~100+ predicates, 30+ rules (far beyond README's "70+/25+") |
| Progressive Tools | `internal/mcp/progressive_tools.go` | 3,019 | `BrowserObserveTool`, `BrowserActTool`, `BrowserReasonTool`, `BrowserMangleTool` |
| Server & Registration | `internal/mcp/server.go` | 244 | `progressive_only` config flag controls which tools are registered |
| Session Manager | `internal/browser/session_manager.go` | ~42K bytes | Rod CDP integration, session forking |
| Correlation | `internal/correlation/keys.go` | 157 | OpenTelemetry-aware: traceparent, B3, Cloud Trace, request_id |
| Fact Tools | `internal/mcp/fact_tools.go` | 663 | Push, read, query, submit-rule, evaluate, subscribe, await |
| Navigation | `internal/mcp/navigation_*.go` | 4 files, ~80K | Elements, links, state, JavaScript extraction |
| Automation | `internal/mcp/automation_tools.go` | ~43K | execute-plan, wait-for-condition, interact, fill-form |

### Actual Tool Count from `server.go`

The `registerAllTools()` method reveals the real tool registration:

**Always registered (6 tools):**

- `launch-browser`, `shutdown-browser` (lifecycle)
- `browser-observe`, `browser-act`, `browser-reason`, `browser-mangle` (progressive)

**Conditionally registered when `progressive_only: false` (33 tools):**

- Session: `list-sessions`, `create-session`, `attach-session`, `fork-session`, `reify-react`, `snapshot-dom`
- Facts: `push-facts`, `read-facts`, `query-facts`, `submit-rule`, `query-temporal`, `evaluate-rule`, `subscribe-rule`, `await-fact`, `await-conditions`
- Diagnostics: `get-console-errors`, `get-toast-notifications`
- Navigation: `get-navigation-links`, `get-interactive-elements`, `discover-grids`, `discover-hidden-content`, `interact`, `get-page-state`, `navigate-url`, `press-key`
- Advanced: `screenshot`, `browser-history`, `evaluate-js`, `fill-form`
- Automation: `execute-plan`, `wait-for-condition`, `diagnose-page`, `await-stable-state`

**Total: 39 tools** (not 37 as README claims) — `browser-mangle` and `discover-grids` were added post-README.

---

## Key Patterns Worth Adopting

### 1. Dual-Store Fact Architecture

The engine maintains **two parallel stores** — a critical pattern:

```go
// Mangle core
store       factstore.FactStore    // For rule evaluation via EvalProgram
// Temporal buffer
facts []Fact                       // For time-windowed queries and fallback
index map[string][]int             // O(m) predicate lookup
```

Facts are added to both on every `AddFacts()` call. The Mangle store drives rule evaluation, while the Go buffer handles temporal queries (`QueryTemporal`) and serves as a fallback when the Mangle store can't match due to arity mismatches.

**Lesson for MangleCP:** Servers will likely need this dual-store pattern. MangleCP's `TemporalStore` (DatalogMTL) should handle both roles natively, but the buffer-fallback pattern shows that Mangle's store has real-world matching limitations that need workarounds.

### 2. Incremental Evaluation on Every Fact Batch

Every call to `AddFacts()` triggers `engine.EvalProgram()`:

```go
// Trigger incremental evaluation if schema loaded
if e.schemaLoaded && e.programInfo != nil {
    if err := engine.EvalProgram(e.programInfo, e.store); err != nil {
        return fmt.Errorf("eval program after fact insertion: %w", err)
    }
    e.checkAndNotifyWatchers()
}
```

**Lesson for MangleCP:** Rule evaluation happens continuously, not just on query. MangleCP servers should decide whether evaluation is on-demand (per intent request) or continuous (on fact arrival). Both have tradeoffs — continuous is more responsive but costlier.

### 3. Adaptive Sampling Under Load

The engine implements **5-tier pressure-based sampling**:

| Buffer Fill | Sampling Rate | Effect |
|-------------|---------------|--------|
| < 50% | 1.0 | Accept all |
| 50-70% | 0.8 | Drop 20% low-value |
| 70-85% | 0.5 | Drop 50% low-value |
| 85-95% | 0.2 | Drop 80% low-value |
| > 95% | 0.1 | Emergency: drop 90% |

High-value predicates (`console_event`, `net_request`, `navigation_event`) are **never sampled**. Low-value predicates (`dom_node`, `dom_attr`, `react_prop`) get probabilistically dropped.

**Lesson for MangleCP:** The protocol's `max_facts_created` constraint maps to this pattern. MangleCP should define a standard way for servers to report their sampling/pressure state in responses, so clients know when facts are being dropped.

### 4. Progressive Disclosure with `progressive_only` Mode

`server.go` has a critical config flag:

```go
if !s.cfg.MCP.IsProgressiveOnly() {
    s.registerIndividualTools()
}
```

When `progressive_only: true`, the client sees only **6 tools** instead of 39. The progressive tools (`browser-observe`, `browser-act`, `browser-reason`, `browser-mangle`) internally delegate to the same logic as the individual tools.

**Lesson for MangleCP:** This is a halfway step toward MangleCP's model. BrowserNERD still registers a fixed set of progressive tools; MangleCP goes further by having the server dynamically synthesize the relevant tools per-intent.

### 5. Pub/Sub Watch Mode

The engine supports reactive subscriptions:

```go
type WatchEvent struct {
    Predicate string    `json:"predicate"`
    Facts     []Fact    `json:"facts"`
    Timestamp time.Time `json:"timestamp"`
}
func (e *Engine) Subscribe(predicate string, ch chan WatchEvent) string
```

After every `EvalProgram`, `checkAndNotifyWatchers()` scans subscribed predicates and pushes new derived facts to subscriber channels (non-blocking).

**Lesson for MangleCP:** MangleCP's current design is request/response only. BrowserNERD's watch mode suggests a future extension: server-initiated notifications when derived facts change. This could be a `subscribe` message type in a MangleCP v2.

### 6. The 990-Line Schema as Contract (`browser.mg`)

Far richer than the README suggests. The actual schema has **13 vectors**:

| Vector | Predicates | Purpose |
|--------|-----------|---------|
| 1: React Fiber | `react_component`, `react_prop`, `react_state`, `dom_mapping` | Component tree |
| 2: Flight Recorder | `net_request`, `net_response`, `console_event`, `navigation_event`, etc. | CDP events |
| 3: Session State | `current_url` | URL tracking |
| 4: Test Assertions | `test_passed` | Logic-based assertions |
| 5: Interactive Elements | `interactive`, `user_click`, `user_type`, etc. | Token-efficient refs |
| 5b: Navigation Links | `nav_link`, `nav_area_has_links`, `internal_nav_target` | Link extraction |
| 6: Automation Events | `screenshot_taken`, `plan_executed`, `action` | Batch operation tracking |
| 7: Docker Logs | `docker_log`, `docker_error`, `api_backend_correlation` | Full-stack correlation |
| 8: Element Fingerprints | `element_fingerprint`, `element_alt_selector`, `unreliable_element` | Reliability monitoring |
| 9: Page State | `page_loading`, `page_has_error`, `page_empty` | UI pattern detection |
| 10: Accessibility | `a11y_issue`, `a11y_critical`, `unlabeled_interactive` | A11y audit |
| 11: Form Validation | `form_validation_error`, `form_has_errors`, `missing_required_field` | Form state |
| 12: Interaction Sequences | `repeated_action_on_element`, `click_then_type`, `click_triggered_navigation` | Pattern tracking |
| 13: Test Helpers | (remaining rules) | Assertion patterns |

Includes aggregation rules using `|> do fn:group_by(), let Count = fn:count()` — proving Mangle aggregation works in production.

**Lesson for MangleCP:** The `facts_profile` in the manifest should support referencing a schema this complex. Consider whether MangleCP should allow servers to expose their `.mg` schema as a discoverable resource.

---

## Things to Improve On

### 1. No `TemporalStore` — Timestamps as Plain Data

**This is the biggest finding.** BrowserNERD uses `factstore.NewSimpleInMemoryStore()`, NOT `factstore.NewTemporalStore()`. Timestamps are regular Go `time.Time` fields on the `Fact` struct, treated as integer arguments in Mangle predicates:

```go
// BrowserNERD: timestamp as a regular Mangle argument
net_request(SessionId, Id, Method, Url, InitiatorId, StartTime).
// StartTime is just an int64 millisecond value
```

Temporal queries (`QueryTemporal`) are done in Go code by filtering the buffer, not via Mangle's temporal operators.

**MangleCP improvement:** MangleCP is built on DatalogMTL from day one. The difference:

```mangle
# MangleCP: temporal annotation is first-class
net_request(SessionId, Id, Method, Url, InitiatorId) @[StartTime, EndTime].

# Enables native temporal operators:
recent_failure(S) <-[5m] failed_request(S, _, _, _).
```

This means MangleCP servers can express *"what was true N minutes ago"* and *"what will expire in the next hour"* natively, without Go-level buffer scanning.

### 2. Tool Explosion Remains Even with Progressive Mode

Even in `progressive_only` mode, BrowserNERD still exposes **6 fixed tools**. The progressive tools are mega-tools with complex input schemas (the `BrowserObserveTool.InputSchema()` alone is 83 lines), requiring the client to understand modes, views, filters, and intents.

**MangleCP improvement:** MangleCP servers return **only the macro-tools relevant to the current intent**, with pre-configured input schemas. The client doesn't need to learn a complex meta-API; each macro-tool is purpose-built for the current context.

### 3. No Evaluation Safety Limits

The engine calls `engine.EvalProgram()` with no timeout, no fact-creation limit, and no interval cap. If a user submits a recursive rule via `submit-rule`, it could run indefinitely.

The only safeguard is `AddRule()`'s rollback-on-error:

```go
if err := engine.EvalProgram(e.programInfo, e.store); err != nil {
    e.ruleUnits = prevRules    // Rollback
    e.programInfo = prevProgram
    e.schemaLoaded = prevLoaded
    return fmt.Errorf("eval program after rule insertion: %w", err)
}
```

**MangleCP improvement:** The protocol defines explicit `constraints` (`max_facts_created`, `max_intervals_per_atom`, `max_compute_ms`) in intent requests. Servers MUST enforce these. DatalogMTL's built-in interval coalescing and derivation limits provide native safety that BrowserNERD lacks.

### 4. Negation Workarounds Throughout the Schema

The schema has multiple comments acknowledging Mangle's negation limitations:

```mangle
# Note: Proper negation would need: !api_backend_correlation(_, _, _, Msg, _)
# But Mangle requires stratified negation - track orphans in Go code instead
```

```mangle
# Note: Requires tracking in Go code since Mangle negation is limited
```

Several "derived" predicates (orphan errors, unresolved errors, form-ready) are conceptual but require Go-side implementation because Mangle can't express the negation.

**MangleCP consideration:** MangleCP servers will hit this same limitation. The protocol should acknowledge that some derived facts must be computed in server-side code, not pure Mangle rules. The `proof_hints` field could distinguish rule-derived vs. code-derived facts.

### 5. OpenTelemetry Correlation is a Hidden Gem

The `correlation/keys.go` package (157 lines) implements production-grade trace correlation:

- **W3C Traceparent** header parsing
- **B3 Single** header parsing  
- **Google Cloud Trace** header parsing
- **Generic request_id / correlation_id** extraction from headers AND log text
- Normalization and deduplication

This enables the browser-to-backend `full_stack_error` rule chain. The schema defines `net_correlation_key` facts that bridge browser HTTP headers to Docker log messages.

**MangleCP lesson:** Cross-domain fact correlation (browser ↔ backend, service ↔ service) is extremely powerful when done through shared correlation keys. MangleCP's `facts_profile` should define standard correlation key predicates for multi-system scenarios.

---

## Architecture Decisions to Question

| BrowserNERD Pattern | Worth Keeping? | Why |
|---|---|---|
| `SimpleInMemoryStore` | **No** — use `TemporalStore` | DatalogMTL is MangleCP's core differentiator |
| Dual-store (Mangle + Go buffer) | **Maybe** — depends on TemporalStore reliability | Buffer fallback compensates for store limitations |
| Incremental eval on every fact batch | **Configurable** | Continuous eval is expensive; intent-driven eval might suffice |
| Adaptive sampling | **Yes** | Essential for high-volume fact ingestion |
| Pub/sub watch mode | **Future extension** | Not needed for stateless request/response protocol v1 |
| `progressive_only` flag | **Evolved** | MangleCP always operates in "progressive only" — no static tool list |
| OpenTelemetry correlation | **Adopt** | Cross-system fact joining is a killer feature |
| 990-line `.mg` schema | **Reference pattern** | Shows what a real production schema looks like |

---

## Summary

BrowserNERD validates MangleCP's thesis but also reveals **where the real complexity lives**: the schema (990 lines), the progressive tool logic (3,019 lines), and the workarounds for Mangle's limitations (negation, temporal stores, arity matching). MangleCP can leapfrog these pain points by building on DatalogMTL's `TemporalStore` from the start, defining safety limits in the protocol, and replacing fixed progressive tools with dynamically synthesized macro-tools.
