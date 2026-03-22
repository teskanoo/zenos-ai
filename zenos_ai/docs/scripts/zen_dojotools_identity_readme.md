# Zen DojoTools Identity — 4.5.0 'Meridian'

*Resolve user and construct identities within the ZenOS-AI household*

---

## Overview

`zen_dojotools_identity` is the identity resolver for ZenOS-AI. When called, it looks up a registered user, AI construct, or cabinet in the household roster and returns the corresponding identity record.

Called with no target, it returns the full roster of all registered identities (directory behavior). Called with a specific target, it returns that identity's record. Called with `mode: build_identity_manifest`, it rebuilds the cached identity manifest in the household cabinet.

The resolver is **MCP-exposed** and stateless for `resolve` and `prompt` modes — it makes no writes, fires no events, and has no side effects. `build_identity_manifest` mode writes to the household cabinet.

> **Note on scope:** The identity system currently handles resolution only — looking up who something is and returning their record. Privilege enforcement, session tokens, and security masking are planned post-GA features. The resolver is the foundation; the gatekeeper is coming.

---

## Modes

| Mode | Description |
|---|---|
| `resolve` (default) | Returns identity record for target, or full roster if no target |
| `prompt` | Returns the rendered prompt capsule for the target construct |
| `build_identity_manifest` | Calls `identity_roster()`, writes `zen_identity_manifest` to household cabinet |

`resolve` is the default — existing callers are unaffected. `build_identity_manifest` is the preferred way to seed or refresh the manifest; it can also be triggered via `zen_event` with `kind: identity_manifest_rebuild`.

---

## Input Fields

| Field | Type | Status | Description |
|---|---|---|---|
| `mode` | select | Active | `resolve`, `prompt`, or `build_identity_manifest` |
| `user_label` | text | Active | Label referencing a registered ZenOS-AI user |
| `user_cabinet` | entity (sensor) | Planned | Cabinet sensor entity_id |
| `user_entity_id` | entity (person) | Planned | HA person entity_id |
| `user_guid` | text | Planned | ZenOS-AI user GUID (UUID format) |

All target fields are optional. Omit all inputs to retrieve the full roster.

When multiple inputs are provided, only one is used — priority order:

```
user_label → user_cabinet → user_entity_id → user_guid
```

`user_label` is the primary and recommended input path. The other three inputs are defined and validated but not yet functional.

---

## Resolution Flow

1. **Input selection** — the first non-empty field in priority order becomes the target
2. **Normalization** — the target is cleaned and classified:
   - `person.*` → preserved as person entity
   - `sensor.*` → preserved as cabinet sensor
   - UUID format → preserved as GUID
   - Plain string → treated as a label
   - Empty / whitespace / invalid → `None` (returns full roster)
3. **Lookup** — delegates to `identity_resolve_source(target)` in `zen_os_1.jinja` — the same full resolution pipeline the prompt uses, so the tool is a true check tool
4. **Response** — returns the identity record(s) with a timestamp

---

## Usage

### Full roster (no target)

```yaml
# No inputs — returns all registered identities (directory behavior)
```

```json
{
  "result": [
    { "name": "Nathan", "role": "head_of_household", ... },
    { "name": "Friday", "type": "ai_construct", ... }
  ],
  "target_normalized": null,
  "timestamp": "2026-03-14T12:00:00.000000+00:00"
}
```

### Build identity manifest

```yaml
mode: build_identity_manifest
# No target needed — resolves full roster and writes zen_identity_manifest to household cabinet
```

```json
{
  "result": "ok",
  "count": 2,
  "manifest_written": true,
  "timestamp": "2026-03-22T10:00:00.000000+00:00"
}
```

Can also be triggered via event: fire `zen_event` with `kind: identity_manifest_rebuild`. The scheduler fires this automatically on `ha_start` and `daily_midnight`.

### Lookup by label

```yaml
user_label: nathan
```

```json
{
  "result": { "name": "Nathan", "role": "head_of_household", ... },
  "target_normalized": "nathan",
  "timestamp": "2026-03-14T12:00:00.000000+00:00"
}
```

### Not found

```json
{
  "result": [],
  "error": "not_found"
}
```

---

## Response Format

| Field | Description |
|---|---|
| `result` | Identity record (object) for single lookup, array for full roster, empty array on error |
| `target_normalized` | The resolved target after normalization (`null` for full roster) |
| `timestamp` | ISO 8601 response timestamp |
| `error` | `"not_found"` or `"empty_input"` on failure (replaces result fields) |

---

## What's Coming Post-GA

The identity module is the foundation for the ZenOS-AI security model. Once the resolver is stable, the planned additions are:

- **Session tokens** — issued per interaction, tied to caller identity
- **Security masks** — every tool call will pass through the identity resolver to receive a privilege mask before execution
- **ACL enforcement** — cabinet-level access control based on identity role and relationship
- **Privilege gating** — constructs can only see and act within their authorized scope

The current resolver is the first layer. The gatekeeper builds on top of it.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `zen_os_1.jinja` → `identity_resolve_source()` | Full identity resolution pipeline — same code path as prompt assembly |
| `zen_os_1.jinja` → `identity_roster()` | Full household roster — used by `build_identity_manifest` |
| `zen_os_1.jinja` → `identity_manifest_loader()` | Reads `zen_identity_manifest` drawer from household cabinet |
| Zen AI User Cabinet | Identity record storage |
