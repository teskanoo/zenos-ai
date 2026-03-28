# Kitchen Sync — Mealie ↔ Grocy Food Sync

*ZenOS-AI Plugin — v4.0.0 RC2*

---

## Overview

`kitchen_sync` is the food catalog sync orchestrator for ZenOS-AI. It performs a governed one-way sync between **Mealie** (recipe manager) and **Grocy** (pantry/inventory manager), reconciling food items across both systems.

One direction per run. Preview before applying. Bounded by limit.

---

## Script

**`script.mealie_grocy_sync_orchestrator`**

---

## Input Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `direction` | select | `mealie_to_grocy` | `mealie_to_grocy` or `grocy_to_mealie` |
| `mode` | select | `preview` | `preview` = dry run, returns what would change. `apply` = execute. |
| `limit` | number | `10` | Max items to process per run (1–100) |
| `confirmation` | boolean | `false` | Required `true` to execute an `apply` run |
| `default_location` | text | `Pantry` | Default storage location assigned to new Grocy products |
| `default_unit` | text | `each` | Default unit of measure for new products |
| `per_page` | number | `50` | Mealie API page size for food list fetches |
| `max_pages` | number | `50` | Max pages to fetch from Mealie (guards against runaway pagination) |

---

## How It Works

**Mealie → Grocy** (default direction):

1. Pages through the Mealie foods list (`zen_dojotools_mealie_helper`, `foods_list`)
2. For each food: looks up the name in Grocy (`zen_dojotools_grocy_helper`)
3. Classifies as `matched_exact`, `matched_ambiguous`, or `would_create`
4. In `apply` mode: creates missing products in Grocy with `default_location` and `default_unit`

**Preview mode** always runs safely — no writes, returns the full diff of what would be created or matched.

---

## Response

```json
{
  "tool": "Zen DojoTools Food Sync Orchestrator",
  "direction": "mealie_to_grocy",
  "mode": "preview",
  "matched_exact": ["chicken breast", "olive oil"],
  "matched_ambiguous": ["milk"],
  "would_create": ["sumac", "preserved lemon"],
  "created": [],
  "errors": []
}
```

In `apply` mode, `created` lists the products actually written to Grocy.

---

## Example: Dry run

```yaml
direction: mealie_to_grocy
mode: preview
limit: 25
```

## Example: Apply sync (first 10 items)

```yaml
direction: mealie_to_grocy
mode: apply
limit: 10
confirmation: true
default_location: Pantry
default_unit: each
```

---

## Requirements

- Mealie integration configured in HA with REST API access
- Grocy integration configured in HA with REST API access
- `zen_dojotools_mealie_helper` and `zen_dojotools_grocy_helper` scripts installed

---

## Related

- `packages/zenos_ai/plugins/mealie/mealie.yaml` — Mealie REST helper
- `packages/zenos_ai/plugins/grocy/grocy.yaml` — Grocy REST helper
- `packages/zenos_ai/plugins/grocy/readme.md` — Grocy plugin docs
