# Release Notes — ZenOS-AI 4.5.5 'Ready Player Two'

**Released:** 2026-03-26
**Tag:** `v4.5.5`
**Follows:** Meridian (`8fbf10c`, 2026-03-21)
**UAT:** Passed — 21/21 gates green (`uat_rp2_2026-03-26.md`)

---

## Summary

Ready Player Two is the identity and lifecycle release. Three major feature areas:

1. **Cabinet provisioning system** — mount-aware state machine, RP2 provisioner, expansion slots, scroll ceremony. Cabinets now have a proper `online_unmounted` / `online_mounted` lifecycle instead of raw HA template state.
2. **Warmup as first-class sensor state** — boot noise eliminated. The 5-minute post-restart window is now a named state (`warmup`) with its own sensor values, UX messaging, and explicit expiry signal.
3. **Identity model v4.5.0** — full family/household group management. Fifteen modes on `zen_dojotools_identity`. Household/family/member graph with principal slots, delegation links, depth-2 security resolution, and an enriched identity manifest with tree view.

---

## Cabinet Provisioning System

### Mount-Aware State Machine

Cabinets now track lifecycle as a first-class sensor state:

| State | Meaning |
|---|---|
| `online_unmounted` | Cabinet exists, not assigned to an identity slot — sits in stacks, ready to provision |
| `online_mounted` | Cabinet is active and assigned to an identity slot via a type label |

The manifest and resolver sensors gate on `online_mounted` only — stacks cabinets are invisible to the identity layer until provisioned.

### RP2 Provisioner (`zen_dojotools_provisioner` v1.1.0)

Three-mode provisioner for full identity lifecycle management:

| Mode | What It Does |
|---|---|
| `provision` | Pulls an `online_unmounted` cabinet into service. Validates GUID uniqueness, applies type label, mounts, preloads profile data, fires identity manifest rebuild. Rolls back on health gate timeout. For `ai_user` cabinets: seeds a blank `zenai_essence` at provision time. |
| `deprovision` | Dismounts cabinet, strips type label, returns it to stacks. Blocked if cabinet holds a `zen_default_*` label. |
| `replace` | Deprovisions `replace_cabinet`, then provisions `target_cabinet` with the same `cab_type`. Atomic swap. |

GUID uniqueness is a hard stop — a cabinet with an existing GUID cannot be re-provisioned. Boot wipe guard: three-layer protection prevents cabinet variable wipe on HA restart.

### Expansion Cabinet Slots

Five expansion cabinet sensor definitions added. All 20 cabinet entity IDs aligned with friendly names. Vault intentionally excluded from auto-init.

### Scroll Ceremony (`zen_scroll`)

Read-only drawer access pattern for mounted cabinets. Formalizes the distinction between read and write access at the tool level. One-way lock via `zen_scroll` label in `label_input` — hard block, no force bypass.

---

## Warmup as First-Class Sensor State

`warmup` is now a named sensor state, sitting between `error` and `warn` in the priority chain:

```
Priority: critical > error > warmup > warn > disabled > ok
```

During the boot window, degraded states consistent with normal startup surface `warmup` instead of `warn`. Dashboard shows **ZenOS — WARMUP** with a "settling after restart, no action needed" message. `zen_warmup_timer` fires `warmup_expired` at 5 minutes post `ha_start`. Flynn re-evaluates all gates on expiry.

| Boot phase | Dashboard header | System Ready | Next step |
|---|---|---|---|
| 0–5 min post-restart | ZenOS — WARMUP | on | "Settling after restart — no action needed" |
| 5+ min, all ok | ZenOS — OK | on | All gates green |
| 5+ min, cab warn | ZenOS — WARN | on | "Cabinet upgrade available — not urgent" |
| 5+ min, real error | ZenOS — ERROR | off | Actionable message |

`binary_sensor.flynn_system_ready` monastery `unavailable` during warmup is non-blocking. After warmup expires, `unavailable` on monastery is a real failure.

---

## Identity Model v4.5.0

### New Modes

`zen_dojotools_identity` grew from a 3-mode resolver into a full identity and group management tool. Twelve new modes added; `resolve`, `prompt`, and `build_identity_manifest` are unchanged.

| Mode | What It Does |
|---|---|
| `household_add_family` | Wires a family into the household members list |
| `household_remove_family` | Removes a family from the household members list |
| `household_add_member` | Adds user or AI; fills HoH/prime principal slot on first add |
| `household_remove_member` | Removes user or AI; warns if principal slot is now stale |
| `set_principal` | Replaces the HoH or prime AI slot for any container cabinet |
| `family_add_member` | Adds user, AI, or sub-family; sets `default_family_guid` on first join |
| `family_remove_member` | Removes member; clears `default_family_guid` if was default |
| `link_partners` | Bidirectional delegation link — `acls.partner[]` on both entities |
| `unlink_partners` | Symmetric removal from `acls.partner[]` on both sides |
| `set_default_family` | Explicitly patches `default_family_guid` on a member's VolumeInfo |
| `membership` | Tree view for containers; reverse lookup for members |
| `is_member` | Depth-2 boolean membership check `{is_member, depth: 1|2|null}` |

### Principal Slots

Each container has two privileged occupant slots:
- `acls.owner` — Head of Household (filled by first `user` added)
- `acls.partner[role=prime]` — Prime AI partner (filled by first `ai_user` added)

Slots block re-entry once filled. Use `set_principal` to transfer. Remove operations warn via `principal_warning` field rather than blocking.

### Delegation Model

`acls.partner[]` is the delegation authority record. Works for any entity pair (user↔user, user↔AI, AI↔AI). No token is issued without an explicit allow — the link records who has delegation authority, it does not grant it automatically.

### Enriched Identity Manifest

`build_identity_manifest` writes `{roster, tree}` to the `zen_identity_manifest` drawer. The `tree` is the depth-2 household membership structure including principal slots, member lists, and nested families. Auto-rebuild fires after all 6 roster-change operations.

### Flynn Bootstrap Wiring

`flynn_bootstrap_content` now wires household membership at bootstrap: `household_add_member` for primary user (HoH slot), `household_add_member` for AI (prime slot), `link_partners` bidirectional, `build_identity_manifest` for initial manifest. All calls are idempotent.

---

## Profile Editor (`zen_dojotools_profile_editor` v4.5.5)

Universal read/write for ZenOS identity cabinets. Targets `ai_user`, `household`, `user`, `family`.

- Read path returns live profile data for all cabinet types (FG-38 guards on all 6 read sites)
- Write path is self-healing — `mode: write` with no fields on a missing or corrupt `zenai_essence` seeds from scratch
- `ai_user` write assembles three-layer essence: `core` (identity anchor), `jacket` (persona), `companion` (familiar), `environment`
- `mode: sign` — MD5 clear-sign of core and jacket; upgrade path to OIDC-backed sig in SP1
- `mode: restore` — roll back to previous snapshot (`zenai_essence_prev`)
- Autosign uses in-memory assembled value, not a post-write state read (race condition fixed)

---

## FileCabinet v4.5.5

- Syscab (`sensor.zenos_system_cabinet`) wire-level lockout — hard RO, no `force_action` bypass
- `_` key guard — reserved prefix blocked by default; `force_action: true` bypasses for system drawer writes
- Slug preserves leading underscore when `force_action: true`
- `cabinet_ro` semaphore + `set_cabinet_ro` mode — drawer-level RO via `zen_scroll` label
- `relabel` action — update `_label_index` without re-parsing value
- `create_label` flag — auto-create unknown HA labels before write
- `clone` action with `transfer_guid` Highlander mode
- `set_mount` / `remove_mount` — drawer-level cross-cabinet links
- `mount_cabinet` / `dismount_cabinet` — cabinet state flip with zen_event
- Wildcard read: `key: '*'` returns all drawer values
- Volume Redirector: `mode: queued, max: 40`; clear routing fix; `confirm: true` on all 19 clear blocks

---

## CabinetAdmin

- Renamed `zen_admintools_cabinetadmin_stamp` → `zen_admintools_cabinetadmin_factory`
- `mode: init` — virgin/good/potentially_bad classifier; delegates to factory for stamp
- `mode: hammer` — `confirm_init: true` for auto-init after wipe; `init_guid` in result
- `mode: help` — full structured operation guide (human and AI-targeted)
- `cabinet_boot_touch` fired after all VolumeInfo writes — RP2-aware state re-derivation
- `guid` field on both factory response paths (init and repair)

---

## Cabinet Health

`warn` is non-blocking throughout the stack. Legacy schema (`needs_touch`) is a non-blocking warn — data intact, one-way admin touch to migrate. `binary_sensor.flynn_system_ready` stays `on` with a `warn` cabinet.

All health sensors carry a `reason` attribute. `sensor.zen_cabinet_health` gains `cabinet_states` — per-cabinet state map readable at runtime.

---

## Prompt / Cortex

- Cortex 32 — True Voice — is now the GA default (`latest`)
- Cortex 31 — Signal Frame
- `render_prompt()` single entry point; `zen_os_1rc.jinja` renamed to `zen_os_1.jinja`
- Environment schema + native wake path
- `identity_resolve_source` safety guard skips roster results (was injecting `identity: {name: ""}` on no-target calls)

---

## Infrastructure

- Dispatcher (`zen_event` dispatch layer) — event-driven tool dispatch, reduces automation boilerplate
- Inspect + Index linked help contract
- Labels: slugify fix on tag/untag service calls
- Index: slugify fix on `label_1`/`label_2`; `*` operator added to enum
- Scribe v1.1.0 bundled; `zen_dojotools_kungfu_writer` retired

---

## FG Coverage

All code swept against full ZenOS footgun registry and HALMark Ratified + Candidate FG set prior to UAT sign-off.

Notable FG fixes in this release:
- **FG-38** — FileCabinet drawer `.value` may be JSON-encoded string. Three-round guard applied at all VolumeInfo read sites across identity, manifest, filecabinet, and profile editor
- **FG-40** — `variables:` block `| tojson` auto-parsed by HA; `else []` on dedup guards silently discarded native lists. All three dedup guards (household_add_family, household_add_member, family_add_member) fixed
- **Autosign race** — post-write `state_attr` returned stale pre-write state; autosign signed empty seed and overwrote real data. Fixed to use in-memory assembled value
- **Manifest FG-38** — manifest embedded template had fallback-to-self on VolumeInfo; crashed with `show_stacks: true`
- **Profile read empty** — all read paths returned empty fields due to fallback-to-self value-unwrap

---

## Compatibility Notes

- **Existing installs:** Flynn bootstrap membership wiring is idempotent — `household_add_member` blocks on slot-occupied for installs with existing principals. Re-running bootstrap is safe.
- **Identity manifest shape change:** `zen_identity_manifest` changed from `{roster, count}` to `{roster, tree}`. `identity_manifest_loader()` handles both shapes. Old manifests load as `status: ok_legacy` until rebuilt.
- **Legacy cabinets:** Pre-RC2 cabinets in `Variables` state are recognized as `needs_touch` — non-blocking warn, data intact, one admin touch to migrate.

---

## Deferred to SP1

- `caller_token` enforcement (plumbing complete, activation deferred)
- KFC 1.1 — `meta.enabled` full enforcement, `requires.cert` / `requires.level`
- Security architecture — zen_auth, TGT, SekretSQRL
- Cabinet import/export — file I/O arch decision unresolved
