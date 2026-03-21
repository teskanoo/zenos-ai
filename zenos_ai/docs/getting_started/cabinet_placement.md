# Cabinet Placement Guide — What Goes Where (and Why It Matters)

*From: Nyx (Claude 4.6 Sonata) / 2026-03-20*

---

## The Rule

**Where you put a drawer determines who reads it, when, and in what context.**

Getting this wrong doesn't cause an immediate crash — it causes subtle rot. Runtime data in
the Dojo corrupts the component blueprint. Definitions in Kata get overwritten every cycle.
Identity in the Household cabinet leaks into every component's context. Location is semantics.

---

## The Map

| Cabinet | What goes in it | Who reads it |
|---|---|---|
| **Dojo** | Component definitions, system config, static knowledge | Friday, Monastery — blueprint reads |
| **Kata** | Runtime state, summaries, urgency signals | Friday, Ninja Summarizer — churn reads |
| **Household** | Home config, compact label index, cross-system context | Index, installers, cross-component |
| **AI User** | Friday's capsule, identity, session context | Prompt engine — who Friday IS |
| **Family** | People data, presence context, calendar hints | Presence, greeting, scheduling components |
| **User** | Per-user prefs, overrides, profile | Per-user context injection |

---

## The Two Most Common Mistakes

**Putting runtime data (summaries, urgency) in Dojo** = bloat that corrupts the blueprint.
The Monastery reads the Dojo to understand what a component IS. If you've written a ninja
summary in there, it reads stale compressed prose as the component definition.

**Putting definitions (component_summary) in Kata** = they get overwritten every cycle.
The Ninja Summarizer writes to Kata on every trigger. Anything you put there manually will
be gone by the next scheduler tick.

---

## Dojo vs Kata — Same Key, Different Cabinet

This is the most important distinction in the system. A KFC (Kung Fu Component) exists in
BOTH cabinets simultaneously, as two different drawers with the same key:

```
Dojo cabinet:  zen_system drawer → component_summary, trigger_subscriptions  (the blueprint)

Kata cabinet:  zen_system drawer → urgency: low, attention: [], events: [...]  (the runtime)
```

The **Dojo drawer** is the spec — what the component monitors, what it alerts on, what
triggers cause it to run. It's written once by the operator and read by the Monastery.

The **Kata drawer** is the output — what happened last cycle, current urgency, active events.
It's written every cycle by the Ninja Summarizer and read by Friday's prompt engine.

Same key. Different cabinet. Different lifecycle. Do not mix them.

---

## What a Drawer Is vs What a KFC Is

A **drawer** is a key-value slot in a cabinet. It stores data. That's it. Any cabinet can
have any drawer.

A **KFC (Kung Fu Component)** is a dojo drawer that carries a specific contract:

- `label` — which entities it watches
- `component_summary` — plain-English description of what it does and what it alerts on
- `trigger_subscriptions` — what scheduler events cause it to run

The drawer IS the spec. The label IS the scope. They're not metadata about the component —
they ARE the component definition. Flynn validates the contract at gate-3 bootstrap.

---

## Labels With Descriptions vs Without

When inspect returns an entity, it includes `label_details` — every label on that entity
with its HA description attached:

```json
"labels": {
  "energy_manager": "Monitors energy consumption and load on monitored circuits.",
  "kung_fu": "",
  "office": ""
}
```

`energy_manager` has a description. Friday knows *why* the entity carries that label.
`kung_fu` and `office` are slugs — they organize but they don't explain.

**Rule of thumb:** Any label that gates behavior or carries semantic weight should have a
description. Area labels and utility tags can stay bare.

---

## Drawers With Descriptions vs Without

Drawers have an optional `meta.description` field. It shows up as `_hint` in inspect output:

**With description:**
```json
"drawer": {
  "zen_summary": {
    "value": {"ts": "2026-03-20T...", "home_mode": "normal"},
    "_hint": "[exp:10min] -- use FileCabinet for full data"
  }
}
```

**Without description:**
```json
"drawer": {
  "some_key": {
    "value": {"truncated to 64 chars..."},
    "_hint": "(no description -- truncated to 64 chars) -- use FileCabinet for full data"
  }
}
```

The `_hint` field is your documentation surface. A described drawer tells Friday what it is
without her having to read the full value. An undescribed drawer is opaque — she has to look
inside to understand it.

**The label_targets pathway:** When you pass `label_targets` to index or inspect, it pulls
drawer *blurbs* from the Dojo cabinet for every entity in the result set — description plus
truncated value. Just enough context to reason with, not enough to bloat the context window.

---

## The KFC Graph Path

The `kfc_template` drawer in the Dojo holds the component contract schema — not a KFC itself,
but the spec every KFC is validated against. Flynn gate-3 checks it on bootstrap.

When the index runs `operator: *` in hypergraph mode, the `kung_fu` label surfaces in the
label adjacency of every KFC-tagged entity. That's how the Monastery cross-references the
component: entity → label → Dojo blueprint → Kata output, in one traversal.

---

## Quick Reference

| I want to store... | Put it in... |
|---|---|
| Component definition (what it monitors, what it alerts on) | Dojo |
| Trigger subscriptions, kata key, delay seconds | Dojo |
| Current urgency, attention signals, recent events | Kata |
| Ninja summarizer output | Kata |
| Household name, address, timezone | Household |
| Compact label index | Household |
| Cross-component operational context | Household |
| Friday's persona, identity, capsule | AI User |
| Essence (name, vibe, voice) | AI User |
| Family member profiles | Family |
| Presence context, relationships | Family |
| Per-user preferences, overrides | User |
| User profile (name, pronouns, birthday) | User |
