# ZenOS-AI Health Sensors — 4.3.0 'Meridian'

*System observability stack — labels, cabinets, cognition, agents*

---

## Overview

ZenOS-AI ships a layered health monitoring system. Every sensor reports one of five states:

| State | Meaning |
|---|---|
| `ok` | All checks pass |
| `disabled` | Pipeline intentionally off via kill switch — not an error |
| `warn` | Degraded but functional — system can run, something needs attention |
| `error` | Functional impaired — action required |
| `critical` | Bootstrap blocked — Flynn will engage |

State ladder (severity): `critical > error > warn > disabled > ok`. `disabled` is a clean, intentional state — not a fault. Turning a kill switch back on returns the sensor to `ok` and triggers an auto-refire of the pipeline (see Kill Switches below).

Sensors are arranged in a dependency stack. Label health feeds cabinet health, which feeds monastery health, which feeds Flynn health, which feeds agent health. A problem at the bottom propagates up.

---

## Sensor Stack

```
sensor.zen_label_health          ← updates every 1 minute
         │
         └──→ sensor.zen_cabinet_health
                      │
                      ├──→ cabinet resolvers (dojo / kata / system)
                      │
                      └──→ sensor.zen_monastery_health
                                   │
                                   ├──→ sensor.zen_summarizer_health
                                   └──→ sensor.zen_supersummary_health
                                              │
                      ┌────────────────────────┘
                      │
                      └──→ sensor.zen_flynn_health
                                   │
                                   ├──→ binary_sensor.flynn_system_ready
                                   └──→ sensor.zen_agent_health
```

---

## sensor.zen_label_health

**What it checks:** All 16 required ZenOS labels exist in the HA label index and are assigned to at least one entity.

**Update frequency:** Every 1 minute (time_pattern trigger)

| State | Condition |
|---|---|
| `critical` | One or more required labels are missing from the label index |
| `error` | One or more required labels have `unavailable` or `unknown` entities assigned |
| `warn` | One or more required labels exist but have 0 entities assigned |
| `ok` | All labels present and healthy |

**Key attributes:**

| Attribute | Contents |
|---|---|
| `missing_label_ids` | Labels not found in the index at all |
| `unassigned_label_ids` | Labels that exist but have no entities tagged |
| `unhealthy_label_ids` | Labels with unavailable/unknown entities |
| `healthy_label_ids` | Labels that pass all checks |
| `slot_to_label` | Canonical slot → label name map (used by Flynn) |

**Required labels (16):** zen_cabinet, zen_system_cabinet, zen_dojo_cabinet, zen_kata_cabinet, zen_default_household, zen_default_household_cabinet, zen_default_family, zen_default_family_cabinet, zen_default_user_cabinet, zen_default_ai_user_cabinet, zen_household_cabinet, zen_family_cabinet, zen_user_cabinet, zen_ai_user_cabinet, zen_history_cabinet, zen_index_cabinet

**Flynn gate:** `critical` → Gate 0 (create labels). `warn` → Gate 1 (assign labels).

---

## sensor.zen_cabinet_health

**What it checks:** Each required cabinet slot has exactly one healthy cabinet entity assigned to it.

**Update frequency:** Continuous (state-based)

| State | Condition |
|---|---|
| `critical` | One or more required cabinet labels are missing from the index |
| `error` | A required slot has multiple cabinets assigned, or a required cabinet is unavailable/unknown |
| `warn` | A required label exists but no cabinet entity is tagged with it |
| `ok` | All required slots have exactly one healthy cabinet |

**Required slots (7):** system, dojo, kata, default_household, default_family, default_user, default_ai_user

**Optional slots (6):** household, family, user, ai_user, history, index

**Key attributes:**

| Attribute | Contents |
|---|---|
| `missing_cabinets` | Required slots with no assigned entity — used by Flynn Gate 2 |
| `invalid_multiples` | Required slots with more than one entity assigned |
| `slot_entities` | `{slot: [entity_ids]}` map for all slots |
| `slot_to_default_entity` | Default sensor entity ID per slot |
| `resolver_suggestions` | Per-slot plain-language action strings |

**Flynn gate:** Not `ok` → Gate 2 (initialize missing cabinets).

---

## sensor.zen_monastery_health

**What it checks:** The cognition pipeline — schema templates present, summaries fresh, active components registered.

**Update frequency:** Continuous (state-based)

| State | Condition |
|---|---|
| `critical` | Dojo or kata cabinet unavailable; OR schema invalid (no `zen_template.value.structure`); OR KF switches active but no `zen_summary` drawer |
| `warn` | Monk status in error/timeout/none; OR summary >15 minutes stale; OR no active Kung Fu switches |
| `ok` | Schema valid, summary <15 min old, monk status ok, ≥1 active switch |

**Staleness thresholds:**
- `>15 min` → warn
- `>30 min` → critical-level warn

**Key attributes:**

| Attribute | Contents |
|---|---|
| `schema_ok` | Whether `zen_template.value.structure` is a valid mapping |
| `active_components` | Entity IDs of Kung Fu System Switches currently `on` |
| `last_timestamp` | Timestamp of most recent `zen_summary` |
| `ninja_summary_status` | Pass-through of `sensor.zen_summarizer_health` |
| `supersummary_status` | Pass-through of `sensor.zen_supersummary_health` |

**Flynn gate:** `critical` → Gate 3 (full content bootstrap). `warn` → Gate 3 schema seed only, no bootstrap.

---

## sensor.zen_summarizer_health

**What it checks:** Whether the Ninja Summarizer (Scheduler) is running and its heartbeat is fresh.

**Update frequency:** Continuous (state-based)

| State | Condition |
|---|---|
| `critical` | Kata cabinet unavailable/unknown/missing |
| `warn` | AI task entity not configured or unavailable; OR `zen_scheduler` drawer missing; OR drawer timestamp >20 minutes stale |
| `disabled` | `input_boolean.zen_ninja_summarizer_enabled` or `input_boolean.zen_summarizers_enabled` is `off` |
| `ok` | AI task configured, scheduler drawer exists, timestamp ≤20 minutes old |

**Key attributes:**

| Attribute | Contents |
|---|---|
| `ai_task_entity` | Current value of `input_text.zenos_ai_task_entity` |
| `last_timestamp` | Timestamp from `zen_scheduler` drawer in kata cabinet |

**If this is `warn`:** Check `input_text.zenos_ai_task_entity` is set and the entity is available. If the drawer is missing or stale, the Scheduler may not be running — check the scheduler automation is enabled and the AI task entity is reachable.

---

## sensor.zen_supersummary_health

**What it checks:** Whether the SuperSummary (High Priestess) is producing fresh summaries.

**Update frequency:** Continuous (state-based)

| State | Condition |
|---|---|
| `critical` | Kata cabinet unavailable/unknown/missing |
| `warn` | Monk status is error/timeout/none/null; OR summary >15 minutes stale; OR summary 15–30 minutes stale |
| `disabled` | `input_boolean.zen_supersummarizer_enabled` or `input_boolean.zen_summarizers_enabled` is `off` |
| `ok` | Monk status ok AND summary ≤15 minutes old |

**Key attributes:**

| Attribute | Contents |
|---|---|
| `monk_status` | From `zen_supersummary_status.value.status` in kata cabinet |
| `last_timestamp` | Timestamp of most recent `zen_summary` |

**If this is `warn`:** Check `monk_status` in Developer Tools. `error` or `timeout` indicates the AI task entity attempted a summary and failed — check inference server logs for context window errors or model availability.

---

## sensor.zen_flynn_health

**What it checks:** Infrastructure rollup — worst state across labels, cabinets, and monastery.

**Update frequency:** Continuous (state-based)

| State | Condition |
|---|---|
| `critical` | Any of label_health, cabinet_health, or monastery_health = `critical` |
| `error` | Any = `error` (none = `critical`) |
| `warn` | Any = `warn` (none higher) |
| `ok` | All three = `ok` |

**Key attributes:**

| Attribute | Contents |
|---|---|
| `current_gate` | Which Flynn gate is currently blocking (`gate_0` through `ok`) |
| `next_step` | Plain-language instruction for the blocking gate |
| `blocking` | Comma-separated list of non-ok subsystems |
| `system_ready` | Pass-through of `binary_sensor.flynn_system_ready` |

Use `current_gate` and `next_step` when troubleshooting a stuck boot — they tell you exactly where Flynn is and what it needs.

---

## sensor.zen_agent_health

**What it checks:** Whether AI agents are bootable — infrastructure + per-agent gate validation.

**Update frequency:** Continuous (state-based)

| State | Condition |
|---|---|
| `critical` | label_health or cabinet_health = `critical` or `error` |
| `error` | label/cabinet = `error`, 0 agents bootable |
| `warn` | Structural gates blocking; or ≥1 agent bootable but monastery ≠ `ok` |
| `ok` | Infrastructure ok, ≥1 agent bootable, monastery ok |

**Agent bootability gates (per agent):**

| Gate | Blocks If |
|---|---|
| `infra` | Flynn infra not ok |
| `conversation_agent` | `input_text.zenos_conversation_agent` empty or unavailable |
| `default_persona` | Cabinet is not labeled `zen_default_ai_user_cabinet` |
| `system_purpose` | System cabinet has no `purpose` drawer |
| `system_directives` | System cabinet has no `directives` drawer |
| `dojo_loaded` | Dojo cabinet missing `kfc_template` |
| `kata_summary` | Kata cabinet has no `zen_summary` drawer |

**Key attributes:**

| Attribute | Contents |
|---|---|
| `roster` | JSON array of all agent objects with per-gate status |
| `agents_ok` | Count of fully bootable agents |
| `active_persona` | Name of the currently selected AI persona |
| `conversation_entity` | Configured conversation agent entity ID |

**This is the sensor to check first when Friday won't wake up.** The `roster` attribute shows exactly which gate is blocking each agent and what's missing.

---

## binary_sensor.flynn_system_ready

**State:** `on` when labels ok AND cabinets ok AND monastery in [`ok`, `warn`]

`warn` monastery is acceptable — schema and cabinets are valid, summarizer may not have run yet. This is the bootstrap eligibility gate, not a health gate.

---

## UI Selectors and Kill Switches

### Companion Input Selects

| Entity | Drives | Options Source |
|---|---|---|
| `select.zenos_persona` | `input_text.zenos_persona_name` | AI user cabinet essence names |
| `select.zenos_primary_user` | `input_text.zenos_primary_user` | `person.*` with `user_id` attr |
| `select.zenos_conversation_agent` | `input_text.zenos_conversation_agent` | `conversation.*` domain |
| `select.zenos_ai_task` | `input_text.zenos_ai_task_entity` | `ai_task.*` domain |

Options resolve dynamically at render time. Each select writes back to its input_text on change — input_texts remain canonical.

### Legacy Selects

| Entity | Purpose |
|---|---|
| `select.zenos_active_persona` | Switch between registered AI personas |
| `select.zenos_reasoning_task` | Select the active reasoning task entity |
| `select.zenos_cabinet_selector` | Inspect/select cabinet slots |

### Summarizer Kill Switches

| Entity | Default | Purpose |
|---|---|---|
| `input_boolean.zen_summarizers_enabled` | `on` | Master — gates both summarizers |
| `input_boolean.zen_supersummarizer_enabled` | `on` | SuperSummary on/off |
| `input_boolean.zen_ninja_summarizer_enabled` | `on` | Ninja Summarizer on/off |

Master is checked first. If off, both summarizers exit immediately regardless of their individual switches. Turning any switch off is non-destructive — schedules, automations, and cabinet data are untouched.

**Auto-refire on re-enable:** `automation.zen_pipeline_autofire_on_enable` watches all three kill switches. When any switch is turned back `on`, it fires the appropriate force event immediately — no waiting for the next scheduled run:

| Switch turned on | Event fired |
|---|---|
| `zen_summarizers_enabled` (master) | `zen_event: {kind: supersummary_force}` |
| `zen_ninja_summarizer_enabled` | `zen_event: {kind: ninja_force}` |
| `zen_supersummarizer_enabled` | `zen_event: {kind: supersummary_force}` |

This means the summarizer is live again within seconds of being re-enabled, not at the next quarter-hour or hourly tick.

---

## Troubleshooting Quick Reference

| Symptom | Check First |
|---|---|
| Friday won't wake up | `sensor.zen_agent_health` → `roster` attribute |
| Summaries are stale | `sensor.zen_supersummary_health` → `monk_status` |
| Summaries stopped running | Check `input_boolean.zen_summarizers_enabled` and individual kill switches — all three must be `on`. If a switch was just turned back on, `zen_pipeline_autofire_on_enable` fires automatically; wait ~10s before assuming it's stuck. |
| Summarizer health shows `disabled` | Kill switch is off — intentional. Turn the switch back on; pipeline will auto-restart. |
| Scheduler not firing | `sensor.zen_summarizer_health` → `ai_task_entity` and `last_timestamp` |
| Flynn stuck at boot | `sensor.zen_flynn_health` → `current_gate` and `next_step` |
| Labels not assigning | `sensor.zen_label_health` → `missing_label_ids` and `unassigned_label_ids` |
| Cabinet missing | `sensor.zen_cabinet_health` → `missing_cabinets` |
| Resolver sensors stuck unavailable | Check `label_entities()` calls use slug IDs — display names return `[]` on strict HA versions. Fire `zen_resolver_refresh` after fixing. |
