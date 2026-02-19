# MangleCP Specification 08: Security Model

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), Spec 02 (Transport), Spec 04 (Manifest), Spec 06 (Macro-Tools)

---

## 1. Purpose

This document specifies MangleCP's security model: the dual-layer safety architecture, authentication and authorization requirements, side-effect gating, confirmation workflows, and data minimization principles.

MangleCP's security posture relies primarily on **structural security** -- the anti-enumeration property, intent-scoped evaluation, and side-effect metadata with confirmation flags provide defense-in-depth without requiring credentials. Authentication is RECOMMENDED for network transports but OPTIONAL, as stated in the project's design principles.

---

## 2. Dual-Layer Safety Architecture

MangleCP mandates two independent, complementary security layers. Neither alone is sufficient.

### 2.1 Layer 1: Protocol-Level Constraints (Unforgeable)

Protocol-level constraints are enforced by the server's host language code (Go, Rust, etc.) and CANNOT be overridden by Mangle rules, client-supplied facts, or configuration changes.

| Constraint | Enforcement | Rationale |
|------------|-------------|-----------|
| Message size limits | Wire format parser rejects oversized messages | Prevents resource exhaustion |
| Message schema validation | JSON Schema validation before processing | Prevents malformed input |
| Authentication verification | Token validation before message dispatch | Prevents unauthorized access |
| Transport security | TLS enforcement for network transports | Prevents eavesdropping |
| Derivation gas limits | Engine-level `DerivedFactsLimit` enforcement | Prevents non-termination |
| Predicate name reservation | `_manglecp_` prefix rejected from client facts | Prevents protocol predicate spoofing |

These constraints are the MangleCP analog of codeNERD's Go-level constitutional checks that block destructive commands, secret exfiltration, and path traversal regardless of what Mangle rules say.

### 2.2 Layer 2: Mangle Policy Rules (Configurable)

Mangle policy rules implement domain-specific safety logic. They are configurable and auditable but operate within the bounds set by Layer 1.

```mangle
# Default-deny permission model
permitted(Action, Target, Payload) :-
    safe_action(Action, Target, Payload),
    !dangerous_content(Action, Target, Payload).

# Side-effect gating
requires_confirmation(MacroId) :-
    macro_has_side_effect(MacroId, Effect),
    high_risk_effect(Effect).

high_risk_effect("payments").
high_risk_effect("destructive").
high_risk_effect("authentication").
```

Policy rules determine:
- Which macro-tools require user confirmation
- Which side effects are classified as high-risk
- Which intents are permitted for authenticated vs. anonymous clients
- Which facts a client is allowed to assert

### 2.3 Layer Interaction

The layers operate sequentially:

```
Client Request
    |
    v
[Layer 1: Protocol Checks]     -- unforgeable, always runs first
    | Pass
    v
[Layer 2: Mangle Policy]       -- configurable, rule-evaluated
    | Pass
    v
[Evaluation / Execution]
```

If Layer 1 rejects a request, Layer 2 is never consulted. If Layer 1 passes but Layer 2 rejects, the request is denied with a policy-level error.

---

## 3. Authentication

### 3.1 Authentication Requirements

| Transport | Authentication | Rationale |
|-----------|---------------|-----------|
| HTTP/HTTPS | RECOMMENDED | Network-exposed endpoints benefit from auth, but structural security (anti-enumeration, intent-scoping) provides baseline protection |
| WebSocket (network) | RECOMMENDED | Same exposure as HTTP |
| WebSocket (localhost) | OPTIONAL | Local services; process-level and structural security provide protection |
| Stdio | OPTIONAL | Local-trust context; process-level isolation provides security |

> **Implementation Note:** While authentication is optional, servers exposed to the public internet SHOULD enable authentication. MangleCP's anti-enumeration property means that even without auth, an attacker cannot discover the server's full capability surface -- but authentication adds valuable defense-in-depth for production deployments.

### 3.2 Supported Authentication Schemes

MangleCP servers MUST support at least one of the following schemes:

#### Bearer Token

```
Authorization: Bearer <token>
```

The simplest scheme. Suitable for API key authentication and pre-shared tokens. Servers validate the token against a configured set of valid tokens or a token introspection endpoint.

#### OAuth 2.1

Servers supporting OAuth 2.1 MUST advertise the token endpoint in the manifest's `auth.token_url` field. Clients obtain tokens via standard OAuth 2.1 flows (Authorization Code with PKCE recommended for interactive clients, Client Credentials for service-to-service).

#### API Key

```
X-MangleCP-API-Key: <key>
```

A header-based alternative to Bearer tokens for environments where Authorization headers are inconvenient. Servers supporting API keys MUST also accept Bearer tokens.

### 3.3 Authentication in the Protocol Flow

For session transports (WebSocket):
1. Client connects to the WebSocket endpoint
2. Server sends the manifest (which includes `auth.required` and `auth.schemes`)
3. If auth is required, the client MUST send an `authenticate` message before any intent request:

```json
{
  "type": "authenticate",
  "id": "auth-001",
  "manglecp": "2026-02-draft",
  "payload": {
    "scheme": "bearer",
    "token": "<token>"
  }
}
```

4. Server responds with authentication result:

```json
{
  "type": "authenticate_response",
  "id": "auth-001",
  "manglecp": "2026-02-draft",
  "payload": {
    "status": "authenticated",
    "identity": "<string | null>",
    "permissions": ["<string>", ...]
  }
}
```

For HTTP transport:
- Authentication is carried in HTTP headers on every request
- No separate authentication message is needed

### 3.4 Anonymous Access

Servers MAY support anonymous access by setting `auth.required: false` in the manifest. This is ONLY appropriate for:

- Local development servers
- Public demo/documentation endpoints
- Stdio transport

Servers supporting anonymous access SHOULD restrict available intents and side effects for unauthenticated clients via Layer 2 Mangle policy rules:

```mangle
# Only observation intents for anonymous users
permitted_intent("anonymous", "observe").
!permitted_intent("anonymous", "modify").
!permitted_intent("anonymous", "execute").
```

---

## 4. Side-Effect Gating

### 4.1 Side-Effect Classification

Every macro-tool MUST declare its side effects in the `safety.side_effects` array (Spec 06 Section 2.3). Servers classify side effects using Mangle rules:

```mangle
macro_has_side_effect(MacroId, "filesystem") :-
    macro_includes_action(MacroId, Action),
    action_writes_file(Action).

macro_has_side_effect(MacroId, "network") :-
    macro_includes_action(MacroId, Action),
    action_makes_request(Action).

macro_has_side_effect(MacroId, "none") :-
    !macro_has_side_effect(MacroId, _).
```

### 4.2 Confirmation Workflow

For macro-tools with `requires_user_confirmation: true`:

1. Client receives the macro-tool in an intent response with `safety.requires_user_confirmation: true`
2. Client presents the tool's description, side effects, and reversibility to the user
3. User confirms (or denies)
4. If confirmed, client generates a confirmation token (implementation-defined, e.g., HMAC of `macro_id + timestamp + user_session`)
5. Client sends the invoke request with `confirmation_token`
6. Server validates the confirmation token

The confirmation token mechanism is intentionally lightweight. The protocol does not mandate a specific token format -- it only requires that the server can verify the token was generated after a user interaction.

### 4.3 Confirmation Bypass

For automated/non-interactive clients (CI/CD pipelines, batch processors), servers MAY support a `trust_level` in the authentication response:

```json
{
  "status": "authenticated",
  "trust_level": "automated",
  "auto_confirm_effects": ["filesystem", "network"]
}
```

When `trust_level` is `"automated"`, the server MAY waive confirmation requirements for the listed side effect categories. This MUST be an explicit server configuration, not a default.

---

## 5. Intent-Scoped Exposure Reduction

### 5.1 The Anti-Enumeration Property

MangleCP's most significant security property versus MCP is the **anti-enumeration property**: a client cannot enumerate all server capabilities through the protocol.

- The manifest declares intent types and fact schemas, NOT tool lists
- Each intent evaluation returns only the macro-tools relevant to that specific intent and fact state
- Different intents with different facts produce different tool sets
- There is no `list_tools` endpoint or equivalent

This means an attacker with protocol access can only discover capabilities by providing valid intents and facts. The server's full capability surface is never exposed in a single response.

### 5.2 Intent-Scoped Fact Filtering

Servers SHOULD implement fact-level access control:

```mangle
# Only show diagnostic tools if the client has diagnostic facts
visible_tool(MacroId) :-
    final_selected(MacroId, _, _),
    !requires_privileged_facts(MacroId).

visible_tool(MacroId) :-
    final_selected(MacroId, _, _),
    requires_privileged_facts(MacroId),
    client_has_privilege("diagnostics").
```

### 5.3 Temporal Scoping as Security Control

Temporal validity windows on macro-tools serve a dual purpose:

1. **Freshness**: Ensuring tools are used while their supporting facts are still valid
2. **Security**: Limiting the window during which a tool invocation is accepted

A macro-tool with `expires_at` set to 5 minutes after evaluation limits the exposure window if a macro_id is leaked or intercepted.

---

## 6. Data Minimization

### 6.1 Client-Side Minimization

Clients SHOULD:

- Send only the facts necessary for the current intent (not all known facts)
- Use the manifest's `intents[].required_facts` and `optional_facts` to determine the minimal fact set
- Avoid sending sensitive data (credentials, PII) in facts unless the intent requires it
- Prefer temporal facts with narrow windows over unbounded facts

### 6.2 Server-Side Minimization

Servers SHOULD:

- Return only the macro-tools relevant to the intent (enforced by the evaluation behavioral contracts)
- Include only necessary skills and context in macro-tool responses
- Limit `state_delta` to facts the client actually needs for follow-on intents
- Use `"condensed"` or `"minimal"` disclosure for lower-relevance tools rather than omitting them entirely (allows awareness without detail exposure)
- Omit `proof_hints` unless explicitly requested

### 6.3 Temporal Data Sensitivity

Temporal facts can encode behavioral patterns (login times, usage frequency, error histories). Servers processing temporal facts SHOULD:

- Apply temporal window bounds to queries (avoid unbounded past lookups)
- Coalesce temporal intervals to reduce granularity when detailed timing is not needed
- Not persist client-supplied temporal facts beyond the evaluation that consumed them

---

## 7. Transport Security

### 7.1 TLS Requirements

For network transports (HTTP, WebSocket over network):

- Servers MUST support TLS 1.2 or higher
- TLS 1.3 is RECOMMENDED
- Servers SHOULD use valid certificates from a trusted CA
- Self-signed certificates are acceptable ONLY for development/testing

### 7.2 Localhost Exemption

Servers running on `localhost` (127.0.0.1, ::1) MAY use plain HTTP/WebSocket without TLS. However, servers SHOULD still support TLS on localhost for defense against local network attacks (DNS rebinding, etc.).

### 7.3 Stdio Security

Stdio transport inherits the security properties of the process boundary:

- The client process controls the server process lifecycle
- Communication is confined to stdin/stdout pipes
- No network exposure
- Authentication is OPTIONAL (process-level trust)

---

## 8. Threat Model

### 8.1 Threats Addressed

| Threat | Mitigation |
|--------|------------|
| Tool enumeration / capability scanning | Anti-enumeration property: no `list_tools` endpoint; intent-scoped evaluation |
| Unauthorized access | Anti-enumeration property provides baseline; authentication (when enabled) adds defense-in-depth |
| Prompt injection via facts | Layer 2 policy rules validate fact predicates; `_manglecp_` prefix reservation |
| Resource exhaustion via evaluation | Gas limits (`max_derived_facts`), timeout (`max_compute_ms`), message size limits |
| Replay attacks | Temporal validity windows on macro-tools; confirmation tokens |
| Side-effect abuse | Side-effect classification, user confirmation workflow, trust levels |
| Data exfiltration via facts | Data minimization principles; server-side fact filtering |
| Non-termination via temporal rules | Interval coalescing, interval-per-atom limits, DatalogMTL termination checks |

### 8.2 Threats Not Addressed

| Threat | Rationale |
|--------|-----------|
| Client-side vulnerabilities | Out of scope (client implementation concern) |
| Server-side code injection | Out of scope (server implementation concern) |
| Network-level attacks (DDoS) | Standard infrastructure concern, not protocol-specific |
| Malicious `.mg` rules | Server operators are responsible for their own rule sets |
