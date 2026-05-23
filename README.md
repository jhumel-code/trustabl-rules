<p align="center">
  <img src="banner.png" alt="Trustabl" width="100%">
</p>

This repository holds the detection rule packs for
[**Trustabl**](https://github.com/trustabl/trustabl), the static analyzer for
agent reliability. The Trustabl engine ships with **no embedded rules** — it
resolves these packs from this repository at scan time (cloning to a local
cache, with offline fallback), so rules can be added or changed without
rebuilding or redistributing the binary.

Every `.yaml` file in this repo (except the top-level `manifest.yaml`) defines
one or more detection rules. The engine's loader walks this tree recursively,
decodes each file, validates it, and runs the matching rules against the SDKs
it discovers in a scanned project.

## How the engine consumes this repo

```
trustabl scan ./repo              # resolves the latest rules, caches, scans
trustabl rules pull               # download/refresh the cache without scanning
trustabl scan ./repo --rules-ref v1.0.0   # pin a tag/branch
trustabl scan ./repo --no-rules-update     # use the cached pack only (offline)
```

The resolved commit SHA of this repo is recorded on every scan result and
folded into the scan's `ScanID`, so a scan is honest about exactly which rule
pack produced it.

## `manifest.yaml` — the schema gate

`manifest.yaml` declares `schema_version`, the rule-schema contract this pack
targets. The engine refuses a pack whose `schema_version` exceeds what its
build supports, so a newer pack never silently misloads against an older
binary. Bump it in lockstep with any engine change that adds a predicate or
schema field.

## Layout

Rules are grouped by `<category>/<topic>.yaml`:

```
manifest.yaml                         schema_version (metadata, not a rule)
claude_sdk/                           Claude Agent SDK rules (CSDK-NNN)
├── agent_safety.yaml                 CSDK-101 (agent scope)
├── error_handling.yaml               CSDK-005
├── idempotency.yaml                  CSDK-006
├── network.yaml                      CSDK-003
├── path_safety.yaml                  CSDK-004
└── tool_definition.yaml              CSDK-001, CSDK-002, CSDK-007
openai_sdk/                           OpenAI Agents SDK rules (OAI-NNN)
├── agent_safety.yaml                 OAI-101..104 (agent scope)
├── decorator_config.yaml             OAI-003, OAI-004
├── mcp_safety.yaml                   OAI-105 (agent scope)
├── network.yaml                      OAI-005
├── path_safety.yaml                  OAI-006
├── tool_definition.yaml              OAI-001, OAI-002
└── tracing.yaml                      OAI-201 (repo scope)
```

The category is the first path segment. Group related rules into a topic file;
1–5 rules per file reads best. The loader walks recursively, so a new category
directory just works once the engine recognizes its `category:` value.

> An `openshell/` pack (OSH-001..005, NVIDIA OpenShell sandbox rules) used to
> live here; it moved to a closed-source companion project. Don't author new
> OSH rules in this repo.

## Authoring rules

The rule schema (every accepted field, with annotations) and the predicate
implementations live in the **engine** repository, not here:

- Schema reference: [`internal/rules/schema.yaml`](https://github.com/trustabl/trustabl/blob/main/internal/rules/schema.yaml)
- Schema types (authoritative): [`internal/rules/schema.go`](https://github.com/trustabl/trustabl/blob/main/internal/rules/schema.go)

Read the schema reference before authoring a rule. The full rule-authoring
contract — required fields, ID conventions, per-scope `applies_to` values,
severity/confidence guidance, and the cross-SDK framing discipline — is in
[`CLAUDE.md`](CLAUDE.md) in this repo.

A rule needing a predicate that doesn't exist yet is a change in the **engine**
repo (schema.go + predicates.go + evaluator.go + schema.yaml), not here. Bump
`manifest.yaml`'s `schema_version` in lockstep and tag a release of this repo
that requires the new engine build.
