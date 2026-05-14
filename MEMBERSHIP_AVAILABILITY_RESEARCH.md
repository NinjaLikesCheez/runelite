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

---

## Addendum: OSRS Wiki `Free-to-play_transportation` as an additional data source

User-requested page reviewed:

- https://oldschool.runescape.wiki/w/Free-to-play_transportation

Machine-friendly fetch endpoints (recommended for ingestion/diff tooling):

- `action=parse&prop=wikitext`  
  `https://oldschool.runescape.wiki/api.php?action=parse&page=Free-to-play_transportation&prop=wikitext&format=json`
- `action=parse&prop=sections`  
  `https://oldschool.runescape.wiki/api.php?action=parse&page=Free-to-play_transportation&prop=sections&format=json`

### What this wiki page gives you

The page is useful as a **curated F2P transport catalog**, grouped by player-facing categories:

- Teleportation spells
- Minigame teleports
- Canoes
- Items
- NPC transports
- Death/respawn travel
- Runic altars
- Stronghold shortcuts
- Misc travel-adjacent mechanics

It also contains editorial context that game data usually does not: one-way warnings, safety notes on PvP worlds, practical utility notes, and edge-case methods.

### Coverage matrix: Wiki vs current transport data

Below is a practical mapping against current data in this workspace and the short-path TSV set:

| Wiki method/category | In shortest-path TSVs? | In RuneLite core data? | Notes |
| --- | --- | --- | --- |
| F2P standard spell teleports (Lumbridge home/Varrock/Lumbridge/Falador) | Yes (`teleportation_spells.tsv`) | Yes (`TeleportLocationData`) | Already represented with runes, levels, and wilderness caps. |
| F2P minigame teleports (Castle Wars/Clan Wars/LMS -> Ferox) | Yes (`teleportation_minigames.tsv`) | Partially (DB metadata available via `DBTableID.MinigameTeleport.COL_MEMBERS_ONLY`) | Good candidate to derive explicit F2P vs members from DB rows. |
| Canoes (incl. Ferox + wilderness pond) | Yes (`canoes.tsv`) | Yes (`TransportationPointLocation` has canoe points) | Directionality/requirements are encoded in TSV. |
| Chronicle / Skull sceptre / Cowbell amulet | Yes (`teleportation_items.tsv`) | Yes (`TeleportLocationData`) | Good overlap; still lacks explicit membership field in transport rows. |
| Disk of returning (Blackhole event travel) | No match in TSV | Not in world map transport data; item constant exists (`ItemID.DISK_OF_RETURNING`) | Coverage gap from wiki perspective. |
| Port Sarim <-> Musa Point (Captain Tobias / Customs officer) | Yes (`ships.tsv`) | Yes (`TransportationPointLocation` ship points) | Encoded with coin cost in TSV. |
| Cabin Boy Colin (Rimmington <-> Corsair Cove) | Yes (`ships.tsv`, quest-gated) | Yes (`TransportationPointLocation`) | Already modeled well. |
| Shantay "outlaw to Port Sarim jail" | No match | No clear transport entry | Wiki includes as niche route; currently missing. |
| Count Check one-time teleport | No match | No clear transport entry | Missing and semantically "single-use". |
| Aubury / Sedridor to rune essence mine | No match in current TSV set | World map has generic "Teleport to Rune Essence" points, but not this F2P pair as explicit transport rows | Gap/ambiguity; likely low route value but still wiki-listed. |
| Ned to Crandor (during Dragon Slayer I) | No match | Present in world map transport points (`PORTSARIM_TO_CRANDOR`) | Gap between RuneLite map hinting and shortest-path graph rows. |
| GE recruiters to Castle Wars lobbies | No clear match | No clear transport entry | Missing; possibly low practical value due overlap with minigame teleports. |
| Sorceress self-teleport near Al Kharid | No match | No clear transport entry | Missing and low utility. |
| Death respawn travel | No (except POH respawn portal modeling, not death-route simulation) | Not represented as route edges | Usually out-of-scope for deterministic routing. |
| Runic altar access via talisman/tiara ruins | Yes (`transports.tsv`, altar portal entries) | Partially represented across map/object data | Encoded in path graph already. |
| Stronghold of Security shortcut portals | Yes (`teleportation_portals.tsv`, `transports.tsv`) | Partially represented in map/objects | Encoded in path graph already. |

### What is already encoded in RuneLite (useful for harmonization)

Even when the wiki page is the discovery source, RuneLite already has robust primitives to verify/classify:

1. **Membership truth for items**: `ItemComposition.isMembers()`
2. **Membership truth for skills**: `Skill` enum `members` flag
3. **World context**: `WorldType.MEMBERS`
4. **DB membership flags**: e.g. `Quest.COL_MEMBERS`, `MinigameTeleport.COL_MEMBERS_ONLY`
5. **Transport/map hints**: world map transport enums (`TeleportLocationData`, `TransportationPointLocation`)

So the wiki should be treated as a **coverage/discovery layer**, not the final authority for machine checks.

### What is not encoded cleanly today

1. A unified, first-class `membership` property on every shortest-path transport row.
2. A canonical F2P transport whitelist in one place (wiki has one; TSVs are broader mixed-world data).
3. Niche/edge transport methods from the wiki (notably Disk of returning, Shantay jail, Count Check, some NPC teleports).
4. Semantics like **single-use**, **exploit-like travel**, **risk/safety context**, and rich editorial caveats.
5. Provenance/versioning per row ("from cache", "from wiki", "manual override"), which is needed for safe reconciliation.

### Harmonization strategy (recommended)

1. **Introduce provenance + membership columns in transport schema**
   - Add `Membership` (`F2P | MEMBERS | AUTO | UNKNOWN`)
   - Add `Source` (`CACHE`, `RUNELITE_MAP`, `WIKI`, `MANUAL`)
   - Add optional `Repeatability` (`REPEATABLE`, `SINGLE_USE`, `CONDITIONAL`)

2. **Build a wiki importer as candidate-generator, not direct truth**
   - Parse `action=parse&prop=wikitext` output.
   - Normalize rows into candidate transport edges.
   - Do not auto-accept rows without identity matching.

3. **Match candidates to existing rows**
   - Primary key heuristic: destination coordinates + interaction type + key requirement signals.
   - Emit: `matched`, `new_candidate`, `conflict`.

4. **Conflict resolution policy**
   - RuneLite/cache-derived IDs, coords, membership flags win for machine logic.
   - Wiki narrative fields (warnings, one-way notes, practical notes) become annotations.
   - Manual override list for known exceptions.

5. **Automated diff report in CI**
   - "Wiki rows not in TSV"
   - "TSV rows not in wiki F2P set"
   - "Membership disagreement after AUTO derivation"

6. **Runtime gating remains RuneLite-driven**
   - Final usability check must still use world/account state:
     - world type
     - skill/quest state
     - varbits/varps
     - item possession + item membership

### High-value first sync pass (low risk)

Start with rows that are easy and unambiguous:

- Add explicit membership to existing F2P spell/canoe/minigame/item rows.
- Add missing wiki rows with clear mechanics:
  - Disk of returning (if still in-game/available for target mode)
  - Ned -> Crandor (Dragon Slayer I state-gated)
  - Count Check (mark as `SINGLE_USE`)
- Leave death-based routing and exploit-like methods behind a feature flag or separate transport profile.

### Bottom line

The OSRS Wiki page is valuable for **completeness and edge-case discovery**, but it should be integrated as a **curated upstream signal** and reconciled against RuneLite/cache truth.  
That gives you better coverage without sacrificing deterministic, game-state-correct routing behavior.
