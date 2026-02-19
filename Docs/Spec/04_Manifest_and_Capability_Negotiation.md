# MangleCP Specification 04: Manifest and Capability Negotiation

**Status:** Draft
**Version:** 2026-02-draft
**References:** Spec 01 (Architecture), Spec 02 (Transport), Spec 03 (Data Model)

---

## 1. Purpose

This document specifies the MangleCP manifest -- the lightweight capability advertisement that a server broadcasts on connection. The manifest enables clients to understand what a server can do, how to interact with it, and what constraints apply, without enumerating individual tools.

---

## 2. Manifest Delivery

### 2.1 Transport-Specific Delivery

| Transport | Delivery Mechanism | Timing |
|-----------|-------------------|--------|
| WebSocket | First application message after handshake | Immediate |
| HTTP | GET `/.well-known/manglecp/manifest.json` | On client request |
| Stdio | First line written to stdout | Immediate on process start |

### 2.2 Manifest Message

The manifest is delivered as a standard MangleCP message envelope:

```json
{
  "type": "manifest",
  "id": null,
  "manglecp": "2026-02-draft",
  "payload": { /* Manifest Object */ }
}
```

The `id` MUST be null for manifests (server-initiated, no correlation).

---

## 3. Manifest Object

### 3.1 Full Schema

```json
{
  "server_name": "<string>",
  "server_version": "<string>",
  "protocol": {
    "manglecp": "<string>",
    "supported_versions": ["<string>", ...]
  },
  "status": "<string>",
  "domain": {
    "id": "<string>",
    "description": "<string>",
    "categories": ["<string>", ...],
    "affinities": {
      "coding": 80,
      "testing": 60
    }
  },
  "endpoints": {
    "intent_eval": "<URI>",
    "macro_invoke": "<URI>"
  },
  "intents": [
    {
      "name": "<string>",
      "description": "<string>",
      "required_facts": ["<predicate name>", ...],
      "optional_facts": ["<predicate name>", ...]
    }
  ],
  "facts_profile": {
    "time_formats": ["<string>", ...],
    "predicates": [ /* Predicate Schema Objects (Spec 03 Section 8) */ ],
    "schema_uri": "<URI | null>"
  },
  "capabilities": {
    "temporal": "<boolean>",
    "aggregation": "<boolean>",
    "external_predicates": [
      {
        "predicate": "<string>",
        "arity": "<integer>",
        "description": "<string>",
        "deterministic": "<boolean>"
      }
    ],
    "rule_submission": "<boolean>",
    "subscriptions": "<boolean>"
  },
  "limits": {
    "max_message_bytes": "<integer>",
    "max_facts_per_request": "<integer>",
    "max_derived_facts": "<integer>",
    "max_intervals_per_atom": "<integer>",
    "max_compute_ms": "<integer>"
  },
  "auth": {
    "required": "<boolean>",
    "schemes": ["<string>", ...],
    "token_url": "<URI | null>"
  },
  "extensions": {}
}
```

### 3.2 Field Definitions

#### 3.2.1 Server Identity

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `server_name` | string | REQUIRED | Human-readable server name |
| `server_version` | string | REQUIRED | Server implementation version (SemVer RECOMMENDED) |

#### 3.2.2 Protocol

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `protocol.manglecp` | string | REQUIRED | Current protocol version: `"2026-02-draft"` |
| `protocol.supported_versions` | array of strings | OPTIONAL | All protocol versions the server supports |

#### 3.2.3 Server Status

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | REQUIRED | Server readiness state (see Section 4) |

#### 3.2.4 Domain

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `domain.id` | string | REQUIRED | Machine-readable domain identifier (e.g., `"browser_automation"`, `"codebase_analysis"`, `"spec_implementation"`) |
| `domain.description` | string | REQUIRED | Human-readable description of the server's domain and purpose |
| `domain.categories` | array of strings | OPTIONAL | Standardized capability categories (see Section 5) |
| `domain.affinities` | object | OPTIONAL | A map of string keys to integer weights (0-100) representing the server's semantic affinity for specific operational workspaces (e.g., `{"coding": 80, "testing": 60}`). |

#### 3.2.5 Endpoints

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `endpoints.intent_eval` | URI | REQUIRED (HTTP) | Endpoint for intent evaluation requests |
| `endpoints.macro_invoke` | URI | REQUIRED (HTTP) | Endpoint for macro-tool invocation requests |

For session transports (WebSocket, stdio), endpoints MAY be omitted -- all message types are multiplexed over the single connection.

#### 3.2.6 Intents

The `intents` array advertises the intent types the server understands. This is NOT an enumeration of tools; it is a declaration of the **kinds of questions** the server can answer.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `intents[].name` | string | REQUIRED | Intent name matching the `intent.name` field in intent requests |
| `intents[].description` | string | REQUIRED | Human-readable description of what this intent evaluates |
| `intents[].required_facts` | array of strings | OPTIONAL | Predicate names that MUST be present in the intent request for this intent type |
| `intents[].optional_facts` | array of strings | OPTIONAL | Predicate names that MAY be present and will enrich the evaluation |

Servers SHOULD list their primary intents. The list MAY be non-exhaustive -- servers MAY accept intents not listed in the manifest if their rules support them.

#### 3.2.7 Facts Profile

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `facts_profile.time_formats` | array of strings | REQUIRED | Accepted time formats: `"rfc3339"`, `"epoch_ms"`, `"epoch_ns"` |
| `facts_profile.predicates` | array of Predicate Schema Objects | RECOMMENDED | Predicate schemas the server understands (Spec 03 Section 8) |
| `facts_profile.schema_uri` | URI \| null | OPTIONAL | URI where the full `.mg` schema can be retrieved for inspection |

The `predicates` array enables client-side fact validation. It SHOULD include at least all `"input"` direction predicates.

#### 3.2.8 Capabilities

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `capabilities.temporal` | boolean | REQUIRED | Whether the server supports DatalogMTL temporal reasoning |
| `capabilities.aggregation` | boolean | OPTIONAL | Whether the server supports Mangle aggregation operators. Default: `true`. |
| `capabilities.external_predicates` | array | OPTIONAL | External predicates bridging Mangle to runtime data sources |
| `capabilities.rule_submission` | boolean | OPTIONAL | Whether the server accepts client-submitted rules (v2 feature). Default: `false`. |
| `capabilities.subscriptions` | boolean | OPTIONAL | Whether the server supports pub/sub notifications (v2 feature). Default: `false`. |

##### External Predicate Declaration

```json
{
  "predicate": "recall_similar",
  "arity": 3,
  "description": "Embedding-based similarity search. Arguments: query string, k count, result.",
  "deterministic": true
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `predicate` | string | REQUIRED | Predicate name |
| `arity` | integer | REQUIRED | Number of arguments |
| `description` | string | REQUIRED | Human-readable description of what the predicate does and its argument semantics |
| `deterministic` | boolean | OPTIONAL | Whether the predicate returns the same results for the same inputs. Default: `true`. |

Servers MAY include engine-specific metadata in the `extensions` object using the `x-` prefix convention. For example, Mangle-based servers MAY include binding patterns as `x-mangle-mode`:

```json
{
  "predicate": "recall_similar",
  "arity": 3,
  "description": "Embedding-based similarity search. Arguments: query string, k count, result.",
  "deterministic": true,
  "x-mangle-mode": "bbf"
}
```

#### 3.2.9 Limits

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `limits.max_message_bytes` | integer | REQUIRED | Maximum message size the server accepts |
| `limits.max_facts_per_request` | integer | REQUIRED | Maximum number of facts in a single intent request |
| `limits.max_derived_facts` | integer | REQUIRED | Maximum derived facts per evaluation (gas limit) |
| `limits.max_intervals_per_atom` | integer | OPTIONAL | Maximum temporal intervals per atom (temporal safety) |
| `limits.max_compute_ms` | integer | OPTIONAL | Maximum evaluation time in milliseconds |

These limits correspond to Mangle's termination controls (`DerivedFactsLimit`, interval caps, coalescing). Clients SHOULD respect these limits when constructing requests. Servers MUST enforce them during evaluation.

#### 3.2.10 Authentication

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `auth.required` | boolean | REQUIRED | Whether authentication is required. RECOMMENDED to be `true` for network transports (see Spec 08). |
| `auth.schemes` | array of strings | REQUIRED if `auth.required` is true | Supported authentication schemes: `"bearer"`, `"oauth2"`, `"api_key"` |
| `auth.token_url` | URI \| null | OPTIONAL | OAuth 2.1 token endpoint for `"oauth2"` scheme |

#### 3.2.11 Extensions

The `extensions` object is a catch-all for server-specific metadata not covered by the standard fields. Extension keys MUST be prefixed with `x-` to avoid collisions with future protocol fields.

---

## 4. Server Readiness

### 4.1 Status Values

| Status | Meaning | Client Behavior |
|--------|---------|-----------------|
| `"ready"` | Server is fully initialized and accepting requests | Normal operation |
| `"booting"` | Server is loading schemas, populating facts, initializing | Client MUST NOT send intent requests. Client SHOULD wait and poll/re-read manifest. |
| `"degraded"` | Server is operational but some capabilities are unavailable | Client MAY send requests. Server SHOULD include details in `extensions`. |
| `"draining"` | Server is shutting down; no new requests accepted | Client MUST NOT send new requests. In-flight requests MAY complete. |

### 4.2 Boot Guard

This formalizes codeNERD's boot guard pattern:

- A server in `"booting"` status MUST reject intent requests with error code `"server_not_ready"`.
- On session transports, the server MAY send an updated manifest with `"ready"` status when initialization completes.
- On HTTP transport, clients SHOULD retry the manifest endpoint until status changes from `"booting"`.

---

## 5. Standardized Domain Categories

MangleCP defines the following standard category identifiers for use in `domain.categories`:

| Category | Description |
|----------|-------------|
| `"browser_automation"` | Web browser interaction, DOM, CDP |
| `"codebase_analysis"` | Source code scanning, cross-system reasoning |
| `"workflow_orchestration"` | Multi-step task sequencing and dependency management |
| `"data_integration"` | Database introspection, API aggregation |
| `"testing"` | Test execution, assertion, coverage analysis |
| `"documentation"` | Spec generation, documentation synthesis |
| `"infrastructure"` | Deployment, monitoring, infrastructure-as-code |
| `"security"` | Vulnerability scanning, policy enforcement |

Servers MAY use custom categories prefixed with `x-`.

---

## 6. Manifest Caching

### 6.1 HTTP Caching

For HTTP transport, the manifest endpoint SHOULD set appropriate cache headers:

- `Cache-Control: max-age=300` (5 minutes) for stable servers
- `Cache-Control: no-cache` for servers with dynamic capability changes
- `ETag` header for conditional requests

### 6.2 Session Transport Updates

On session transports (WebSocket, stdio), the server MAY send an updated manifest at any time to signal:

- Status changes (`"booting"` -> `"ready"`)
- Capability changes (new external predicates, limit adjustments)
- Degradation events

Clients MUST accept and process updated manifests received after the initial manifest. The most recently received manifest supersedes all previous ones.

---

## 7. Example Manifest

```json
{
  "type": "manifest",
  "id": null,
  "manglecp": "2026-02-draft",
  "payload": {
    "server_name": "BrowserNERD-MangleCP",
    "server_version": "2.1.0",
    "protocol": {
      "manglecp": "2026-02-draft",
      "supported_versions": ["2026-02-draft"]
    },
    "status": "ready",
    "domain": {
      "id": "browser_automation",
      "description": "Token-efficient browser automation with Mangle-powered reasoning over DOM, network, console, and React state.",
      "categories": ["browser_automation", "testing"],
      "affinities": {
        "browser_automation": 100,
        "testing": 70
      }
    },
    "endpoints": {
      "intent_eval": "/manglecp/evaluate",
      "macro_invoke": "/manglecp/invoke"
    },
    "intents": [
      {
        "name": "observe",
        "description": "Observe current browser state: DOM, network, console, React components",
        "required_facts": ["current_url"],
        "optional_facts": ["session_id", "observation_focus"]
      },
      {
        "name": "diagnose_error",
        "description": "Diagnose page errors using causal chain analysis across console, network, and DOM",
        "required_facts": ["current_url"],
        "optional_facts": ["error_context", "console_event"]
      },
      {
        "name": "automate",
        "description": "Execute a batch automation plan on the current page",
        "required_facts": ["current_url", "automation_goal"],
        "optional_facts": ["interactive_elements", "form_state"]
      }
    ],
    "facts_profile": {
      "time_formats": ["rfc3339", "epoch_ms"],
      "predicates": [
        {
          "predicate": "current_url",
          "arity": 1,
          "arg_types": ["string"],
          "arg_names": ["url"],
          "direction": "input",
          "description": "The URL of the current browser page"
        },
        {
          "predicate": "console_event",
          "arity": 4,
          "arg_types": ["string", "string", "string", "number"],
          "arg_names": ["session_id", "level", "message", "timestamp"],
          "temporal": true,
          "direction": "both",
          "description": "Browser console event"
        }
      ]
    },
    "capabilities": {
      "temporal": true,
      "aggregation": true,
      "external_predicates": [],
      "rule_submission": false,
      "subscriptions": false
    },
    "limits": {
      "max_message_bytes": 16777216,
      "max_facts_per_request": 10000,
      "max_derived_facts": 100000,
      "max_intervals_per_atom": 1000,
      "max_compute_ms": 30000
    },
    "auth": {
      "required": true,
      "schemes": ["bearer"],
      "token_url": null
    },
    "extensions": {}
  }
}
```
