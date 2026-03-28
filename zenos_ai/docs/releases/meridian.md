# Release Notes — Codename Meridian

**Merged:** 2026-03-21
**Merge commit:** `8fbf10c`
**Branch:** `fix/meridian → main`
**Follows:** Highlander (`c1942ad`, 2026-03-19)

---

## Summary

Meridian is a hardening and governance batch. No new major features — this release closes
the gap between the Highlander resolver architecture and a stable, production-clean baseline
for GA. The three primary work streams are:

1. **Governance plumbing** — `caller_token` SP1 §4a stub ported to all 15 AI-accessible
   DojoTools scripts. Noop today; enforcement gate is ready for SP1.
2. **Cortex 31 / Cabinet State** — cabinet state visibility wired into the health sensor
   layer. Boot-time race conditions fixed. Expansion cabinet definitions added.
3. **Bug batch** — scheduler heartbeat literal, summarizer double-encode, history
   TemplateSyntaxError, GC KFC drawer protection, labels UPDATE non-atomic risk, and
   several minor footguns cleared.

---

## Changes

### Governance — caller_token §4a Stub

All 15 AI-accessible DojoTools scripts now accept a `caller_token` field and echo it in
their response. The field is a noop — SP1 enforcement is not yet active. This wiring
establishes the full plumbing path from MCP tool call through to response so SP1 can
be activated without touching script internals.

Scripts updated: `filecabinet`, `index`, `inspect`, `query`, `history`, `identity`,
`labels`, `manifest`, `profile_editor`, `summarizers`, `admintools`, `core`,
`image_generator`, `library`, `office`, `systemtools`, `utilities`, `scheduler`.

**Bug fixed alongside:** `default('')` in block dict template values causes
`TemplateSyntaxError: unexpected char ''' at N` in HA's Jinja sandbox. All block dict
`caller_token` fields stripped to `{{ caller_token }}` (no `default()` filter). Safe
because the field carries `default: ''` in the script fields declaration.

---

### Cortex 31 — Cabinet State Visibility

Cabinet state is now surfaced at the health sensor layer.

- `zen_cabinet_health` sensor gains a `cabinet_states` attribute — per-cabinet init/online
  state map, readable at runtime without a FileCabinet call.
- `this` reference removed from `state:` fields in trigger-based template sensors (FG-TBD
  asyncio race — HA evaluates `this` before the sensor registers on first boot).
- 5 expansion cabinet sensor definitions added (`v4.3.0`).
- Duplicate `cabinet_states` attribute removed from `zenos_cabinet_health`.

---

### KFC Schema 1.2.0

KFC drawer `meta` block extended with three new fields:

- `meta.enabled` — replaces the external HA switch as the component on/off signal.
  Component state lives in the drawer itself.
- `meta.requires` — machine-readable capability requirement (`cert`, `level`). Runtime
  predicate, not documentation. Checked at dispatch time when zen_auth is live.
- `label` — seed label for the component. Used by Flynn Gate 3 bootstrap validation
  and the provisioner.

`reset_template` upconverted to emit 1.2.0 schema. `zen_admintools_reset_template`
should be pressed after upgrade to stamp all active KFC drawers.

---

### Labels — UPDATE Patch Semantics

`zen_dojotools_labels` UPDATE action is now safe.

Previously: delete → recreate → retag, with no validation before the delete. A failed
recreate (invalid color, missing param) would permanently destroy the label and all entity
associations with no rollback.

Now:
- **Pre-read**: existing label metadata fetched via REST API before any destructive action.
- **Param resolution**: `new_description`, `new_icon`, `new_color` resolved against
  existing values before delete. Color falls back to `primary`, never the raw stored hex.
- **Confirm gate**: resolved plan returned to caller on first call; actual delete/recreate
  only fires on second call with `confirm: true`.
- **Entity pre-capture**: `label_entities()` captured before delete; retag runs after
  recreate.

---

### Identity / Profile Editor v2

- Three-layer identity schema shim (`familiar → companion seed`) — legacy identity
  records migrate cleanly into the canonical three-layer structure.
- Profile editor v2: cabset resolver paths wired in, principal chain connected.
- Cabset expanded to 4 user cabinets + 2 AI user cabinets + vault structural exclusion.

---

### Cortex 30 — Token Optimization

Cortex 30 prompt primitives optimized: 65% token reduction across Purpose and Directives
blocks. Functional parity maintained. Cortex 31 prompt primitives added alongside.

---

### Flynn / Sentinel

- Sentinel self-heal wired: sentinel KFC drawers missing from Dojo are re-stamped on
  boot rather than silently dropped.
- Expansion slot wiring: Flynn now initializes the 5 expansion cabinet sensors in the
  correct slot-to-type map. Vault intentionally excluded from auto-init.
- `flynn_onboarding.jinja` relocated from `custom_templates/` root to
  `custom_templates/zenos_ai/` namespace.

---

### Scheduler Bug Fixes

Two bugs fixed in `zen_dojotools_scheduler`:

**Bug 1 — Heartbeat literal:** `trigger_entity_id` variable had an `else: ''` branch
that rendered as the literal string `"''"` (two single quotes) rather than an empty
string. Removed — block is now guardless, returning an empty string naturally when no
entity is present.

**Bug 2 — Kata resolver race:** `_subscribers` loop could fire at `ha_start` with
`_kata_cab = 'unknown'` if the kata resolver sensor settled after the dojo resolver.
Guard added at the top of `_subscribers`: returns `[]` immediately if either `_dojo_cab`
or `_kata_cab` is `unknown` / `unavailable`.

---

### Summarizers — Double-Encode Fix

`zen_summary` written by the supersummary path is already a JSON string (from
`monk_response.data`). Applying `| tojson` wrapped it in a second encoding layer,
causing downstream readers to receive a JSON-encoded JSON string rather than a parsed
object. `| tojson` removed from the write path.

---

### History — TemplateSyntaxError

`zen_dojotools_history` error branches (H-0001 CREATE, H-0002 DELETE, H-0003 UPDATE)
had `caller_token` inline in a YAML block dict context. Single quotes inside a Jinja
expression in this context cause `TemplateSyntaxError: unexpected char ''' at N`.

Also removed: invalid `combine()` calls from the HELP and ERROR read branches — these
branches return string or dict values directly, not objects that accept `combine()`.

All three error branches fixed. Help and error read branches fixed. History UAT-passed
on live install 2026-03-21.

---

### Manifest — FG-38 Guard

`AI_Cabinet_VolumeInfo` value may be a JSON string (FG-38). Manifest now applies
two-round normalization (`_vi_raw` / `_vi_norm`) before accessing volume metadata.
Downstream KeyError on non-dict `volumeinfo` values eliminated.

---

### Health — Essence Probe

`essence_probe` macro added to `zen_health_report`. Wired into agent health sensor and
Flynn Gate 3.5. Probes for persona essence presence in the AI user cabinet — surfaces as
a health signal rather than a silent missing-data condition.

---

### Admintools — Cleanup

`zen_admintools_kfc_migration_press` removed from `dojotools_admintools.yaml`. This
script lives in `packages/zenos_ai/private/kfc_migration_press.yaml` for installs that
carry it. Duplicate definition caused HA package load failure (`duplicate key 'alias'`)
on those installs.

---

## Compatibility Notes

- `zen_admintools_reset_template` should be pressed after upgrade — KFC 1.2.0 schema
  adds fields that the existing kata structure template does not include.
- `flynn_onboarding.jinja` path has changed. Any reference to
  `custom_templates/flynn_onboarding.jinja` must be updated to
  `custom_templates/zenos_ai/flynn_onboarding.jinja`.
- No cabinet schema changes requiring migration. Existing drawer content is forward-compatible.

---

## Open / Deferred

- **KFC 1.1 / `meta.enabled` full enforcement** — queued for SP1.
- **SP1 caller_token enforcement** — plumbing complete; activation deferred to SP1.
- **RC1 B3 provisioner: `reset_template` call** — not yet wired into the provisioner
  sequence. Manual press required after upgrade until wired.

---

## HALMark Pre-GA Review

**Sweep date:** 2026-03-22
**Reviewer:** Cayt (dev) / Nyx (live trace)
**Verdict:** Clean — no blocking footguns found.

Full scan of all YAML and Jinja files under `packages/zenos_ai/` and
`custom_templates/zenos_ai/` against the complete ZenOS footgun registry and HALMark
Ratified + Candidate FG set.

### Patterns tested

| FG | Pattern | Result |
|---|---|---|
| FG-1 | `from_json` on `tojson`'d `variables:` values | Clear — no double-decode instances |
| FG-2 | `state_attr().get()` as drawer read path in scripts | Clear — all occurrences are in Jinja template context where FileCabinet cannot be called; all have downstream FG-38 normalization |
| FG-3 | `\| sort(attribute=x, default=0)` | Clear — not present |
| FG-4 | `trigger \| tojson` | Clear — not present |
| FG-5 | Dict variable in `value:` field without `\| tojson` | Clear — not present |
| FG-6/38 | `.get('value')` chains without `is mapping` guard | Clear — all instances use canonical two-round or three-round normalization with explicit `is mapping` guards |
| FG-7 | `\| default('')` without `true` on potentially-None sources | Clear — not present |
| FG-8 | `label_entities()` in non-trigger sensor | Clear — one instance in `sensor_helpers.yaml` is label-scoped enumeration (occupancy timers), not cabinet resolution; re-evaluates on state change |
| FG-9/35 | `automation.reload` | Clear — present in `systemtools` only, documented, preflight-guarded |
| FG-9/36 | `homeassistant.reload_all` | Clear — present in `systemtools` only, documented, preflight config check wired |
| FG-10 | Non-atomic label delete→add without rollback | Clear — not present; Labels UPDATE now uses confirm gate (Meridian) |
| FG-11 | FG-38 `.value` fallback-to-self without `is mapping` | Clear — all fallback-to-self patterns have subsequent `is mapping` check |
| FG-12 | `TemplateState` in `variables:` treated as non-string | Clear — not present |
| FG-13 | `range()` over 100000 | Clear — one `range(0, 100000)` in utilities is exactly at the limit |
| FG-14 | `>-` block scalar with bare digits in string compare | Clear — not present |

### Intentional patterns confirmed non-issue

- `flynn.yaml` — `in_fam | from_json | tojson` in a `value:` cabinet write: correct FG-5 safety pattern, not a roundtrip bug.
- `command_interpreter.jinja` — `{ ... } | tojson | from_json` dict construction: intentional `TemplateState` stripping, not accidental double-encode.
