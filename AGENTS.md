# MangleContextProtocol — Agent Instructions

## Mission

**Replace the "static list of atomic tools" paradigm with server-computed, context-validated macro-tools powered by Mangle's temporal Datalog reasoning.**

MangleCP is a wire protocol — not an agent framework, not a planner, not a memory system. It specifies how a server embeds the Mangle runtime, evaluates DatalogMTL-style rules against client-supplied intent and time-scoped facts, and returns the smallest, most relevant tool payloads over the wire. The result is a protocol that is:

- **Intent-scoped** — tools are synthesized per-request, not listed statically
- **Temporally correct** — evaluation is deterministic relative to an explicit time
- **Structurally secure** — authentication is optional (as in MCP), but the protocol is inherently more guarded: no tool enumeration, intent-scoped evaluation gates what the server reveals, and side-effects metadata with confirmation flags provide defense-in-depth without requiring credentials
- **Token-efficient** — inspired by browserNerd's progressive disclosure and macro-execution model

The north star: **any AI client should be able to connect to a MangleCP server, declare an intent, and receive a mathematically constrained, context-proven set of capabilities — nothing more, nothing less.**

---

## Repository Structure

```
MangleContextProtocol/
├── AGENTS.md              ← You are here
├── .gitignore
├── Docs/
│   └── Research/          ← Protocol whitepaper and reference materials
├── mangle/                ← Upstream Mangle repo (git-ignored, see below)
└── (future: server/, protocol/, examples/, tests/)
```

---

## Upstream Mangle Dependency

The `mangle/` directory contains a full clone of [google/mangle](https://github.com/google/mangle) as of **2026-02-18**. It is listed in `.gitignore` and is **not** tracked in this repository's version control.

### Sync Policy

The upstream Mangle repo is under active development (DatalogMTL, temporal stores, new built-ins). Agents working in this codebase should:

1. **Check for upstream changes weekly** — pull the latest from `https://github.com/google/mangle.git` into the `mangle/` directory.
2. **Review changelogs and commit history** after each pull for:
   - New temporal operators or evaluation semantics
   - Changes to `factstore.TemporalStore` or engine evaluation
   - New built-in functions relevant to MangleCP
   - Breaking API changes in the Go library surface
3. **Document notable upstream changes** in `Docs/Research/` with a dated note when they affect MangleCP's protocol design or implementation assumptions.

### How to Sync

```bash
cd mangle
git pull origin main
```

If the upstream introduces breaking changes to APIs MangleCP depends on, flag it immediately — don't silently adapt.

---

## Design Boundaries (What MangleCP Is *Not*)

Agents contributing to this project must respect these hard boundaries:

- **No agent loop** — MangleCP does not define planning, reasoning, or autonomy
- **No memory model** — MangleCP does not define client or server state persistence
- **No natural language** — Intent is a structured payload, not a prompt
- **Stateless wire protocol only** — JSON message shapes, negotiation, and payload semantics

If a feature requires crossing these boundaries, it belongs in a *client* or *server implementation*, not in the protocol specification itself.

---

## Reference Project Research Methodology

When studying reference projects (e.g., BrowserNERD, MCP, A2A):

1. **Start with docs** — README, AGENTS/CLAUDE.md, changelogs. Get the high-level picture.
2. **Verify against source code** — Docs go stale fast. Always inspect actual Go/TS/Rust source to confirm tool counts, predicate schemas, engine architecture, and feature claims.
3. **Create a case study** — Write findings to `Docs/Research/<ProjectName>_CaseStudy.md` with source-verified data.
4. **Add to reference list** — Update `Docs/Research/referenceProjectList.md`.
5. **Update upstream docs** — If the reference project is internal and the docs are wrong, fix them in the original project.

---

## Key References

| Document | Location |
|----------|----------|
| Protocol Whitepaper | `Docs/Research/gpt5.2MangleContextProtocol.md` |
| Mangle README | `mangle/README.md` |
| Mangle Temporal Docs | `mangle/docs/temporal.md` |
| Mangle Examples | `mangle/examples/` |
