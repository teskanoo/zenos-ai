# ZenOS-AI Patch 4.5.6

*Post-GA identity and alertmanager correctness patch*

**Branch:** `patch/post-ga-identity-fixes` → merged to `main`
**Base:** 4.5.5 'Ready Player Two'
**UAT:** Passed — Nyx on live H:\ install (2026-03-26/27)

---

## Summary

Nine correctness fixes and two admin improvements. No architecture changes. All fixes are idempotent — existing installs safe.

---

## Changes

### Identity — `unlink_partners` variable name fix

**File:** `dojotools/dojotools_identity.yaml`

`unlink_partners` computed `_a_vi_updated` / `_b_vi_updated` but the two cabinet write events referenced `_user_vi_updated` / `_ai_vi_updated` — names that never existed in that block. Both writes were silent no-ops. ACL removal had never worked.

**Fix:** Variable references in the two write events corrected to match the computed names.

---

### Identity — Bootstrap family graph wiring

**File:** `flynn.yaml`

Cold bootstrap (`flynn_bootstrap_content`) wired default user and AI into the household and linked them as partners, but never called `household_add_family` or `family_add_member`. The default family cabinet existed but was an orphan — not connected to the household graph. The identity manifest correctly reflected this gap, which caused the security roster to drop family-contained entities as invalid.

**Fix:** Flynn bootstrap now calls `household_add_family` + `family_add_member` (user + ai_user) after `link_partners`, before the final `build_identity_manifest`. All three calls are guarded (skips if cabinet unavailable) and idempotent.

**Existing installs:** Run `zen_dojotools_systemtools` with `tool: run_repair`, `repair_action: identity_family_repair_4_5_6`. Or run `script.zen_maint_4_5_6_identity_family_repair` directly from Developer Tools.

---

### Identity — Name resolution fallback (#110)

**File:** `dojotools/dojotools_identity.yaml`

The identity tree builder and manifest used a two-step name resolution chain (`friendly_name → entity_id`). Pre-RP2 VolumeInfo entries stored the display name in a `name` field, not `friendly_name` — these resolved to bare entity IDs in the membership tree and manifest output.

**Fix:** Resolution chain extended to `friendly_name → name → entity_id` at 8 locations: `build_identity_manifest`, membership tree renderer, and `member_lookup`/`is_member` paths.

---

### Home Mode — Timer seeding without `initial:` (#112)

**Files:** `dojotools/zen_home_mode.yaml`, `flynn.yaml`

The 10 `input_datetime` schedule and quiet/work-hours helpers carried `initial:` values. HA evaluates `initial:` on every restart — any time you changed a helper time, the next restart would reset it to the `initial:` default. User-configured schedules were silently discarded.

**Fix (zen_home_mode.yaml):** All 10 `input_datetime` helpers — `initial:` removed.

**Fix (flynn.yaml):** Flynn bootstrap now seeds each timer individually on first boot (guard: `in ['', 'unknown', 'unavailable']`). Each timer has its own `if` block — partially-configured installs are not disrupted.

Default times seeded:

| Helper | Default |
|---|---|
| `zen_am_start` | 06:00 |
| `zen_morning_start` | 08:00 |
| `zen_daytime_start` | 10:00 |
| `zen_evening_start` | 17:00 |
| `zen_night_start` | 21:00 |
| `zen_late_night_start` | 23:00 |
| `zen_quiet_hours_start` | 23:00 |
| `zen_quiet_hours_end` | 06:00 |
| `zen_work_hours_start` | 09:00 |
| `zen_work_hours_end` | 17:00 |

---

### AlertManager — Persistence fix (#113)

**File:** `dojotools/dojotools_alertmanager.yaml`

Both write paths (alert fire and alert clear) used `action_type: write`, which is not a valid FileCabinet action type. FileCabinet returned a help response on every call — `_zen_active_alerts` was never persisted. Every AlertManager run treated the active-alerts dict as empty, so every `alert_fire` was treated as a new alert and fired the notification again. The intended fire-once deduplication never worked.

**Fix:** Both paths updated to `action_type: create` + `force_action: true` (create-or-overwrite). Deduplication is now functional.

---

### SystemTools — `run_repair` admin entry point

**Files:** `dojotools/dojotools_systemtools.yaml`, `maint/maint_4_5_6.yaml` *(new)*

Adds a `run_repair` tool to SystemTools with a `repair_action` picklist. Dispatches to versioned one-time repair scripts in `packages/zenos_ai/maint/`.

Repair scripts are **not AI-accessible** — they are operator-run via SystemTools or directly from Developer Tools → Services. Each script includes a version declaration and idempotency guarantee.

**4.5.6 repair script:** `zen_maint_4_5_6_identity_family_repair` — wires default family into household graph for installs bootstrapped before 4.5.6.

---

### Identity — GUID id-preference (`id` over legacy `guid`)

**File:** `dojotools/dojotools_identity.yaml`

RP2 provisioner stamps identity GUIDs as `id` in VolumeInfo. Legacy pre-RP2 cabinets carry the old `guid` field. All `.get('guid', '')` reads in the identity tool were reading the legacy field directly — installs with both fields present would pick up the stale `guid` instead of the canonical `id`, producing GUID mismatches in roster entries.

**Fix:** 7 distinct call shapes across 23 call sites updated to `id`-first with `guid` fallback: `obj.get('id', obj.get('guid', ''))`. Provisioner already writes `id` — no provisioner-side change needed.

---

### Identity — Household root name profile-first resolution

**File:** `dojotools/dojotools_identity.yaml`

The household root `name` field in tree output (both `resolve`/`build_identity_manifest` and `membership` mode) was reading directly from VolumeInfo `friendly_name → name → entity_id`. Family and user roots already used a profile-drawer-first chain (`_family_profile` / `_user_profile`). Household root did not.

**Fix:** Two sites updated to read `_household_profile` drawer first, fall back to VolumeInfo. `membership` mode uses a broader fallback chain (`_household_profile → _family_profile → VolumeInfo`) since `_root_entity` can be either cabinet type.

---

### AdminTools — `zen_admintools_run_repair` + repair moved from SystemTools

**Files:** `dojotools/dojotools_admintools.yaml`, `dojotools/dojotools_systemtools.yaml`, `maint/maint_4_5_6.yaml`

`run_repair` was in SystemTools (AI-accessible). Design decision: repair scripts are destructive (teardown/re-wire cycles) and must not be fireable by an unattended LLM. Moved to AdminTools behind a hard `confirm_action: true` gate.

**Changes:**
- `dojotools_systemtools.yaml` — `run_repair` tool option + condition block + `repair_action` field removed entirely
- `dojotools_admintools.yaml` — `zen_admintools_run_repair` added: human-confirmed passthrough, `choose:` dispatch (no sequential-if bug), passes `entity_id` to `stamp_cab_guid_4_5_6`
- `maint/maint_4_5_6.yaml` — Two new repair scripts added (see below)

**Via Developer Tools → Services → `script.zen_admintools_run_repair`:**
```yaml
repair_action: identity_family_repair_4_5_6 | stamp_cab_guid_4_5_6 | roster_guid_repair_4_5_6
confirm_action: true
entity_id: sensor.your_cab   # stamp_cab_guid_4_5_6 only
```

---

### Maint — `zen_maint_4_5_6_stamp_cab_guid` + `zen_maint_4_5_6_roster_guid_repair`

**File:** `maint/maint_4_5_6.yaml`

Two new repair scripts for pre-provisioner installs where `VolumeInfo.id` is empty, causing blank GUID fields in roster entries.

**`zen_maint_4_5_6_stamp_cab_guid`** — stamps a new UUID v4 into a single cabinet's `VolumeInfo.id`. Idempotent (no-op if `id` already present). UUID generation uses `range(0, 65536)` splits to avoid HA sandbox `MAX_RANGE=100000`. Fires `cabinet_boot_touch` after stamp.

**`zen_maint_4_5_6_roster_guid_repair`** — full GUID repair for installs with blank ids on all 4 default cabs. Stamps GUIDs on all 4 (household/family/user/ai_user), then does a full remove+re-add cycle on all family and household members so roster entries get real GUIDs baked in. Reads current member lists dynamically — safe for any topology.

**Affected installs:** Pre-provisioner installs (bootstrapped before RP2). Run `roster_guid_repair_4_5_6` once, verify manifest shows real GUIDs.

---

### Maint — `repair_fam_cab_volumeinfo` (standalone script)

**File:** `maint/repair_fam_cab_volumeinfo.yaml`

One-shot repair for the three-rogue-key VolumeInfo incident on the family cabinet. Caused by pre-guard writes that created three keys instead of one canonical `AI_Cabinet_VolumeInfo`:

| Key | Content |
|---|---|
| `AI_Cabinet_VolumeInfo` | Stomped — `{"flags": {"gc_eligible": false}}` |
| `ai_cabinet_volumeinfo` | Good data — full VolumeInfo with real `id` |
| `_ai_cabinet_volumeinfo` | Unknown content |

Script reads good data, validates `id` present, writes back to canonical key, clears both rogue keys. `dry_run: true` default — must pass `dry_run: false` to execute. Step 1 verify gate before cleanup.

Not wired into `zen_admintools_run_repair` — surgical script, run directly from Developer Tools.

---

### Cabinet Health — Legacy warn triage by slot presence

**File:** `custom_templates/zenos_ai/zenos_health.jinja`, `sensors/zenos_cabinet_health.yaml`

`zen_cabinet_health` emitted `warn` for every cabinet with state `Variables` in `all_cabs` (parent `Zen Cabinet` label), including retired stubs whose slot-specific labels had been intentionally pulled. On installs with many decommissioned legacy cabs this produced persistent noise warn state unrelated to anything needing migration attention.

**Fix:** `legacy_schema` detection in both `zen_cabinet_health_state()` and `zen_cabinet_health_reason()` now builds a slotted set first (entities assigned to any required or optional slot label) and only flags legacy-schema cabs that are in that set. Unslotted cabs — intentionally retired, parent label only — are silent.

Behavior unchanged for any cabinet that currently holds a slot assignment.

---

## PRs Merged

| PR | Author | Summary |
|---|---|---|
| #115 | teskanoo | Scribe `label_description` → `label_description_generic`, `version` → `version_semantic`, person selector fix |
| #116 | teskanoo | AdminTools/filecabinet caller_token normalize, is_relabel/is_write_action moved earlier |
| #117 | teskanoo | FileCabinet normalize-at-top refactor |

---

## Files Changed

| File | Change |
|---|---|
| `dojotools/dojotools_identity.yaml` | unlink_partners fix, name fallback fix, GUID id-preference (23 sites), household root profile-first name |
| `dojotools/dojotools_alertmanager.yaml` | action_type fix (fire + clear paths) |
| `dojotools/dojotools_systemtools.yaml` | run_repair removed (moved to admintools) |
| `dojotools/dojotools_admintools.yaml` | PR #116 caller_token normalize + `zen_admintools_run_repair` |
| `dojotools/dojotools_filecabinet.yaml` | PR #117 normalize-at-top refactor |
| `dojotools/zen_home_mode.yaml` | Strip initial: from all 10 input_datetime helpers |
| `dojotools/dojotools_scribe.yaml` | PR #115 parameter renames + person selector fix |
| `flynn.yaml` | Bootstrap family graph wiring + timer seeding |
| `maint/maint_4_5_6.yaml` | Three repair scripts: identity_family_repair, stamp_cab_guid, roster_guid_repair |
| `maint/repair_fam_cab_volumeinfo.yaml` | New — family cabinet VolumeInfo rogue key repair |
| `custom_templates/zenos_ai/zenos_health.jinja` | Legacy warn triage — slot presence gate |
| `sensors/zenos_cabinet_health.yaml` | Version bump |

---

## Verification

1. **unlink_partners** — call with linked pair, confirm both VolumeInfo `acls.partner` lists updated
2. **Bootstrap wiring** — fresh install or manual bootstrap: `sensor.zen_default_family_cabinet_resolved` should appear in household `members.families`; user/AI VolumeInfo should have `acls.family` entries
3. **Timer seeding** — fresh install: helpers should seed to defaults; restart: custom values should persist unchanged
4. **AlertManager** — call `alert_fire` twice with same key: second call should be a no-op (no duplicate notification)
5. **run_repair** — call `zen_admintools_run_repair` with `repair_action: identity_family_repair_4_5_6` on a broken install, confirm `status: ok` and family graph wired
6. **GUID id-preference** — `zen_dojotools_identity → resolve`: confirm roster entries show `id` field (not stale `guid`) on RP2+ installs
7. **stamp_cab_guid** — call `zen_admintools_run_repair` with `repair_action: stamp_cab_guid_4_5_6` + `entity_id: sensor.<blank-cab>`: confirm `was_stamped: true`, re-run confirms `was_stamped: false`
8. **roster_guid_repair** — run on pre-provisioner install: confirm all 4 default cabs have `id` in VolumeInfo, roster entries show real GUIDs
9. **fam_cab_volumeinfo repair** — run dry first: confirm planned ops in system log. Live run: confirm `AI_Cabinet_VolumeInfo` restored, rogue keys cleared
