# Flynn — Stepgate Sentinel & Bootstrap Engine — 4.5.0 'Meridian'

*ZenOS-AI's boot guard, initializer, and onboarding driver*

---

## Overview

Flynn is the first thing that runs and the last line of defense before ZenOS-AI hands off to Friday.

On every HA startup — and whenever a health sensor changes state — Flynn walks through a **sequential stepgate sequence**. Each gate checks one precondition. If it passes, Flynn advances. If it fails, Flynn repairs what it can, notifies you, and stops. The next health sensor update re-engages the sequence.

Flynn also drives **OOBE** — the first-boot configuration flow that names your AI, seeds your household profile, and stamps the system ready.

Nothing inside ZenOS-AI runs until Flynn says it can.

---

## What Flynn Monitors

Flynn re-runs any time these sensors change state:

| Sensor | What It Watches |
|---|---|
| `sensor.zen_label_health` | Required labels exist and are assigned to entities |
| `sensor.zen_cabinet_health` | Required cabinet entities are present and initialized |
| `sensor.zen_monastery_health` | Schema templates + cabinet content are seeded |

States that re-engage Flynn: `critical`, `error`, `warn`, `ok`

---

## Early Exit

Flynn skips all gates and exits immediately if **all** of the following are true:

- `binary_sensor.flynn_system_ready` is `on`
- All health sensors are `ok` (monastery may be `warn` — degraded but functional)
- OOBE is not pending
- Both `kfc_template` (Dojo) and `zen_template` (Kata) are already seeded

If the system is already clean and stable, Flynn steps aside.

---

## The Stepgate Sequence

### Gate 0 — Label Creation

**Trigger:** `sensor.zen_label_health == critical`

Labels don't exist yet. Flynn calls DojoTools Labels to create the full required label set, then fires a persistent notification telling you to restart HA. Label creation requires a restart to take effect.

**What you'll see:** Notification — *"Labels Created — Restart Required"*

**What to do:** Restart Home Assistant.

---

### Gate 1 — Label Assignment

**Trigger:** `sensor.zen_label_health == warn`

Labels exist but aren't assigned to the right cabinet entities. Flynn calls `script.flynn_assign_labels` to wire each cabinet sensor to its required labels. Three passes:

1. All slot entities → `zen_cabinet` parent label
2. Each slot → its specific slot label (e.g., `zen_dojo_cabinet`, `zen_kata_cabinet`)
3. Broad labels → default household, family, user, AI user cabinet entities

Once complete, Flynn stops and waits for `zen_label_health` to update.

**Event fired:** `flynn_stepgate_event` (gate: 1, action: labels_assigned)

---

### Gate 2 — Cabinet Initialization

**Trigger:** `sensor.zen_cabinet_health != ok`

One or more required cabinet entities are uninitialized. Flynn reads the `missing_cabinets` list from the health sensor, maps each slot to its cabinet type, and calls `zen_admintools_cabinetadmin` with `mode: initialize` for each one.

Cabinet types initialized on demand:

```
system, dojo, kata, default_household, default_family,
default_user, default_ai_user, history, index
```

Flynn stops and waits for `zen_cabinet_health` to update.

**Event fired:** `flynn_stepgate_event` (gate: 2, action: cabinets_initialized)

---

### Gate 3 — Schema Seeding + Content Bootstrap

**Trigger:** `sensor.zen_monastery_health` is not fully settled, OR schema templates are missing

Gate 3 has three distinct sub-steps:

#### 3a — Resolver Stability Check

Before doing anything, Flynn checks that the three cabinet resolver sensors have settled:

- `sensor.zen_dojo_cabinet_resolved`
- `sensor.zen_kata_cabinet_resolved`
- `sensor.zen_system_cabinet_resolved`

If any resolver is `unknown`, `unavailable`, or empty — Flynn waits. Label assignment from Gate 1 needs time to propagate.

#### 3b — Schema Seed (always runs)

Regardless of monastery health state, Flynn checks whether:
- Dojo cabinet has a `kfc_template` drawer
- Kata cabinet has a `zen_template` drawer

If either is missing, Flynn calls `zen_admintools_reset_template` to press both templates. This is infrastructure — it runs on every boot if needed and is fully idempotent.

#### 3c — Content Bootstrap (critical only)

If `zen_monastery_health == critical`, Flynn calls `script.flynn_bootstrap_content` with your configured helper values:

- `input_text.zenos_conversation_agent`
- `input_text.zenos_household_name`
- `input_text.zenos_primary_user`
- `input_text.zenos_persona_name`

> `warn` state = degraded but functional. Schema seed runs (3b), content bootstrap does not. System continues.

**Bootstrap does the following:**

1. **Validates conversation agent** — if missing or unavailable, stops with notification. Nothing else proceeds without an agent.
2. **Auto-resolves reasoning task** — reads `input_text.zenos_reasoning_task`. If unset and exactly one conversation entity exists, auto-writes it. Multiple found with none configured → stops with notification.
3. **Auto-resolves AI task entity** — reads `input_text.zenos_ai_task_entity`. Same auto-resolve logic. Zero found = non-fatal (summarizer degrades gracefully).
4. **Seeds household name** — non-destructive write to `_household_name` drawer.
5. **Seeds AI persona essence** — writes default legacy-schema essence with `identity.name = 'your AI'` as placeholder (non-destructive — skips if drawer already exists). Three-layer cabinets already have a real name stamped at mint; this step is a no-op for them.
6. **Seeds schema templates** — calls `reset_template` if either template drawer is missing.
7. **Loads prompt substrate** — calls `zen_admintools_zenos_prompt_loader` (Cortex, Directives, Purpose).
8. **Flags completion** — calls `flynn_unified_engine` with `action_type: flag_complete`.

**Event fired:** `flynn_stepgate_event` (gate: 3, action: bootstrap_complete)

---

### Gate 3.5 — OOBE

Runs after Gate 3, with a 10-second delay to let Gate 3 writes settle.

**OOBE is pending when:**
- The AI user cabinet's persona name resolves to `''` or `'your AI'` — checked via `essence_probe()` which handles both legacy (`identity.name`) and three-layer (`jacket.name`) schemas
- AND no `_oobe_complete` flag exists in the cabinet

**Pending name values:** `''`, `'your AI'`, `'unknown'` (all treated as placeholder — system will prompt for a real name)

Three cases:

| Situation | What Flynn Does |
|---|---|
| Input helper has a name + cabinet has placeholder | Writes persona name to essence. Dismisses notification. |
| Input helper is empty + cabinet has placeholder | Creates "Welcome — Let's name your AI" notification. Directs to conversation or Agent Builder. |
| Cabinet already has a real name OR `_oobe_complete` flag | Dismisses any pending OOBE notification. Nothing to do. |

The fastest path through OOBE: set `input_text.zenos_persona_name` in Settings → Helpers, then restart or let Flynn re-run. Flynn will write the name and clear the gate.

The conversational path: start chatting with your AI agent — Flynn's Agent Builder (mode: oobe) walks through the Q&A setup interactively.

---

### Gate 4 — System Ready

**Trigger:** All health sensors green + OOBE complete

Fires **once on transition** (not on subsequent green re-runs). Creates a persistent notification:

> *"ZenOS-AI: System Ready — Everything's looking good. All gates green. Handing off to the real talent."*

---

## Agent Builder (`script.flynn_build_agent`)

An interactive configuration tool for setting up the system through conversation. **MCP-exposed.**

| Mode | What It Does |
|---|---|
| `help` | Returns setup instructions and current state. Auto-fires on first call. |
| `status` | Read-only: returns current persona name, household name, primary user. |
| `build` | Non-destructive writes: household name, persona name, primary user, family members. |
| `oobe` | Delegates to the full OOBE protocol. |

All `build` writes are non-destructive — existing values are never overwritten.

---

## Companion Selects

Four `template: select` entities provide UI-level dropdowns for the critical input helpers. Each select reads the current value of its corresponding `input_text` and writes back to it on change. The `input_text` entities remain the canonical source of truth — selects are overlay controls only.

| Select Entity | Drives | Options Source |
|---|---|---|
| `select.zenos_persona` | `input_text.zenos_persona_name` | AI user cabinet essence names |
| `select.zenos_primary_user` | `input_text.zenos_primary_user` | `person.*` with `user_id` attr (login-bound only) |
| `select.zenos_conversation_agent` | `input_text.zenos_conversation_agent` | All `conversation.*` domain entities |
| `select.zenos_ai_task` | `input_text.zenos_ai_task_entity` | All `ai_task.*` domain entities |

Options resolve dynamically at render time. The persona select only shows personas with a valid name in their essence — `unknown`, `your AI`, and empty strings are excluded.

**Fastest setup path:** Use these selects from Settings → Helpers instead of typing entity IDs manually.

---

## Health Sensor → Gate Map

| Sensor State | Gate Triggered | Action |
|---|---|---|
| `zen_label_health: critical` | Gate 0 | Create labels, notify, stop |
| `zen_label_health: warn` | Gate 1 | Assign labels to entities |
| `zen_cabinet_health: not ok` | Gate 2 | Initialize missing cabinets |
| `zen_monastery_health: critical` | Gate 3 | Full content bootstrap |
| `zen_monastery_health: warn` | Gate 3 (partial) | Schema seed only |
| All green + OOBE complete | Gate 4 | System ready notification |

---

## Troubleshooting

### Stuck at Gate 0

`zen_label_health: critical` — labels haven't been created yet, or HA hasn't restarted after creation.

**Fix:** Check for the "Labels Created" notification. Restart HA.

---

### Stuck at Gate 1

`zen_label_health: warn` — labels exist but aren't assigned. Flynn should self-resolve on the next trigger. If it doesn't:

**Check:** Developer Tools → States → `sensor.zen_label_health` attributes. Look at `required_labels` vs what's assigned.

**Manual fix:** Run `script.flynn_assign_labels` directly from Developer Tools → Services.

---

### Stuck at Gate 2

`zen_cabinet_health: not ok` — one or more cabinets uninitialized.

**Check:** `sensor.zen_cabinet_health` attributes → `missing_cabinets` list.

**Manual fix:** Run `script.zen_admintools_cabinetadmin` with `mode: initialize` and the specific cabinet entity.

---

### Stuck at Gate 3 — "Conversation Agent Required"

Flynn stopped in `bootstrap_content` because `input_text.zenos_conversation_agent` is empty or the entity is unavailable.

**Fix:** Settings → Helpers → use the **ZenOS: Conversation Agent** select — it shows all available `conversation.*` entities as a dropdown. Or set `zenos_conversation_agent` manually. Verify the entity is available in Developer Tools.

---

### Stuck at Gate 3 — "Reasoning Task / AI Task Entity Required"

Multiple conversation or ai_task entities found and neither helper is configured.

**Fix:** Settings → Helpers → set `zenos_reasoning_task` or `zenos_ai_task_entity` to the correct entity ID explicitly.

---

### OOBE Notification Won't Dismiss

`binary_sensor.flynn_system_ready` is likely still off. Check upstream health sensors — if anything is warn/error, Flynn won't reach Gate 3.5 to dismiss it.

Alternatively: set `input_text.zenos_persona_name` to your desired AI name and trigger a restart or manually run the Stepgate Sentinel.

---

### `dojo_loaded: warn` in logs

Resolver sensors haven't settled after label assignment. Flynn gates on resolver state in Gate 3a — this is expected briefly after a restart. Give it 30–60 seconds. If it persists, check that `zen_dojotools_labels` created the label and that `zenos_dojo_cabinet` entity exists.

---

## Flynn as Prompt Fallback

Flynn is not just a boot guard — he is the fallback persona when the system cannot build a valid prompt. `render_prompt()` in `zen_os_1.jinja` checks for Flynn conditions before assembling Friday's prompt.

### Mode Detection Chain

`render_prompt(ai_entity)` evaluates these conditions in order:

| Priority | Condition | `_flynn_reason` |
|----------|-----------|-----------------|
| 1 | `input_boolean.zen_flynn_override` is `on` | `override` |
| 2 | `ai_entity` is blank or whitespace | `blank_persona` |
| 3 | persona label explicitly set to `flynn` | `explicit` |
| 4 | `identity_resolve_source()` returns error | error key from resolver |

If any condition matches, `prompt_system_flynn()` is returned — a hardcoded system prompt with zero cabinet dependencies.

### input_boolean.zen_flynn_override

Defined in `flynn.yaml`. Auto-creates on HA reload. Toggle this on to force Flynn mode for any conversation regardless of the selected persona — useful for maintenance, debugging, and testing without changing the persona selector.

Default state: `off`. Safe to leave off indefinitely.

### prompt_system_flynn()

Hardcoded system prompt — no Jinja cabinet reads. Works on a bare install, works when cabinets are corrupt, works when the identity resolver fails. Flynn will always answer.

### active_notification()

`active_notification()` macro in `flynn_onboarding.jinja` scans `persistent_notification.flynn_*` for active notifications and returns the highest-priority one with context label. Flynn's prompt reads this and acknowledges the active notification in its opening — giving coherent UX during first-boot and error states.

All four Flynn persistent notification messages are written in Flynn's voice and include an invitation to open the conversation agent.

---

## Events Reference

| Event | When | Payload |
|---|---|---|
| `flynn_stepgate_event` | After each gate action | `gate`, `action`, `timestamp` + gate-specific fields |
| `flynn_onboarding_updated` | Unified engine state changes | `type`, `drawer`, `key`, `timestamp` |

---

## Dependencies

| Dependency | Role |
|---|---|
| `sensor.zen_label_health` | Gate 0/1 trigger + label assignment data |
| `sensor.zen_cabinet_health` | Gate 2 trigger + missing cabinet list |
| `sensor.zen_monastery_health` | Gate 3 trigger |
| `script.zen_dojotools_labels` | Gate 0 label creation |
| `script.zen_admintools_cabinetadmin` | Gate 2 cabinet initialization |
| `script.zen_admintools_reset_template` | Gate 3 schema seed |
| `script.zen_admintools_zenos_prompt_loader` | Gate 3 prompt substrate |
| `script.zen_dojotools_filecabinet` | All cabinet reads and writes |
| `binary_sensor.flynn_system_ready` | Early exit gate |
| `input_boolean.zen_flynn_override` | Forces Flynn mode regardless of persona |
| `zen_os_1.jinja` → `render_prompt()` | Calls `prompt_system_flynn()` when Flynn conditions met |
| `flynn_onboarding.jinja` → `active_notification()` | Surfaces active Flynn notifications in prompt |
| `input_text.zenos_conversation_agent` | Bootstrap validation |
| `input_text.zenos_persona_name` | OOBE persona name |
| `input_text.zenos_primary_user` | Bootstrap primary user seed |
| `input_text.zenos_reasoning_task` | Auto-resolved at bootstrap |
| `input_text.zenos_ai_task_entity` | Auto-resolved at bootstrap |
| `select.zenos_persona` | Companion UI for persona name |
| `select.zenos_primary_user` | Companion UI for primary user |
| `select.zenos_conversation_agent` | Companion UI for conversation agent |
| `select.zenos_ai_task` | Companion UI for AI task entity |
