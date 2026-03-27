# ZenOS-AI Troubleshooting Guide — 4.5.5 'Ready Player Two'

*Gauges → Kill Switches → Repair Tools. Start at the top, work down.*

---

## How to Use This Guide

ZenOS-AI is self-healing. Most problems resolve on their own once the right condition is fixed. This guide is for when they don't.

The structure mirrors the system's own repair logic: **read the gauges first**, then **use the least invasive tool that fixes it**. The repair sequence is ordered from safest to most destructive. Don't skip ahead.

---

## 1. Gauges — Read These First

### Health Sensor Stack

Check these in order. A problem at the bottom propagates up.

| Sensor | What It Tells You | Where to Check |
|---|---|---|
| `sensor.zen_label_health` | Labels exist and are assigned | Developer Tools → States |
| `sensor.zen_cabinet_health` | Cabinet entities initialized | Developer Tools → States |
| `sensor.zen_monastery_health` | Schemas seeded, summaries fresh | Developer Tools → States |
| `sensor.zen_summarizer_health` | Ninja Summarizer heartbeat | → `ai_task_entity`, `last_timestamp` attrs |
| `sensor.zen_supersummary_health` | SuperSummary freshness | → `monk_status`, `last_timestamp` attrs |
| `sensor.zen_flynn_health` | Infrastructure rollup | → `current_gate`, `next_step` attrs |
| `sensor.zen_agent_health` | Is Friday bootable | → `roster` attr (per-gate status per agent) |

**Start with `sensor.zen_agent_health` → `roster`.** It tells you exactly which gate is blocking each agent and what's missing. If you see `friday won't wake up`, this sensor explains why in one attribute read.

### Resolver Sensors

| Sensor | Healthy State | Problem State |
|---|---|---|
| `sensor.zen_dojo_cabinet_resolved` | `sensor.zenos_dojo_cabinet` | `unavailable` |
| `sensor.zen_kata_cabinet_resolved` | `sensor.zenos_kata_cabinet` | `unavailable` |
| `sensor.zen_system_cabinet_resolved` | `sensor.variables` | `unavailable` |

If any resolver shows `unavailable`, Flynn's Gate 3a will stall. Resolvers evaluate on boot, every 15 minutes, and on `zen_resolver_refresh` event.

### Flynn Gate States

| `zen_label_health` | Flynn Gate | What's Needed |
|---|---|---|
| `critical` | Gate 0 | Labels don't exist — restart HA after Flynn creates them |
| `warn` | Gate 1 | Labels exist but aren't assigned — Flynn self-resolves |
| `ok` | Pass | — |

| `zen_cabinet_health` | Flynn Gate | What's Needed |
|---|---|---|
| `error` / `critical` | Gate 2 (hard stop) | One or more cabinets uninitialized — operator action required |
| `warn` | Gate 2 (non-blocking) | Legacy schema detected — upgrade available, system fully operational |
| `ok` | Pass | — |

| `zen_monastery_health` | Flynn Gate | What's Needed |
|---|---|---|
| `critical` | Gate 3 (full bootstrap) | Schema missing or summary/cabinets dead |
| `warn` | Gate 3 (schema seed only) | Schema seed runs, content bootstrap skipped |
| `ok` | Pass | — |

---

## 2. Kill Switches — Immediate, Non-Destructive

Use these to stop or restart the summarization pipeline without touching anything else. Safe to toggle at any time.

| Entity | Default | Effect |
|---|---|---|
| `input_boolean.zen_summarizers_enabled` | `off` | **Master** — off stops both summarizers immediately |
| `input_boolean.zen_supersummarizer_enabled` | `off` | Off stops SuperSummary only |
| `input_boolean.zen_ninja_summarizer_enabled` | `off` | Off stops Ninja Summarizer only |

**The summarizers ship disabled by default.** Enable them once you've confirmed your AI task entity points at a local model (see install guide). Master is checked first. Individual switches only apply when master is on.

When a switch is re-enabled, `automation.zen_pipeline_autofire_on_enable` fires the appropriate force event immediately — no waiting for the next scheduled run.

**Toggle from:** Settings → Helpers, or any dashboard card.

---

## 3. Repair Tools — Graduated Severity

Work through this list from top to bottom. Try the least invasive fix first.

### Step 1 — Kick the Resolvers (safest, try first)

```
zen_resolver_refresh  (fire as HA event)
```

Fires the resolver sensors to re-evaluate immediately without touching labels, cabinets, or schemas. Use this after any label change, after a reload, or when resolver sensors show `unavailable` after a clean boot.

**How:** Developer Tools → Events → Event type: `zen_resolver_refresh` → Fire Event.

---

### Step 2 — Reseed Schemas

```
script.zen_admintools_reset_template
```

Re-presses `kfc_template` into the Dojo cabinet and `zen_template` into the Kata cabinet. Fully idempotent — safe to re-run. Does not touch any other drawer.

**Use when:** `sensor.zen_monastery_health` reports schema missing or corrupt, or Flynn's Gate 3b keeps firing on every boot.

---

### Step 3 — Restamp Prompt Substrate

```
script.zen_admintools_zenos_prompt_loader
  cortex_version: latest   (29 = GA Ninja Fusion, 30 = Living Index opt-in, 27 = RC2)
```

Reloads Cortex, Directives, and Purpose into the system cabinet. Use after an upgrade or if Friday's behavior has drifted from expected.

**Use when:** `sensor.zen_agent_health` shows `system_purpose` or `system_directives` gate failing.

---

### Step 4 — Soft Label Reset

```
script.zen_dojotools_labels
  action_type: reset
  confirm: true
```

Wipes all entity assignments from every `zen_` label. Labels survive — only the tagging is cleared. `zen_resolver_refresh` fires automatically. Flynn re-assigns via Gate 1 on the next health sensor change.

**Use when:** Labels exist but entities aren't tagged correctly. Resolver sensors are `unavailable` and a `zen_resolver_refresh` didn't fix it.

---

### Step 5 — Nuclear Label Reset

```
script.zen_admintools_reset_labels
  confirm: true
```

⚠️ **Deletes all `zen_` labels and all assignments. No undo.**

Non-zen labels (`home`, `media_players`, `hot_tub`, etc.) are never touched. Flynn rebuilds the full label set automatically — Gate 0 recreates, Gate 1 reassigns. No HA restart required on HA 2024.x+.

**Use when:** Label IDs are corrupt, have wrong names, or soft reset didn't resolve the problem.

---

### Step 6 — Single Cabinet Reset

```
script.zen_admintools_cabinetadmin
  mode: reset
  target_cabinet: sensor.zenos_<cabinet_name>
```

Wipes all drawers from a single cabinet. Use when one specific cabinet has corrupt data but others are fine.

**Use when:** One cabinet is causing errors but the rest of the system is healthy. Check `sensor.zen_cabinet_health` → `slot_entities` to confirm which cabinet is the problem.

---

### Step 7 — Nuclear Cabinet Reset (Last Resort)

```
# Run in order:
1. script.zen_admintools_reset_labels     (confirm: true)
2. script.zen_admintools_cabinetadmin     (op: reset_all, confirm: true)
```

⚠️ **Wipes all Ring-0 canonical cabinets + all zen_ labels. No undo.**

Step 1 nukes labels. Step 2 wipes all 14 Ring-0 cabinets, reseeds schemas via `reset_template`, and fires Flynn bootstrap. User cabinets (`sensor.nathan_curtis_s_cabinet`, etc.) are never touched. Order matters — labels first, cabinets second.

Flynn handles the full rebuild sequence automatically.

**Use when:** Nothing else worked, or you're doing a clean reinstall.

---

## Common Symptoms → Where to Start

| Symptom | Start Here |
|---|---|
| Friday won't wake up | `sensor.zen_agent_health` → `roster` attr |
| Summaries stopped | Check kill switches — ships `off` by default; all three must be `on` to run |
| Summarizer health shows `disabled` | Kill switch is off — intentional state. Enable the relevant switch; autofire restarts it. |
| Summaries are stale | `sensor.zen_supersummary_health` → `monk_status` |
| Scheduler not firing | `sensor.zen_summarizer_health` → `ai_task_entity`, `last_timestamp` |
| Flynn stuck at boot | `sensor.zen_flynn_health` → `current_gate`, `next_step` |
| Resolver sensors unavailable | Fire `zen_resolver_refresh`. If still stuck → Step 4 (soft label reset) |
| Labels not assigning | `sensor.zen_label_health` → `missing_label_ids`, `unassigned_label_ids` |
| Cabinet missing or corrupt | `sensor.zen_cabinet_health` → `missing_cabinets` → Step 6 |
| Schema missing / Gate 3 keeps firing | Step 2 (`reset_template`) |
| Full reinstall needed | Step 7 (nuclear sequence) |

---

## Notes

- **Flynn is self-healing.** After any repair action, wait for the relevant health sensor to update — Flynn re-engages automatically. You rarely need to trigger it manually.
- **`zen_resolver_refresh`** is always safe to fire. It re-evaluates resolver sensors without changing anything.
- **Kill switches** are non-destructive. Turning the summarizers off and back on is always safe.
- **Backup before nuclear ops.** Steps 5–7 have no undo. If your cabinet data matters, export it via `zen_admintools_cabinetadmin mode: inspect` before proceeding.
