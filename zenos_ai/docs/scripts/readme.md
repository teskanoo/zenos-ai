# **📘 ZenOS-AI Script Modules**

Welcome to the **Script Modules** section of the ZenOS-AI documentation.
This directory contains formal documentation for all **DojoTools** and operational scripts that drive Friday's real-time automation, reasoning, telemetry, observability, storage access, and system reflexes.

These modules form Friday's **hands, nerves, forensics, and breadcrumbs** — the tools that speak directly to:

* Home Assistant services
* Cabinet storage
* Calendar providers
* Inspectors
* The Zen Index
* The Manifest
* The FileCabinet
* And the Monastery's internal reasoning pathways

Each module is fully documented with:

* Technical behavior
* Expected inputs & outputs
* Provider constraints
* Safety rules & failure modes
* Integration flows for LLM agents (Friday, Veronica, Kronk, High Priestess)

If Friday performs an action, reads something, correlates entities, inspects a device, updates a drawer, or leaves a breadcrumb —
**it probably came from here.**

---

# 📂 Included Script Documentation

Below is the official documentation index for all ZenOS-AI DojoTools modules.

---

## **0. Zen DojoTools Scribe — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_scribe_readme.md`
**Type:** Technical Documentation

**Summary:**
Guided authoring and lifecycle management for KF4 artifacts. Fully MCP-exposed — Friday can use it directly.

* Three artifact classes: `thought` (mutable draft), `scroll` (formal, read-only), `kfc` (published, live in Dojo)
* Full lifecycle: `new_thought` → `patch` → `formalize_scroll` → `publish_kfc`
* Scope discovery built in — `new_thought` calls Index automatically to capture a live entity snapshot before the first line is written
* `dry_run` — preview scope and guidance without committing
* `list_triggers` — return all available scheduler trigger IDs
* Non-destructive by default — `patch` preserves existing content; `replace`, `clear`, `delete`, and `publish_kfc` all require explicit confirmation
* Component group model — one corpus, multiple component views, shared clock and labelset
* Self-indexing and drawer feedback loop guards built in

A capable LLM can run the full authoring loop — scope a domain, draft the component, refine iteratively, formalize, and publish to the Dojo — without operator intervention beyond initial label design. `zen_dojotools_kungfu_writer` is retired; Scribe is the replacement.

---

## **1. Zen DojoTools AdminTools — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_admintools_readme.md`
**Type:** Technical Documentation

**Summary:**
Ring-2 administrative tools for component registration, cabinet repair, template management, and AI identity loading.

* `zen_admintools_reset_template` — Press zen_template and kfc_template into cabinets (Flynn gate-3)
* `zen_admintools_reset_labels` — **Nuclear.** Delete all `zen_` labels and assignments; Flynn auto-rebuilds via Gate 0/1
* `zen_admintools_cabinetadmin` — Inspect, restore, reset, hammer, init, or `reset_all` Ring-0 cabinets
* `zen_admintools_cabinetadmin_factory` — Factory-stamp or repair a cabinet's VolumeInfo header
* `zen_admintools_kfc_migration_press` — One-time migration: seed KF4 scheduling fields into existing drawers
* `zen_admintools_zenos_prompt_loader` — Load versioned Cortex, Directives, and Purpose (v29/latest = GA Ninja Fusion, v30 = Living Index opt-in, v27 = RC2, v31 = Signal Frame)

All tools in this module are admin-only. For KFC component registration, use `zen_dojotools_scribe` (DojoTools namespace, fully MCP-exposed).

---

## **2. Flynn — Stepgate Sentinel & Bootstrap Engine — 4.5.5 'Ready Player Two'**

**File:** `zen_flynn_readme.md`
**Type:** Technical Documentation

**Summary:**
ZenOS-AI's boot guard, initializer, OOBE driver, and prompt fallback.

* Gates 0–4: labels, cabinet init, schema seed, content bootstrap, system ready
* Early exit when system is clean and stable
* OOBE flow: names the AI, seeds household profile, flags completion
* Agent Builder (MCP-exposed): interactive config through conversation
* Auto-resolves reasoning task and AI task entity at bootstrap
* Four companion `template: select` entities for persona, primary user, conversation agent, AI task
* Prompt fallback: `render_prompt()` routes to `prompt_system_flynn()` when system not ready — zero cabinet dependencies, always answers

If the system isn't coming up clean, Flynn is why — and where to look first.

---

## **3. Zen DojoTools Profile Editor — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_profile_readme.md`
**Type:** Technical Documentation

**Summary:**
Universal read/write interface for ZenOS-AI identity profiles.

* `mode: read` — inspect current ai_user, household, user, or family profile
* `mode: write` — patch fields non-destructively (skips existing values unless `force: true`)
* `mode: help` — full field reference with examples

Expose this to your conversation agent — Friday can then help you configure your household, update persona details, and fill in profile fields conversationally. Four cabinet types: `ai_user` (persona essence), `household` (location/contact), `user` (household members), `family` (extended family).

---

## **4. Zen DojoTools Labels — 4.1.0**

**File:** `zen_dojotools_labels_readme.md`
**Type:** Technical Documentation

**Summary:**
Core primitive for label CRUD and entity tagging. Backbone of the label index that powers HyperIndex, KFC components, and cabinet resolution. MCP-exposed — write operations are gated behind `confirm: true`.

* `create` / `update` / `delete` — manage labels (confirm required)
* `update` — delete→recreate→retag with new metadata; entity assignments preserved automatically
* `read` — full index or filtered by label/entity
* `tag` / `untag` — assign or remove labels from entities
* `reset` — soft reset: wipe all entity assignments from every `zen_` label (labels survive); fires `zen_resolver_refresh`

`new_description`, `new_icon`, `new_color` params available on `create` and `update`. Label descriptions travel inline in every Inspect call as `{slug: description}` — annotate labels at creation time.

Includes label design guidance: cross-entity labels outperform single-use labels; layer broad domain labels with specific sub-labels for richer HyperIndex traversal.

---

## **5. Zen DojoTools Identity — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_identity_readme.md`
**Type:** Technical Documentation

**Summary:**
Identity resolver for ZenOS-AI household members and AI constructs. MCP-exposed.

* No target → full household roster (all valid cabinet-backed identities)
* Target (label, person entity, cabinet entity, or GUID) → single identity record
* `mode: build_identity_manifest` — write cached `zen_identity_manifest` to household cabinet
* Normalizes and validates all input forms before lookup
* Delegates resolution to `identity_resolve_source()` in `zen_os_1.jinja` — same path the prompt uses

Current scope is resolution only. Privilege enforcement, session tokens, and security masking are planned post-GA.

---

## **6. Zen DojoTools Scheduler — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_scheduler_readme.md`
**Type:** Technical Documentation

**Summary:**
Trigger orchestrator for the ZenOS-AI pipeline. Dojo-driven — components declare which triggers they care about; the scheduler routes automatically.

* 20+ trigger IDs: time patterns, home mode changes, occupancy, door/lock/window events, force events
* `summary_force` / `ninja_force` / `supersummary_force` / `gc_force` / `identity_manifest_rebuild` — manual force events
* Component subscription via `trigger_subscriptions` in Dojo drawer
* Heartbeat drawer (`zen_scheduler`) written after every run — staleness detected by `sensor.zen_summarizer_health`
* Hardware triggers: strip hardware entity IDs from core, use local `zen_dojotools_scheduler_custom.yaml`
* Protected drawers: `zen_summary`, `zen_library_manifest`, `zen_identity_manifest` — never summarized

---

## **6b. Pipeline Trigger Pattern — Act on the Monk After Kata Is Written**

**File:** `zen_pipeline_trigger_pattern.md`
**Type:** Guide

**Summary:**
How to wire an HA automation that fires a component summarizer run on a real-world trigger and acts on the monk's output.

* Fire `zen_event → summary_force` for a specific component from any automation trigger
* Wait for the kata write (fixed delay or `wait_template`), then read the kata drawer and branch on urgency/summary
* Direct MCP path: call Ninja Summarizer inline if the agent is the caller — synchronous response, no wait needed
* Full working example: flow sensor → force summarize → announce if high urgency
* Forward reference: post-pipeline event emission (kata-written signal) is a direction being explored — no ETA

---

## **7. Zen DojoTools Summarizers — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_summarizers_readme.md`
**Type:** Technical Documentation

**Summary:**
The cognition pipeline — Ninja Summarizer and SuperSummary. Both MCP-exposed.

* **Ninja Summarizer** (`zen_dojotools_ninja_summarizer`) — per-component kata writer. Reads one KFC component's Dojo drawer, runs HyperIndex + library command, sends to AI monk, writes kata drawer.
* **SuperSummary** (`zen_dojotools_supersummary`) — whole-home synthesizer. Reads all active kata drawers (gated by `meta.enabled`), sends to AI monk, writes `zen_summary` — the canonical home state that loads Friday's prompt.
* Three kill switches: master (`zen_summarizers_enabled`), ninja, supersummary — **all default off** (ship disabled; enable after verifying AI task entity points at a local model — cost risk if pointed at a paid API)
* Auto-refire on re-enable via `zen_pipeline_autofire_on_enable`

> **Warning:** Do not point the AI task entity at a paid inference API — the pipeline fires multiple times per hour. Use a local model.

---

## **8. Zen DojoTools Library — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_library_readme.md`
**Type:** Technical Documentation

**Summary:**
Friday's unified system utility runner. MCP-exposed. Dispatches to library command interpreters and utility functions.

* `library` — routes query through `command_interpreter.jinja` (retiring at GA — migrating to index-supported constructs); returns `{query, output}`
* `hash_md5` — MD5 hash of input string
* `slugify` — applies HA slugify filter to input

The `library` tool is the engine the Ninja Summarizer uses for component `command` field dispatch. KFC components register their library command in the Dojo drawer; the Ninja calls it automatically before building the monk prompt.

---

## **9. Zen Home Mode — 4.5.5 'Ready Player Two'**

**File:** `zen_home_mode_readme.md`
**Type:** Technical Documentation

**Summary:**
Eight-state home presence and time-of-day state machine. Ambient context layer for Friday, the Scheduler, and Kung Fu components.

* 8 modes: Home-Wake, Home-Morning, Home, Home-Evening, Night, Night-Late, Away, Paused
* Time-based transitions via 6 configurable `input_datetime` anchors (no code changes required)
* Presence-driven: Away auto-triggers when `zone.home` drops to 0; releases on return
* Quiet hours (midnight-wrap safe) + Work hours — both configurable, fail-open
* Scheduler trigger IDs: `home_mode_updates`, `start_home_wake`, `start_home_evening`, `home_occupancy_change`
* Vacation mode: apply `zen_vacation_mode` label to your calendar entity — resolved by label, no hardcoded ID

---

## **10. Zen DojoTools History — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_history_readme.md`
**Type:** Technical Documentation

**Summary:**
Recorder statistics query engine. Time-bucketed historical data for sensors with a valid state_class.

* `read` — query by entity, time range, period (5min/hour/day/week/month), and stat type
* Stat types: change (cumulative), mean/min/max, state, sum, last_reset
* Sensors must have statistics enabled in HA Recorder
* Never creates, updates, or deletes history (intentional, permanent)

---

## **11. Zen DojoTools Utilities — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_utilities_readme.md`
**Type:** Technical Documentation

**Summary:**
General-purpose utility collection.

* `calculator` — arithmetic, trig, conversions, GUID generation
* `dice_roller` — D&D dice notation, coin flip, random integer/float
* `announce` — TTS router to HA areas (label-resolved)
* `music_search` — Music Assistant library + internet search
* `notification_router` — multi-target notify with quiet hours and breakthrough override
* `help` — live system overview and script inventory
* `wait` — timed delay (1–120s)
* `volume_auditor` — cabinet accessibility scanner

---

## **12. Zen DojoTools Core (FileCabinet GC) — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_core_readme.md`
**Type:** Technical Documentation

**Summary:**
FileCabinet garbage collector — enforces the drawer hide/delete/recycle lifecycle across all AI Data Storage cabinets.

* `gc` — full daily scan (scheduled at midnight via Scheduler)
* `recycle` — hide a drawer + set 24h expiry
* `hide` / `unhide` — manual visibility control
* `dry_run` — preview what would be evicted

Protected drawers (`_prefix`, VolumeInfo, schema keys) are never touched. Post-eviction SuperSummary trigger built in.

---

## **13. Zen DojoTools SystemTools — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_systemtools_readme.md`
**Type:** Technical Documentation

**Summary:**
HA lifecycle management, log reading, and event emission.

* `zen_dojotools_systemtools` — Config check, safe restart (auto-validates before restarting), update install/skip
* `zen_dojotools_ha_log_viewer` — Five log modes + HA 2025.11+ journal detection
* `zen_dojotools_event_emitter` — Structured zen_event emission (see also dedicated emitter readme)
* `zen_dojotools_ha_api` — Internal API wrapper (do NOT expose)

Requires a long-lived HA token in `secrets.yaml` (ha_bearer).

---

## **14. Zen DojoTools Office — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_office_readme.md`
**Type:** Technical Documentation

**Summary:**
Provides unified, deterministic access to all Home Assistant calendar entities:

* Multi-calendar read aggregation
* Timestamp normalization
* Deep event inspection
* Safe event creation
* Provider-verified update/delete
* Label-based calendar targeting
* Strict ambiguity prevention
* Fully structured JSON response envelopes

This is Friday's and Veronica's primary tool for anything date-, event-, or schedule-related.

---

## **15. Zen DojoTools Event Emitter — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_event_emitter_readme.md`
**Type:** Technical Documentation

**Summary:**
A universal, contract-safe telemetry tool for emitting structured ZenOS-AI events onto the Home Assistant EventBus.

Supports the canonical ZenOS event shape:

```
{
  "timestamp": "ISO-8601",
  "component": "string",
  "severity": "debug|info|warn|error",
  "kind": "classifier",
  "summary": "single-sentence explanation",
  "metadata": {},
  "kata": {},
  "monk": {}
}
```

This is how Friday, Veronica, Kronk, and the High Priestess leave traceable breadcrumbs, emit summaries, and coordinate system-awareness.

---

## **16. Zen DojoTools FileCabinet — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_filecabinet_readme.md`
**Type:** Technical Documentation

**Summary:**
The authoritative, health-aware read/write controller for all Cabinet Volumes.

Supports:

* Drawer create/update/delete
* Cross-volume move/copy
* Label indexing & label-based lookup
* Directory listings
* Volume-wide reads
* Manifest passthrough
* JSON-safe parsing
* Concurrency protection
* Health validation (schema, flags, GUID, storage thresholds)

If any drawer changes anywhere in ZenOS-AI, it happened through FileCabinet.

---

## **17. Zen DojoTools Manifest — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_manifest_readme.md`
**Type:** Technical Documentation

**Summary:**
The runtime-only Cabinet manifest scanner.
Builds a complete, zero-persistence health and metadata model for every Cabinet Volume.

Provides:

* GUID checks
* Schema compatibility
* Drawer discovery
* Label index extraction
* Capacity analysis
* Access flags
* ACL expansion
* Health classification

If Friday trusts a Cabinet, it's because the Manifest told her it's safe.

---

## **18. Zen DojoTools Index — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_index_readme.md`
**Type:** Technical Documentation

**Summary:**
The entity-and-label correlation engine of ZenOS-AI.

Supports:

* Entity & label set logic (AND/OR/NOT/XOR/`*`)
* Label → entity resolution
* Index Command DSL mode
* Integration with Zen Inspect
* Adjacency label discovery
* Drawer-level correlation
* JSON-safe structured outputs

The Zen Index is Friday's "graph engine," letting her understand relationships between entities, labels, and volumes.

---

## **19. Zen DojoTools Inspect — 4.5.5 'Ready Player Two'**

**File:** `zen_dojotools_inspect_readme.md`
**Type:** Technical Documentation

**Summary:**
A safe, deterministic deep-inspection tool for entities — Friday's x-ray machine.

Provides:

* Entity snapshots
* Labels as `{slug: description}` dict — inline semantic context, no secondary lookup needed
* Fully sanitized attributes
* Cabinet sensor header detection
* Statistics eligibility detection
* Optional extended device forensics
* JSON-stable output for LLM use

Inspect shows Friday "what something is" without ever exposing dangerous internals.

---

# ✔️ Summary

This directory represents the **core operational toolkit** of ZenOS-AI.
Together, these modules form the backbone of:

* storage
* introspection
* indexing
* scheduling
* telemetry
* safe state mutation
* reasoning pipelines
* agent support

They are the **foundation** upon which Friday, Veronica, Kronk, and the High Priestess express intelligence inside Home Assistant.
