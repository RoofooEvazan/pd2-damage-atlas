# Auras and state buffs: what the owner gets (PD2 = 1.13c + ProjectDiablo.dll)

Status key as in FINDINGS.md: **READ** = read from disassembly (not executed natively). Nothing here was run in the harness;
the one PD2 behaviour change (the NOP) was confirmed at byte level against the patch record.

## Skills.txt record layout (D2Common column table, recovered with tools/txtdesc.py @ 0x6FDB3645)
| column | offset | | column | offset |
|---|---|---|---|---|
| aurafilter | +0x50 dword | | aurastate / auratargetstate | +0x80 / +0x82 word |
| aurastat1..6 | +0x54..+0x5E word | | aurastatcalc1..6 | +0x68..+0x7C |
| auralencalc / aurarangecalc | +0x60 / +0x64 | | passivestate / passiveitype | +0x94 / +0x96 |
| passivestat1..5 | +0x98..+0xA0 word | | passivecalc1..5 | +0xA4..+0xB4 |
| srvstfunc / srvdofunc | +0x2C / +0x2E | | record size | 0x23C |

## Shared helpers (D2Game)
- `AuraStatsToList` **0x6FC648E0** (eax = skill rec, args unit, statlist, skillId, level): for aurastat1..6 with stat ≥ 0,
  value = calc(unit, aurastatcalcN, skillId, level) (#10786); adds non-zero values to the list; stat 68 attackrate also sets 69.
  PD2 exports it through its import slot 0x103D4094 → PD ptr 0x104E4884.
- `PassiveStatsToList` **0x6FC647D0**: the same loop over passivestat1..5 / passivecalc1..5.
- Calc keywords: `lvl` = the level argument; `blvl` = base (hard) level `skill+0x28` of the **calc unit's own** skill entry
  (D2Common 0x6FDA14C6 → 0x6FD9E480 picks the entry with that id whose +0x34 == −1 (not item-owned), else the highest base).
  The unit does not have the skill → 0.
- Skill level (#10306, D2Common 0x6FDA05C0): lvl = base(+0x28) + bonus, clamped [0, max]. Bonus (0x6FD9FCB0) is only added when skill+0x34 == −1:
  stat127 allskills (every skill, oskills included) + 0x6FD9EBA0(unit) + skill+0x2C
  + (player and skill charclass == unit class: stat83[class] + stat188[tab] + stat97[skill]; other class: stat97[skill];
  non-player: stat97, capped at 3 when base > 0) + stat126 (element) + stat107[skill].

## Q1. Paladin / item / merc auras (aura=1)
Aura timer: start **0x6FCCDB30** (needs `aura` flag and aurastate ≥ 0; timer type 9, level fixed at start). Every tick it calls srvdofunc.
Area iteration **0x6FCC0C70** (range, aurafilter, callback) **always skips the owner** (0x6FCC0D90: `cmp edi,[owner]`).
Per-target callback **0x6FCBA1A0** creates/refreshes the state list (0x6FCC00B0, PD → 0x102BFE20), sets the 6 precomputed
(stat, value) pairs, then stat 350/351 (skill id / level). Special cases in the callback:
- stat 110 is not written; it shortens the target's poison (0x6FCB9B60). This is Cleansing's aurastat1.
- stats with the ItemStatCost "direct" flag are added to the target's base stat, capped by its maxstat. This is how Prayer's `hitpoints` heals.
- stats 67 and 68 are floored for targets (0x6FCFAE20); 68 also writes 69.
- negative resistances against monsters with resist ≥ 100: stock divides by 5 (0x6FC6E230), PD2 by 2 (0x102C0540). This is Conviction against immunes.
- if the skill has a passivestate, that state's statlist is removed from the target (#10871/#11111/#11108) and the state flag is set.

### srvdofunc 65: Might, Prayer, Resist*, Thorns, Defiance, Blessed Aim, Cleansing, Concentration, Vigor, Meditation, Fanaticism, Salvation, druid spirit auras (0x6FCBA8D0)
1. The 6 aurastats are evaluated once with (owner, skillId, aura level). They are skipped (all 0) only when the owner is a player whose current mana (stat 8) is below the mana cost (#10090).
2. **Owner:** the callback is called directly on the owner with `aurastate` (no filter check). The owner gets **aurastat1–6**.
3. **Owner passives:** then passivestat1–5 are added to that same owner list (value ≠ 0, stat > 0).
   - Conditions: the list was created, and owner mana > mana cost (strictly; an owner with 0 mana gets none).
   - Stock D2Game also required `passivestate ≤ 0` (0x6FCBAA75 `jg`). **PD2 NOPs that jump** (code-init record: D2Game rva 0x9AA75, 6 × 0x90). So in PD2 the owner gets passivestats for *every* do-65 aura.
4. **Others:** if auratargetstate ≥ 0, units in `aurarangecalc` that pass `aurafilter` (owner excluded) get **auratargetstate with aurastat1–6 only**. This covers party players and mercenaries, and allied monsters when bit 0x2 is set. They never get passivestats.
5. Passivestate skills (Prayer, Resist Fire/Cold/Lightning, Defiance, Blessed Aim, Cleansing, Vigor, Meditation):
   - D2Common #10056 (0x6FDA2480) builds the passivestate list from passivestat at GetSkillLevel(unit, skill, 1). It returns without building it while the owner has the aurastate, and the callback removes it when the aura lands.
   - When the owner's aura state ends, the removal callback 0x6FCB8450 → 0x6FCBE370 re-runs #10056 for every skill with a passivestate.
   - Net result for the owner: passivestats always (from the passive list when the aura is off, from the aura list when it is on), plus aurastats while the aura is on. Nothing is counted twice.
   - Side effect (READ): another Paladin standing in your Prayer loses his own passive_regen list until his next refresh.

### srvdofunc 66 (Holy Fire/Shock, Sanctuary, Conviction) and 81 (Holy Freeze) — 0x6FCBAF50 / 0x6FCBABC0
- **Owner:** aurastate with **passivestat1–5 only**. Values are taken from passivecalc; no passivestate test.
- **Enemies:** those passing the filter (hostile bit 0x8000) get auratargetstate with **aurastat1–6**. Allies get nothing.
- srvdofunc 82 (Redemption) carries no stats.

### Item auras (stat 151 item_aura) and mercenary auras
- The stat-change callback (0x6FCF995E region) calls start 0x6FCCDB30 with level = stat value, or end 0x6FCCD050.
- The same do-func runs, so owner and target rules are exactly as above, with lvl = item level.
- `blvl` = the owner's hard points in that skill id, or 0 (e.g. The Beast Fanaticism worn by a non-Paladin → blvl 0).
- A mercenary owner is handled by the same code. It needs mana > cost to get the passivestats.

## Q2. Non-aura state buffs
Level passed to the do-func = the cast level. Calcs use the caster as unit, so `lvl` = total level (+skills, formula above) and `blvl` = the caster's own hard points in that skill id.
| srvdofunc | code | caster gets | others |
|---|---|---|---|
| 18 (Shiver/Chilling Armor, Bone/Cyclone Armor, Quickness=Burst of Speed, Fade, Venom, Holy Shield, Holy Sword, SelfAura 555/556) | 0x6FC628F0 | aurastate list = aurastat1–6 **+ passivestat1–5** (Fade's `fade`=2 comes from here) | — |
| 23 (Energy Shield, Blaze) | 0x6FC62730 | aurastate list = passivestat1–5 only | — |
| 68 (Shout, Battle Orders, Battle Command, Battle Cry, BattleOrdersCTA 360) | 0x6FC49720; per-unit apply patched → PD 0x102C9F00 | aurastate list = aurastat1–6 (via 0x6FC648E0) | party in range: same state, aurastat1–6, calcs evaluated with the caster |
| 25 (Enchant, Cold Enchant) | PD 0x102F98D0 (replaces stock 0x6FC62570) | **auratargetstate** list = aurastat1..n, stopping at the first empty aurastat column | every unit in aurarangecalc passing aurafilter (0x10003 → allied players/monsters) |
| 116 (Werewolf, Werebear, Vampire/Delerium form), 120 (Feral Rage, Maul) | 0x6FC66210 / 0x6FC66000 | aurastat1–6 | — |
| 47 (Cloak of Shadows; PD wrapper 0x102F9F00 → stock 0x6FCB4B10) | | aurastate list = passivestat1–5 (item_armor_percent) | enemies: auratargetstate + aurastats |
| srvstfunc 38 (Whirlwind, Blade Dance) | 0x6FC48290 | aurastat + passivestat | — |
- CTA oskill: Runes.txt gives `oskill Battle Orders` (skill 149, not 360). A non-Barbarian has base 0 → blvl 0. lvl = allskills + stat97 + stat107 (+ element).
  - A Barbarian with hard points in BO uses blvl = those hard points and lvl = hard + all bonuses including stat97.
  - Only BC's `1+blvl/10` and BattleOrdersCTA's length calc read blvl.
- Frenzy/Berserk (do 9, st 78 → PD 0x103014B0) were not traced. They have no passivestats, so the answer is aurastats only either way.

## Q3. aurafilter and the owner
aurafilter never decides whether the owner is affected:
- The area iteration always excludes the owner.
- The do-func applies the owner's own state directly: do 65, 66, 81, 18, 23, 47, 68 and PD's 25 do this unconditionally.

The rule "self if the filter lacks 0x80 and 0x100" is **wrong**. Replace it with the do-func tables in Q1/Q2.
Filter bits (0x6FCC0370): 0x1 players, 0x2 monsters, 0x8 objects, 0x10 missiles, 0x20 items, 0x1000 dead only, 0x4 (monster extra check),
0x80 needs unit flag +0xC4 bit 2, 0x400 needs bit 3, 0x100 excludes units in town, 0x10000 allies of the owner (merc → its player), 0x8000 enemies,
0x20000 an owner/target relation check (0x6FC2A754), 0x80000 skips units in state 0x56, 0x200 line of sight, 0x2000 room check, 0x4000 extra check.
Meditation (0x12001) has no monster bit, so mercenaries don't receive it.

## Q4. Aura count, item auras, equipped skills
- The aura start keys its timer on its args (0x6FCAE900 removes the existing type-9 timer for the same key). States are one list per state id, so two sources of the same aura never stack: the list is replaced by whichever applied last (0x6FCC00B0 compares skill/level/owner).
- No PD2 limit on the number of auras was found. PD2 replaces the immediate dispatch (0x6FCC19A0 → PD 0x102C63C0, patched at 0x6FCCDBE5); it only adds checks (town/map, skills 249/250/593 packet).
- stat 191 item_skillonequip (PD 0x102C6030): if the unit lacks the skill, PD adds it with base 0, then starts the aura at level = stat value (PD 0x102EF480 → 0x6FCCDB30). Unequip ends it.
  - Used by Chilling Armor SelfAura 555 and Quickness SelfAura 556, both srvdofunc 18: aurastat + passivestat, blvl = 0 unless the unit owns hard points in *that* id.
- PD2 do-func overrides (srvdofunc table 0x6FD274A8 written by PD 0x102BE412): 4, 6, 8, 25, 39, 44–51, 54–57, 64, 71, 104, 112–115, 118, 119, 139, 143, 148, 153–189.
  - PD2 srvstfunc overrides: 28, 29, 37, 66–78.
  - None of 9, 18, 23, 65, 66, 68, 81, 82, 116 or 120 is overridden. The aura path changes are: the NOP above; the callback's state create (→0x102BFE20) and resist-immune divisor (→0x102C0540); the #10819 call (→0x10269200, skipped in state 0x12); and the D2Common find-list-by-state (→0x102692B0).

## Caveats
- Everything is READ. The NOP is certain; the argument roles (unit, level) were traced by stack offsets.
- The do 116/120/9 contents were only skimmed (they call 0x6FC648E0).
- The mana > cost gate on do-65 passives uses stat 8 in 8.8 fixed point against #10090's result. Every PD2 aura costs 0, so in practice the owner only needs mana > 0.
- The code was checked against the client install's D2Game. Realm servers may differ.
