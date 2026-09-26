# PD2 reverse engineering, round 2: summary

This round covers loose ends, smaller systems, progression, maps and items, in the order 1 → 4 → 3 → 2. Each section points to the full write-up.

**Status labels:**
- **VERIFIED:** the game's own code was run natively and matched.
- **READ:** read from the disassembly only.
- **DATA:** from the tables.

**Independent audit:** every VERIFIED check was re-run by a separate checker, with 0 mismatches and every mutation caught. Its findings and 17 correction notes are in `AUDIT_2.md`.

---

## 0. Data source: which tables the live game uses (`data_sources.md`)

The live game reads `pd2data.mpq`, the copy in the Live folder. The uploaded `data.zip` is an older snapshot of it.

**What differs between the zip and the live MPQ:**
- **Map tiers:** 22 maps change tier. For example, Kurast is T1 live but T2 in the zip.
- **New items:** 10, including the maps t57, t58 and t3b, and the Corrupted-player ears.
- **Jewelry sockets:** rings and amulets can no longer have sockets.
- **Cube recipes:** 39 new CubeMain rows. The map corruption roll is 1..3000, not 1..1000.
- **Halls of Torture:** its level row changed.
- **Unchanged:** skills, missiles, monster stats, weapons, armor, ItemStatCost, uniques, sets, runewords, affixes and treasure classes. The combat and speed tools were therefore already right.

**What the loose `data/global/excel` folder is:** a separate test branch (Diablo 1/Tristram content). The uploaded saves come from it.

**Done:**
- Every data file was rebuilt from the live MPQ.
- The Advanced Stats page gives identical results for all three uploaded saves.
- The Hit Chance page picks up 21 renamed monsters and the Halls of Torture levels.
- The loot add-on picks up the new map tiers.

## 1. Loose ends

**Exploit leads (`exploit_checks.md`, VERIFIED)**
- **Power Strike:** its melee hit really does get +1000% damage, so the weapon roll is ×11.
- **Charged Strike and Lightning Strike:** they also add small hidden damage % (their bolt count and chain radius).
- **Splash:** the ordering is real, but the added damage is still resisted, so it does *not* bypass immunities.

**Projectile pierce (`pierce_ow.md`, VERIFIED)**
- Pierce is rolled once, when the missile is created, not on each hit.
- A player's roll seed is always 0, so the number of pierces is fixed by the total chance:

  | Total pierce chance | Pierces per missile |
  |---|---|
  | 0–66% | 0 |
  | 67–86% | 1 |
  | 87–94% | 5 |
  | 95%+ | 10 (Lightning Fury 6) |

**Open wounds (`pierce_ow.md`, VERIFIED)**
- There is one state per monster, and up to 3 of your own procs add together.
- **Two attackers:** another player's proc resets the stack counter to 1, so with two attackers the drain has no upper limit.

**Whirlwind (`whirlwind.md`, VERIFIED)**
- One weapon hits a flat 5 times per second, whatever the IAS or weapon speed. Dual wield hits 8.33/s, and 10/s on PvP maps.
- The first hit lands 0.12 s after the cast starts.
- Each hit takes the nearest target that wasn't hit last.

**Advanced Stats page**
- Every line where the in-game panel's math is wrong now shows the game's real value next to it.
- It also uses the live data.

## 4. Smaller systems

**Item procs, cooldowns and charges (`procs_cooldowns.md`, VERIFIED 11 modes × 40k)**
- Covers every proc trigger: on attack, striking, struck, kill, death, level-up, cast and block. For each it gives the event, the roll, whether a cooldown applies, and the skill level used.
- "On attack" fires only on melee swings that hit.
- Also covers skill cooldowns and charges.

**Shrines, wells, stamina and movement (`shrines_movement.md`)**
- Shrine types and odds, their effects and durations.
- Stamina regen and drain.
- Run and walk speed.

**Monster AI (`monster_ai.md`, VERIFIED decision functions)**
- How a monster picks its target and chooses between attacks and skills.
- PD2's changes, and the champion and unique mods that affect the AI.

## 3. Progression and maps

**Experience (`experience.md`, VERIFIED 50k cases)**
- PD2 replaces the kill-XP handler.
- It covers the level-difference scaling, player count (applied at spawn), the party split, the merc share, the death penalty and the level cap.
- **BH panel:** its "XP %" line is approximate at best and often wrong in PD2.

**Mercenaries (`mercenaries.md`, VERIFIED 7 functions)**
- **Stats:** every PD2 merc at levels 50/75/90/98. A Normal-hired merc has the same stats as a Hell one and needs 15–20% less XP.
- **Experience:** the merc stops at the player's level and at 98.
- **Revive cost:** min(15·⌊L²/2⌋, 50,000).
- **Leech:** mercs skip the difficulty divisor, so in Hell they leech 3× what a player with the same % does.
- **Skill picks and aura start:** covered in the write-up.
- **Equipment:** PD2's equip rules.
- **Bugs:**
  - A Hell-hired merc below its row's base level gets negative growth.
  - The merc panel shows wrong damage and resistances (display only).

**Map system (`maps.md`)**
- **Map-open rules:** covered in the write-up.
- **Mods:** magic maps always get 1 prefix and 1 suffix. Rare maps always get 6 affixes (VERIFIED).
- **Density and bosses:** covered in the write-up.
- **Bugs:**
  - The mod applier loses Splash's skill on about 37% of rare T1–T3 maps and 71% of T4 (VERIFIED).
  - "Physical damage as extra element" adds only +1 damage (VERIFIED).
  - T2/T3 maps mostly roll lower-tier affixes.
  - Six unique-map boss-drop mods do nothing in these DLLs.
  - Fortify gives +40% damage, not the +20% its description says.

## 2. Items

**Item generation (`items.md`, VERIFIED 6 checks)**
- **Affix level:** alvl uses the stock formula.
- **Magic items:** PD2 forces both a prefix and a suffix on higher-level items.
- **Rare affix count:** set by item level for every rare. Item level 85+ always gets 6; jewels always get 4.
- **Examples at ilvl 99:**
  - rare amulet with +2 to one class's skills: 14.7%;
  - rare diadem with +2 to one class's skills: 29.8%;
  - magic grand charm skiller: 9.05% at ilvl 90+.
- **Ethereal:** PD2 gives +25% base damage/defense, not +50%, and keeps full durability.

**Cube and crafting (`cube.md`, VERIFIED corruption engine + crafted-affix roll)**
- **Corruption:** outcome tables for every item type.
  - One-hand weapons: 25% brick.
  - White items: 50% brick.
  - A brick comes back as a new rare with its own corruption outcome.
- **Crafted items:** item level = ⌊clvl/2⌋ + ⌊base ilvl/2⌋. Crafts at item level 71+ always get 4 random affixes.
- **Bugs:**
  - **T4 map corruption:** 2/3 of attempts do nothing, and they lock the map against map orbs and further corruption.
  - **Skull reroll:** rare maps can still be rerolled with skulls.
  - **Three magic/rare T1 maps** give a white T1.

**Gambling, vendors, prices and gold (`vendors.md`, VERIFIED gamble fill + PD2 price code)**
- **Gamble odds:** unique 0.05%, set 0.1%, rare 10%. MF does nothing.
- **Circlets:** a gambled circlet can never be a diadem in PD2.
- **Stores:** vendors never sell rares, except the map Gheed event store (5 uniques at 1,000,000 each).
- **Prices:** a repair costs at most 250,000.
- **Gold:** the inventory cap is 5,000,000, and the shared stash holds 10,000,000.

---

## Still open
- Map corruption: the item level of maps re-rolled in the cube.
- Store generation and refresh timers: READ only.
- Which Npc.txt quest flags change prices.
- Merc aura activation: it seems to start only through a 1-in-101 pick. Worth one in-game look.
- Superior and low-quality rolls, and the set/unique property application: READ only.
- The vendor buy handler: the client packet chooses the transaction mode, and item ownership is checked only for buy and gamble. The rest of that path was not traced, so whether it can be abused is unknown.
