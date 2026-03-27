# Custom Templates — ZenOS-AI 4.5.5 'Ready Player Two'

*Jinja templates powering prompt assembly, context construction, and query processing*

---

ZenOS-AI's cognitive surface runs on two Jinja template libraries. These live in `custom_templates/zenos_ai/` and are loaded by Home Assistant at startup.

---

## Templates

### `zen_os_1.jinja` — Prompt Engine & Macro Library

The core cognitive assembly engine. Everything Friday knows about herself, her household, and the current state of the home is compiled here.

**Entry point:** `render_prompt(ai_entity)` — single macro that runs the full pipeline and returns Friday's system prompt. This is what `conversation_agent_prompt_template.yaml` calls.

Key capabilities:

* Full persona resolution — jacket, core, companion, environment schema
* Highlander cabinet resolution via resolver sensors
* Identity roster (`identity_roster()`) and manifest loader (`identity_manifest_loader()`)
* Home overview: presence, active components, home mode, quiet/work hours
* Wake scene assembly from essence environment block
* Prompt integrity check (`prompt_health_check()`) — schema, signature, manifest presence
* Prompt length audit (`prompt_length_audit()`) — per-section character counts, 9 sections
* Flynn detection chain: `zen_flynn_override` → blank persona → explicit `flynn` label → resolver error → `prompt_system_flynn()`
* `prompt_system_flynn()` — hardcoded fallback prompt, zero cabinet dependencies, always works

**Used by:** `conversation_agent_prompt_template.yaml`, `zen_dojotools_supersummary`, `zen_dojotools_profile`, `zen_dojotools_identity`, `sensor.zen_prompt_health`, `sensor.zen_prompt_length`

→ **[Full reference](zen_os1_jinja.md)**

---

### `zen_query.jinja` — ZenQuery Filter Engine

The ZQ-1 query language implementation. Provides deterministic, label-graph-driven entity filtering with forward (query) and reverse (redaction) modes.

Used internally by HyperIndex and the Index tool for hypergraph traversal, adjacency expansion, and ACL-aware filtering.

→ **[Full reference](zen_query_jinja.md)**

---

## Other Files in `custom_templates/zenos_ai/`

| File | Purpose |
|---|---|
| `conversation_agent_prompt_template.yaml` | Paste this into your conversation agent's system prompt in HA. Three lines — imports `zen_os_1.jinja` and calls `render_prompt()`. |
| `command_interpreter.jinja` | Library v1 command dispatch engine. Routes `~COMMANDS~` syntax to subsystem handlers. **Retiring at GA** — individual commands are migrating to index-supported constructs. No new commands should be added here. |
| `library_index.jinja` | Library index — registered command domains and their handlers. |
| `zenos_health.jinja` | Health sensor macro library — `required_labels()`, `slots_all()`, `slot_to_label()`, cabinet state helpers, `is_warmup()`. |
| `flynn_onboarding.jinja` | Flynn onboarding macros including `active_notification()` — surfaces highest-priority Flynn persistent notification into the prompt. |

---

## How Templates Load

HA loads all files under `custom_templates/` at startup. Import them in YAML or other Jinja with:

```jinja
{%- import 'zenos_ai/zen_os_1.jinja' as zen -%}
{{ zen.render_prompt(states('input_text.zenos_persona_name')) }}
```

Templates are **not** hot-reloaded on file change — restart HA or trigger a template reload after edits.

---

→ **[Documentation Hub](../readme.md)**
→ **[zen_os_1.jinja full reference](zen_os1_jinja.md)**
