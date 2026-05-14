# Membership Availability Research for RuneLite Shortest-Path Routing

## Scope

This research answers: **how the project determines whether a route option is F2P vs members**, with emphasis on:

- teleport spells
- teleport items/tabs/jewellery
- agility shortcuts and other transports

I reviewed both:

1. This workspace (RuneLite core APIs and world map transport datasets)
2. The Shortest Path plugin source (`Skretzo/shortest-path`), which is the route-planning implementation used in Plugin Hub

---

## Executive Summary

### Current state (important)

In Shortest Path, **membership is not a first-class field** in transport data or parser logic.  
Transport availability is inferred from:

- `Skills`
- `Quests`
- `Items`
- `Varbits` / `VarPlayers`
- config toggles

This means membership behavior is currently **implicit and incomplete**.

### Key known limitation in code/data

There is an explicit TODO-style comment in transport data:

- `src/main/resources/transports/teleportation_portals.tsv:6`  
  _"Disable the big doors for now as we can't distinguish between f2p and members' worlds atm."_

So the project already documents a membership-classification gap.

---

## How Shortest Path currently determines availability

## 1) Data model: TSV schema has no `Members` column

Transport rows are parsed from TSV with canonical fields:

- `Origin`, `Destination`
- `Skills`
- `Items`
- `Quests`
- `Varbits`, `VarPlayers`
- etc.

See parser-facing schema doc:

- `docs/Transport-TSV-format.md` (canonical columns list; no members column)
- `src/main/java/shortestpath/transport/parser/TransportRecord.java` (recognized field names)

## 2) Runtime gating pipeline

Core runtime checks in:

- `src/main/java/shortestpath/pathfinder/PathfinderConfig.java`

Main checks are:

- `hasRequiredLevels(...)` (skill thresholds)
- `completedQuests(...)` (quest finished)
- var checks (`Varbits` / `VarPlayers`)
- `hasRequiredItems(...)`
- transport-type enable/disable config (`TransportTypeConfig`)

There is **no direct `isMembers` check** using item metadata/world type for normal routing.

## 3) Teleportation item settings can bypass requirements

`PathfinderConfig.hasRequiredItems(...)` has modes where teleport item types return `true` immediately:

- `ALL`, `ALL_NON_CONSUMABLE`, `UNLOCKED`, `UNLOCKED_NON_CONSUMABLE`

Reference:

- `PathfinderConfig.java:899-914`

This can amplify misclassification if membership is not separately filtered.

---

## Transport-type-specific membership behavior

## Teleport spells

Data source:

- `src/main/resources/transports/teleportation_spells.tsv`

Header:

- `Destination, Items, Skills, Quests, ... , Varbits, VarPlayers`

What this gives you:

- rune requirements
- magic level
- explicit quest gates
- spellbook varbit gates (`4070=...`)

What it does **not** give you:

- explicit members-only flag

Practical impact:

- Standard spellbook contains both F2P and members spells; not all members spells are quest-gated.
- So relying only on level/runes/quest/varbits leaves ambiguous cases.

## Teleportation items/tabs/jewellery

Data source:

- `src/main/resources/transports/teleportation_items.tsv`

What is encoded:

- required item IDs / item groups
- optional skill/quest/var conditions
- consumable flag

What is missing:

- explicit item membership metadata

Practical impact:

- If an item is present (or if config mode bypasses inventory checks), the plugin may consider it usable without proving F2P legality.

## Agility shortcuts

Data source:

- `src/main/resources/transports/agility_shortcuts.tsv`

What is encoded:

- mostly `Skills` (often `X Agility`)
- item/var conditions where needed

Membership implication:

- Agility itself is a members skill in RuneLite (`Skill.AGILITY` has `members=true`).
- But this implication is not modeled as an explicit transport membership flag.

## Other transport groups (boats, ships, gliders, carpets, portals, etc.)

Each has TSV requirement-based gating, but still no universal members field.

---

## What RuneLite core already provides for robust F2P/P2P classification

These are available in this workspace and can be leveraged:

## 1) Item membership metadata (strong signal for teleport items/tabs)

- `ItemComposition.isMembers()`  
  (`runelite-api/src/main/java/net/runelite/api/ItemComposition.java:123-127`)

Backed by cache definition opcode:

- item opcode `16 -> members=true`  
  (`cache/src/main/java/net/runelite/cache/definitions/loaders/ItemLoader.java:124-127`)

## 2) Skill membership metadata

- `Skill` enum has `members` boolean  
  (`runelite-api/src/main/java/net/runelite/api/Skill.java:35-75`)

## 3) World membership context

- `WorldType.MEMBERS` exists for world-state checks  
  (`runelite-api/src/main/java/net/runelite/api/WorldType.java:37-41`)

## 4) DB table membership columns (underused but valuable)

RuneLite `DBTableID` includes relevant membership fields, e.g.:

- `Quest.COL_MEMBERS`  
  (`runelite-api/.../DBTableID.java`, `Quest` table block)
- `MinigameTeleport.COL_MEMBERS_ONLY`  
  (`runelite-api/.../DBTableID.java`, `MinigameTeleport` table block)

Client APIs to read DB values:

- `Client.getDBTableField(...)`
- `Client.getDBRowsByValue(...)`  
  (`runelite-api/src/main/java/net/runelite/api/Client.java:987,995`)

## 5) World-map element membership

- `WorldMapElementDefinition.membersOnly`  
  (`cache/src/main/java/net/runelite/cache/definitions/WorldMapElementDefinition.java:36`)

Loaded from map element stream:

- `WorldMapElementLoader` reads `membersOnly` boolean  
  (`cache/.../WorldMapElementLoader.java:39`)

---

## Important edge cases you should account for

1. **Standard spellbook has mixed F2P and members spells**  
   A spellbook check alone is insufficient.

2. **Quest-gating is not a reliable proxy for membership**  
   Not all members-only options have a quest requirement.

3. **Inventory presence != usable in F2P**  
   Membership usability must be checked separately from possession.

4. **Agility requirement is a strong members signal**  
   But some transports may be members-only for reasons beyond skills.

5. **Portal/interaction data can be world-dependent**  
   The plugin already has at least one disabled case due to F2P/P2P ambiguity.

---

## Recommended implementation direction

## A) Add explicit membership field to transport schema

Add an optional TSV column (e.g. `Membership`) with values like:

- `F2P`
- `MEMBERS`
- `AUTO` (default)

Then resolve `AUTO` via deterministic rules.

## B) Implement `AUTO` derivation hierarchy

Suggested order:

1. If `Skills` includes a members skill -> `MEMBERS`
2. If `Items` include any `ItemComposition.isMembers()==true` item -> `MEMBERS`
3. If transport quest set is known-members-only (from quest DB metadata) -> `MEMBERS`
4. If minigame teleport DB row says members-only -> `MEMBERS`
5. Else `F2P` (or `UNKNOWN` if you want strict mode)

## C) Keep a runtime world/account gate separate from static classification

Even with static classification, final availability should also apply current-world rules (`WorldType.MEMBERS` context).

## D) Add diagnostics

Expose why a transport was filtered (`members-only`, `missing quest`, `missing item`, etc.) for easy validation.

---

## Quick conclusion

Today, the path project determines F2P/P2P **indirectly** through requirement fields, not through a dedicated membership model.  
That works for many routes but is not sufficient for complete correctness (especially teleport spells/items and portal-world cases).

The good news is RuneLite already exposes the metadata needed (`ItemComposition.isMembers`, skill membership, DB table membership fields, world type), so this can be made robust without guessing.
