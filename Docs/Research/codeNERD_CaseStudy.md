# Protocol Verification Study: codeNERD

> [!NOTE]
> **Focus:** Extracting Protocol Requirements for MangleCP
> **Subject:** `codeNERD` (Agentic Reference Implementation)
> **Goal:** Validate MangleCP wire-format decisions against production patterns.

---

## 1. Introduction

This study identifies the *minimal wire requirements* to support a `codeNERD`-class agent over MangleCP. It filters out agent-internal logic (prompts, heuristics) to focus strictly on **Transport**, **Data Structures**, and **State Negotiation**.

**Core Finding:** The "Agentic" behaviors of `codeNERD` (self-correction, tool synthesis, dynamic context) rely on specific data structures that standard protocols (LSP, MCP) do not carry. MangleCP must adopt these structures as first-class protocol primitives.

---

## 2. Wire Format: The "Piggyback" Control Channel

**Source:** `internal/articulation/emitter.go`
**Struct:** `PiggybackEnvelope`

`codeNERD` implements a strict dual-channel response format to solve the "Premature Articulation" problem (where an LLM hallucinates an action before the system is ready).

### 2.1 The Envelope Structure

```go
// From articulation/emitter.go
type PiggybackEnvelope struct {
 Control ControlPacket `json:"control_packet"` // Logic/System channel
 Surface string        `json:"surface_response"` // User/Chat channel
}
```

The `ControlPacket` carries the machine-readable state *invisibly* to the user:

* `MangleUpdates []string`: Direct Datalog fact assertions (`state_delta` in MangleCP).
* `IntentClassification`: The finalized intent that drove the action.
* `ContextFeedback`: Relevance scoring for the previous context window.

**Protocol Requirement:**
MangleCP responses **MUST** separates `content` (Surface) from `state_delta` (Control). The `state_delta` must be capable of carrying raw logic atoms (predicates) to update the client's view of the world.

---

## 3. Tool Discovery: Render Modes (Dynamic Manifests)

**Source:** `internal/mcp/types.go`
**Enum:** `RenderMode`

`codeNERD` does not have a static `tools.json`. Instead, its `JITToolCompiler` selects tools and assigns them a "Render Mode" based on relevance to the current *Intent*.

### 3.1 The Progressive Disclosure Pattern

```go
// From mcp/types.go
type RenderMode string
const (
 RenderModeFull      RenderMode = "full"      // Complete JSON Schema
 RenderModeCondensed RenderMode = "condensed" // Name + 1-line doc
 RenderModeMinimal   RenderMode = "minimal"   // Name only (hallucination primer)
 RenderModeExcluded  RenderMode = "excluded"  // Hidden entirely
)
```

**Protocol Requirement:**
MangleCP's `MacroTool` object **SHOULD** support a `render_mode` or equivalent hinting mechanism. This allows the server to say "I have this tool, but I'm only giving you the name right now to save token budget; ask for the full schema if you really need it."

---

## 4. Negotiation: The "Pending" State (Autopoiesis)

**Source:** `internal/autopoiesis/autopoiesis_types.go`
**Enum:** `LoopStage`

When a required tool is missing, `codeNERD` engages the **Ouroboros Loop** to build it. This is not instantaneous. The agent moves through distinct states while the user waits.

### 4.1 The States of Creation

```go
// From autopoiesis/autopoiesis_types.go
const (
 StageDetection    LoopStage = iota // "I see I need a tool"
 StageSpecification                 // "I am writing the spec"
 StageSafetyCheck                   // "I am auditing the code"
 StageSimulation                    // "I am dry-running the tool"
 StageRegistration                  // "Tool is ready"
)
```

**Protocol Requirement:**
MangleCP needs a standard way to represent **Long-Running Capability Negotiation**.

* **Draft Proposal:** `IntentResponse` should support a status of `generating` or `pending` with a `progress` field, mapping to these stages.
* **State Machine:** The client must know *not* to timeout while the server is in `StageSimulation`.

---

## 5. Transport: SSE + POST

**Source:** `internal/mcp/transport_sse.go`

`codeNERD` uses Server-Sent Events (SSE) for the "Control Channel" (server-initiated events, progress updates) and HTTP POST for the "Action Channel" (client-initiated intent requests).

**Protocol Requirement:**
MangleCP should mandate **SSE (or WebSocket)** as the primary transport for Agentic/Stateful connections. Pure HTTP Request/Response is insufficient for an agent that needs to emit "Thought Updates" (Reasoning Traces) or "Generation Progress" (Ouroboros) while processing a single user intent.

---

## 6. Feedback Loops: Context Relevance

**Source:** `internal/articulation/emitter.go`
**Struct:** `ContextFeedback`

The agent provides explicit feedback on whether the context facts it received were useful used.

```go
type ContextFeedback struct {
 OverallUsefulness float64
 HelpfulFacts      []string // "These facts helped me solve it"
 NoiseFacts        []string // "These facts wasted my tokens"
}
```

**Protocol Requirement:**
MangleCP's `InvokeResponse` and `IntentResponse` should optionally carry `feedback` metrics. This allows the client (the "Host") to optimize its context-gathering algorithms (Spreading Activation) based on what the server (the "Agent") actually found valuable.

---

## 7. Conclusion

`codeNERD` confirms that MangleCP is solving the right problems, but identifies specific data structures that are mature enough to be lifted directly into the standard:

1. **PiggybackEnvelope** -> `MangleCP.ResponseEnvelope`
2. **RenderMode** -> `MangleCP.ToolView`
3. **LoopStage** -> `MangleCP.NegotiationState`
4. **ContextFeedback** -> `MangleCP.ContextMetrics`
