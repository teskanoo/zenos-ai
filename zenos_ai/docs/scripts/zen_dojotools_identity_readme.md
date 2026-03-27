# Zen DojoTools Identity — 4.5.5 'Ready Player Two'

*Identity resolution and household/family group management for ZenOS-AI*

---

## Overview

`zen_dojotools_identity` is the identity resolver and group management tool for ZenOS-AI.

It handles two distinct concerns:

**Resolution** (`resolve`, `prompt`) — looks up registered users, AI constructs, or cabinets and returns identity records. Stateless, no side effects. Used by the AI prompt pipeline and MCP callers.

**Group management** — all household and family membership operations: adding and removing members, wiring families into households, linking delegation partners, and querying the membership graph. These modes write to cabinet drawers and fire events.

The tool is **MCP-exposed**. Resolution modes are safe to call at any time. Write modes are idempotent where noted.

---

## The Identity Model

```
System
  └─ Household (occupancy — "lives here")
       └─ Family (belonging — "part of this unit")
            └─ Sub-family (extended family — not on-premises)
```

**Household membership = residence.** In the household's `members` list means you occupy that space.

**Family membership = belonging.** Being in a family means you're part of that unit.

**Depth rule:** Families nest arbitrarily deep in the data model. Security resolution only chases 2 levels — `is_member(A, B)` checks: is A directly in B? Or is A in C which is directly in B? Anything deeper is not resolved for security purposes.

**Principal slots:** Each household and family has two privileged occupant slots:
- `acls.owner` — Head of Household (HoH) for a household; primary user for a family
- `acls.partner[role=prime]` — Prime AI partner

These slots fill on first add and block re-entry. Use `set_principal` to transfer them.

**Partner = delegation authority.** `acls.partner[]` records who is authorized to delegate on an entity's behalf. This is a governance relationship, not a social one. No token is issued without an explicit allow — the link records who has delegation authority, it does not grant it automatically. Works for any entity pair (user↔user, user↔AI, AI↔AI).

---

## Modes

| Mode | Description | Writes? |
|---|---|---|
| `resolve` | Returns identity record for target, or full roster if no target | No |
| `prompt` | Returns rendered prompt capsule for target construct | No |
| `build_identity_manifest` | Builds `{roster, tree}` and writes to household cabinet | Yes |
| `household_add_family` | Wires a family cabinet into the household `members.families` list | Yes |
| `household_remove_family` | Removes a family cabinet from the household members list | Yes |
| `household_add_member` | Adds a user or AI to household; fills HoH/prime slot on first add | Yes |
| `household_remove_member` | Removes a user or AI from household | Yes |
| `set_principal` | Sets or replaces the HoH or prime AI slot for a household or family | Yes |
| `family_add_member` | Adds a user, AI, or sub-family to a family; sets default family on first join | Yes |
| `family_remove_member` | Removes a user, AI, or sub-family from a family | Yes |
| `link_partners` | Writes `acls.partner[]` on both entities (bidirectional delegation link) | Yes |
| `unlink_partners` | Removes each entity from the other's `acls.partner[]` | Yes |
| `set_default_family` | Patches `default_family_guid` on a member's VolumeInfo | Yes |
| `membership` | Tree view for containers; reverse lookup for members | No |
| `is_member` | Depth-2 boolean check — is entity A a member of container B? | No |

`resolve` is the default mode — existing callers are unaffected.

---

## Input Fields

| Field | Type | Description |
|---|---|---|
| `mode` | select | Operation mode (see table above) |
| `user_label` | text | Label referencing a registered ZenOS-AI user (resolve/prompt only) |
| `user_cabinet` | entity (sensor) | Cabinet sensor entity_id (resolve/prompt only, planned) |
| `user_entity_id` | entity (person) | HA person entity_id (resolve/prompt only, planned) |
| `user_guid` | text | ZenOS-AI user GUID (resolve/prompt only, planned) |
| `member_entity` | entity (sensor) | Cabinet to add/remove as a member, or entity A for link_partners |
| `member_type` | select | `user`, `ai_user`, or `family` — type of member being operated on |
| `family_entity` | entity (sensor) | Family cabinet target (family ops, set_principal override) |
| `ai_entity` | entity (sensor) | Entity B for link_partners / unlink_partners |
| `household_entity` | entity (sensor) | Explicit household cabinet — overrides default resolver |

**Resolution priority** (resolve/prompt modes):
```
user_label → user_cabinet → user_entity_id → user_guid
```

**Multi-household:** All household ops default to `sensor.zen_default_household_cabinet_resolved`. Pass `household_entity` to target a specific household cabinet.

---

## Mode Reference

### `resolve` / `prompt`

Stateless identity lookup. `resolve` returns the identity record; `prompt` returns the rendered prompt capsule.

```yaml
# Full roster — omit all target fields
zen_dojotools_identity:
  mode: resolve

# Specific target
zen_dojotools_identity:
  mode: resolve
  user_label: nathan
```

---

### `build_identity_manifest`

Builds the identity manifest and writes `{roster, tree}` to the `zen_identity_manifest` drawer in the household cabinet. The tree is the depth-2 household membership structure including principal slots.

```yaml
zen_dojotools_identity:
  mode: build_identity_manifest
```

Can also be triggered via event:
```
zen_event kind: identity_manifest_rebuild
```

**Response:** `{result: ok, manifest_written: true, timestamp}`

---

### `household_add_family`

Wires a family into the household's `members.families` list at depth 1. Family members become household residents by graph traversal.

```yaml
zen_dojotools_identity:
  mode: household_add_family
  family_entity: sensor.<family_cabinet>
```

---

### `household_remove_family`

Removes a family from the household members list.

```yaml
zen_dojotools_identity:
  mode: household_remove_family
  family_entity: sensor.<family_cabinet>
```

> ⚠️ **Family teardown order matters.** `deprovision` does NOT remove a family from the household. Skipping `household_remove_family` before deprovisioning leaves a stale entry in `members.families` — re-provisioning and re-adding the family then creates a duplicate.
>
> Correct teardown sequence:
> 1. `family_remove_member` for each member
> 2. `household_remove_family`
> 3. Deprovision member cabinets
> 4. Deprovision family cabinet

---

### `household_add_member`

Adds a user or AI to the household.

```yaml
zen_dojotools_identity:
  mode: household_add_member
  member_entity: sensor.<cabinet>
  member_type: user   # or ai_user
```

**First occupant behavior:**
- First `user` → also fills `acls.owner` (Head of Household slot)
- First `ai_user` → also fills `acls.partner[role=prime]`

**Occupied slot:** If the HoH or prime slot is already filled, add is blocked with `slot_occupied` error. Use `set_principal` to transfer the slot.

**Response includes** `slot_filled: hoh | prime | ''`.

**Event fired:** `zen_event kind: household_member_joined`

---

### `household_remove_member`

Removes a user or AI from the household.

```yaml
zen_dojotools_identity:
  mode: household_remove_member
  member_entity: sensor.<cabinet>
  member_type: user   # or ai_user
```

Does **not** clear `acls.owner` or `acls.partner` slots — those require `set_principal`.

**Response includes** `principal_warning: hoh_slot_stale | prime_slot_stale | ''` — warns if the removed member held a principal slot.

**Event fired:** `zen_event kind: household_member_left` with `principal_warning` field.

---

### `set_principal`

Sets or replaces the Head of Household or prime AI slot for a container cabinet.

```yaml
zen_dojotools_identity:
  mode: set_principal
  member_entity: sensor.<cabinet>
  member_type: user      # user → fills/replaces acls.owner (HoH)
                         # ai_user → fills/replaces acls.partner prime slot
  # family_entity: sensor.<family_cabinet>  # optional — targets family instead of default household
```

**Default target:** `zen_default_household_cabinet`. Pass `family_entity` to target a specific family cabinet instead.

**Replacement:** Previous occupant of the slot is replaced. For prime AI slot, non-prime partner entries are preserved.

**Event fired:** `zen_event kind: principal_changed`

---

### `family_add_member`

Adds a user, AI, or sub-family to a family.

```yaml
zen_dojotools_identity:
  mode: family_add_member
  family_entity: sensor.<family_cabinet>
  member_entity: sensor.<cabinet>
  member_type: user   # user | ai_user | family
```

**Default family:** If this is the member's first family (no `default_family_guid` set), this family is automatically set as their default.

**Sub-families:** `member_type: family` wires in a sub-family at depth 2. Sub-family members are extended family — not household residents.

**Bidirectional sync:** Also writes `acls.family[]` on the member's VolumeInfo.

**Event fired:** `zen_event kind: family_member_joined` with `prompt_profile_update: true`

---

### `family_remove_member`

Removes a user, AI, or sub-family from a family.

```yaml
zen_dojotools_identity:
  mode: family_remove_member
  family_entity: sensor.<family_cabinet>
  member_entity: sensor.<cabinet>
  member_type: user   # user | ai_user | family
```

**Default family cleanup:** If the removed family was the member's `default_family_guid`, that field is cleared. The member must explicitly set a new default via `set_default_family`.

**Bidirectional sync:** Scrubs `acls.family[]` entry by GUID on the member's VolumeInfo.

**Response includes** `principal_warning` and `was_default_family` fields.

**Event fired:** `zen_event kind: family_member_left` with `was_default_family` and `principal_warning` fields.

---

### `link_partners`

Writes `acls.partner[]` on **both** entities. Works for any entity pair (user↔user, user↔AI, AI↔AI).

```yaml
zen_dojotools_identity:
  mode: link_partners
  member_entity: sensor.<cabinet_a>   # any type: user, ai_user
  ai_entity: sensor.<cabinet_b>       # any type: user, ai_user
```

Each entry carries `{guid, entity_id, cab_type, role: partner, sid}`.

Idempotent — re-running does not create duplicate entries.

**Event fired:** `zen_event kind: partner_linked`

---

### `unlink_partners`

Removes each entity from the other's `acls.partner[]`. Symmetric.

```yaml
zen_dojotools_identity:
  mode: unlink_partners
  member_entity: sensor.<cabinet_a>
  ai_entity: sensor.<cabinet_b>
```

**Event fired:** `zen_event kind: partner_unlinked`

---

### `set_default_family`

Patches `default_family_guid` in the member's VolumeInfo. The first family joined sets this automatically; use this to change it explicitly.

```yaml
zen_dojotools_identity:
  mode: set_default_family
  member_entity: sensor.<user_or_ai_cabinet>
  family_entity: sensor.<family_cabinet>
```

---

### `membership`

Read-only graph query. Two behaviors depending on entity type:

**Container entity** (household, family) → returns depth-2 tree:

```yaml
zen_dojotools_identity:
  mode: membership
  member_entity: sensor.<household_or_family_cabinet>
```

Returns: `{principals: {hoh, prime}, members: {users, ai_users}, families: [{..., subfamilies: [...]}]}`

**Member entity** (user, ai_user) → returns reverse lookup: which households and families is this entity in?

```yaml
zen_dojotools_identity:
  mode: membership
  member_entity: sensor.<user_or_ai_cabinet>
```

Returns: `{household_memberships: [...], family_memberships: [...]}`

---

### `is_member`

Depth-2 boolean membership check.

```yaml
zen_dojotools_identity:
  mode: is_member
  member_entity: sensor.<entity_to_check>
  family_entity: sensor.<container_to_check_against>   # household or family
```

Returns: `{is_member: true|false, depth: 1|2|null}`

- `depth: 1` — direct member of the container
- `depth: 2` — member of a sub-family that is directly in the container
- `depth: null` — not a member at either depth

---

## Onboarding Sequence (Standard)

When provisioning a fresh system, the recommended identity wiring sequence:

1. Mint household — provision `cab_type: household`
2. Add family to household — `household_add_family`
3. Mint user — provision `cab_type: user`
4. Add user to family — `family_add_member` (first family auto-sets `default_family_guid`; user fills HoH slot in household)
5. Mint AI user — provision `cab_type: ai_user`
6. Add AI to household — `household_add_member` (first AI fills the prime slot)
7. Link user and AI — `link_partners` (bidirectional delegation link)
8. Optionally invite AI into family — explicit `family_add_member`; fires join event

Flynn bootstrap (`flynn_bootstrap_content`) runs steps 4–7 automatically on first boot.

---

## Events Reference

| Event | Mode | Key Fields |
|---|---|---|
| `household_member_joined` | `household_add_member` | `member_entity`, `member_type`, `slot_filled` |
| `household_member_left` | `household_remove_member` | `member_entity`, `member_type`, `principal_warning` |
| `principal_changed` | `set_principal` | `member_entity`, `member_type`, `target_cabinet`, `previous_principal` |
| `family_member_joined` | `family_add_member` | `member_entity`, `member_type`, `family_entity`, `prompt_profile_update: true` |
| `family_member_left` | `family_remove_member` | `member_entity`, `family_entity`, `was_default_family`, `principal_warning` |
| `partner_linked` | `link_partners` | `entity_a`, `entity_b` |
| `partner_unlinked` | `unlink_partners` | `entity_a`, `entity_b` |

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `zen_os_1.jinja` → `identity_resolve_source()` | Full identity resolution pipeline — resolve/prompt modes |
| `zen_os_1.jinja` → `identity_roster()` | Full household roster — build_identity_manifest |
| `zen_os_1.jinja` → `identity_manifest_loader()` | Reads `zen_identity_manifest` from household cabinet |
| `zen_dojotools_filecabinet` | All cabinet reads and writes |
| Household Cabinet | `members` drawer, `AI_Cabinet_VolumeInfo`, `zen_identity_manifest` |
| User / AI User / Family Cabinets | `AI_Cabinet_VolumeInfo.acls` — partner, family, owner entries |
