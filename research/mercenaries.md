# Mercenaries (hirelings) in PD2 (1.13c + ProjectDiablo.dll)

Scope: stats by level, hiring, skills and how often they are used, equipment rules, damage, defenses, leech,
experience, revive and hire costs, and auras on the party. JS model: `adv/re/mercenaries.js`
(`mercStats({type, act, diff, level}, opts)`, `mercBase`, `hireOffer`, `hireCost`, `reviveCost`, `addMercExp`,
`killExpShare`, `mercLeech`, `skillChoice`, `attackGate`, `canEquip`, `classItemOK`, `serverResist`).
Related: `minions.md` §2 (row pick, setter, panel), `auras.md`, `dmg_A_attack.md`, `dmg_B_pipeline.md`, `dmg_D_incoming.md`, `mf_misc.md` §3.

Status: **VERIFIED** = the real code ran in the harness and matched the model on every case (with mutation checks);
**READ** = disassembly only; **DATA** = PD2 tables. `tdiv` = C division toward zero; `>>` = arithmetic shift (floor).

## Native checks
| harness | game code | cases | result | mutations (mismatches) |
|---|---|---|---|---|
| `harness/merc.c` + `merc.py` mode 1 | D2Common #11097 0x6FD7CC60 resurrect cost | 2,100 | 0 mismatches | 15·L²/2 → 853; cap 60000 → 351 |
| mode 2 | D2Common #10458 0x6FD7CCC0 hire-list entry (real hireling.bin, real #10453) | 20,000 | 0 | level window +1 → 19,158; trunc instead of floor → 7,003; no Gold floor → 8,605 |
| mode 3 | D2Game 0x6FCFDCB0 merc experience gain | 20,000 | 0 | no player-level gate → 6,320; gain ×1 instead of ×2 → 2,915 |
| mode 4 | D2Game 0x6FCFBA40 leech (player vs merc attacker) | 20,000 | 0 | merc divided by LifeStealDivisor → 4,020 |
| mode 5 | D2Game 0x6FC3C4E0 merc skill pick (real RNG, real #11156, real skills.bin flags) | 39,938 (62 skipped) | 0 | ChancePerLvl rounding → 15,039; default threshold +1 → 1,861 |
| `harness/mercpd.c` + `mercpd.py` | PD 0x102C0BE0 equip rule, every base item × 8 classes × random other hand | 19,680 | 0 | barb swords only → 348; A3 2-handers → 57; rogue bows only → 61 |
| same | PD 0x102C0E50 class-item rule | 20,000 | 0 | A3 necro items → 310 |
| `minions_harness/run_checks.py d2game` (re-run) | D2Game 0x6FC68BA0 merc stats + #11156 row pick + skills | 20,000 | 0 | (existing check) |

Stubs: stat get/set, unit/player-data lookups, packets, item-type test (driver-built ItemTypes closure over type and
type2, as #10744 does), body-slot lookup. Images are stock 1.13c; PD2 has no patch record inside 0x6FC68BA0, 0x6FCFDCB0,
0x6FCFBA40, #10458, #11097 (only the ItemStatCost-size immediates at 0x6FD7CC89/0x6FD7CD9D, which sit in the stubbed
stat read). In 0x6FC3C4E0 PD redirects only the two exit calls (set-skill 0x6FC3C73D → 0x102C8190, melee 0x6FC3C80D), which are stubbed.

## 1. Stats by level

### 1a. Row choice (VERIFIED, `minions.md` §2a)
- A merc keeps its Hireling `Id` for life (saved in the .d2s). Rows are keyed by (`Version` 100, `Id`). D2Common #11156
  (0x6FDA32C0) takes the first row of that Id, then each later row of the same Id whose `Level` ≤ merc level.
- PD2 Ids 0–38 (DATA): Normal Ids have three level brackets, Nightmare two, Hell one (e.g. Rogue Fire: Id 0 rows 3/36/67,
  Id 2 rows 36/67, Id 4 row 67). The bracket rows of the Normal, NM and Hell Ids of one line are **identical except
  `Exp/Lvl`** (Normal 100/110 … Hell 120/140) and A5 `Share` (unused by the server). Two A3 NM rows swap Chance2
  (Cold 75 vs 70, Lightning 70 vs 75).

### 1b. Growth — D2Game 0x6FC68BA0 (VERIFIED, 20,000 cases)
```
d = level − row.Level                                        // negative for a merc below its row
life  = max(40, HP + HP/Lvl·d)          def = max(0, Defense + Def/Lvl·d)     AR = max(0, AR + AR/Lvl·d)
str   = max(10, Str + tdiv(Str/Lvl·d, 8))    dex = max(10, Dex + tdiv(Dex/Lvl·d, 8))   // /Lvl columns are "per 8 levels"
dmg   = max(0, Dmg-Min + tdiv(Dmg/Lvl·d, 8)) – max(1, Dmg-Max + tdiv(Dmg/Lvl·d, 8))   // stored in stats 23/24 (2-hand pair)
resist (fire/cold/light/poison) = max(0, Resist + tdiv(Resist/Lvl·d, 4))              // "per 4 levels"
skill k: if level ≥ Skills.reqlevel and Mode_k < 16: lvl = min(32, Level_k + ((LvlPerLvl_k·d) >> 5)), skipped if ≤ 0
life regen (stat 74) = maxlife_raw / 2000
```
So HP, Def and AR grow in whole points per level; Str/Dex/Dmg store eighths, Resist quarters, skill levels 1/32.
Note the skill **reqlevel gate**: e.g. Holy Shock/Sanctuary/Meditation (24), Vigor/Dodge (18), Blessed Aim/Cleansing (12).

### 1c. Hiring (VERIFIED #10458; list handling READ)
- The NPC hire screen and the server both call D2Common #10458 (0x6FD7CCC0) with the slot's seed, the current act and
  difficulty. Candidates = all rows with that `Act` and `Difficulty` at the first row's `Level` (#10453), i.e. the base
  rows of every line (A1 Normal: Fire/Ice/Phys). LCG (lo = seed, hi = 0x29A): row = cand[rand % n]; then
  `level = max(2, clvl − 5 + rand % 5)` → **clvl−5 … clvl−1**.
- The offer shows life/str/dex/def/damage with the same growth but **floor** (`sar 3`) instead of trunc, and the base
  `Resist` without growth. Cost: `max(Gold, tdiv(Gold·(100 + 15·d), 100))`, d = level − base row Level.
  Starting experience = Exp(level) (below).
- Server hire (D2Game 0x6FCE1510): Kashya needs quest 2 and Qual-Kehk quest 36 (READ); gold is taken from the
  inventory, then the stash (0x6FCDCC60, READ). PD2 replaces the seller test (0x6FCDD346 → 0x102EEF50) to add the Act 4 NPC.

### 1d. Level cap (VERIFIED 0x6FCFDCB0)
A merc receives no experience while **its level ≥ the player's level** or ≥ 98 (`#10066 max level − 1`), so hired at
≤ clvl−1 it can only reach the player's level (by one overshooting gain) and never exceeds 98 through play. Loading
from a save (0x6FC75CF1, READ) allows 99.

### 1e. Table (VERIFIED formula; PD2 data). Normal Ids; NM/Hell Ids give the same numbers once at/above their row
`2H base dmg` is stats 23/24 before items; resist is each of fire/cold/light/poison before items (server cap 90, §5).
| Id | merc | lvl | row | life | def | str | dex | AR | 2H base dmg | resist | skills (base level) |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 0 | Rogue Scout Fire | 50 | 36 | 594 | 489 | 94 | 139 | 742 | 11–13 | 90 | Vigor 14, Merc Fire Arrow 16, Exploding Arrow 16, Dodge 5, Evade 5 |
| 0 | Rogue Scout Fire | 75 | 67 | 1140 | 920 | 126 | 189 | 1438 | 22–24 | 131 | Vigor 18, Merc Fire Arrow 24, Exploding Arrow 24, Dodge 5, Evade 5 |
| 0 | Rogue Scout Fire | 90 | 67 | 1590 | 1250 | 144 | 219 | 1978 | 29–31 | 149 | Vigor 18, Merc Fire Arrow 29, Exploding Arrow 29, Dodge 5, Evade 5 |
| 0 | Rogue Scout Fire | 98 | 67 | 1830 | 1426 | 154 | 235 | 2266 | 33–35 | 159 | Vigor 18, Merc Fire Arrow 31, Exploding Arrow 31, Dodge 5, Evade 5 |
| 1 | Rogue Scout Ice | 50 | 36 | 594 | 489 | 94 | 139 | 742 | 11–13 | 90 | Meditation 15, Merc Cold Arrow 16, Freezing Arrow 16, Dodge 5, Evade 5 |
| 1 | Rogue Scout Ice | 75 | 67 | 1140 | 920 | 126 | 189 | 1438 | 22–24 | 131 | Meditation 21, Merc Cold Arrow 24, Freezing Arrow 24, Dodge 5, Evade 5 |
| 1 | Rogue Scout Ice | 90 | 67 | 1590 | 1250 | 144 | 219 | 1978 | 29–31 | 149 | Meditation 26, Merc Cold Arrow 29, Freezing Arrow 29, Dodge 5, Evade 5 |
| 1 | Rogue Scout Ice | 98 | 67 | 1830 | 1426 | 154 | 235 | 2266 | 33–35 | 159 | Meditation 29, Merc Cold Arrow 31, Freezing Arrow 31, Dodge 5, Evade 5 |
| 6 | Desert Merc Comb | 50 | 43 | 805 | 552 | 129 | 101 | 596 | 16–23 | 98 | Thorns 12, Jab 16 |
| 6 | Desert Merc Comb | 75 | 75 | 1430 | 1027 | 173 | 139 | 1196 | 33–40 | 142 | Thorns 18, Jab 24 |
| 6 | Desert Merc Comb | 90 | 75 | 2030 | 1447 | 199 | 161 | 1736 | 42–49 | 157 | Thorns 18, Jab 24 |
| 6 | Desert Merc Comb | 98 | 75 | 2350 | 1671 | 213 | 173 | 2024 | 47–54 | 165 | Thorns 18, Jab 24 |
| 7 | Desert Merc Def | 50 | 43 | 805 | 552 | 129 | 101 | 596 | 16–23 | 98 | Defiance 12, Jab 16 |
| 7 | Desert Merc Def | 75 | 75 | 1430 | 1027 | 173 | 139 | 1196 | 33–40 | 142 | Defiance 18, Jab 24 |
| 7 | Desert Merc Def | 90 | 75 | 2030 | 1447 | 199 | 161 | 1736 | 42–49 | 157 | Defiance 18, Jab 24 |
| 7 | Desert Merc Def | 98 | 75 | 2350 | 1671 | 213 | 173 | 2024 | 47–54 | 165 | Defiance 18, Jab 24 |
| 8 | Desert Merc Off | 50 | 43 | 805 | 552 | 129 | 101 | 596 | 16–23 | 98 | Blessed Aim 12, Jab 16 |
| 8 | Desert Merc Off | 75 | 75 | 1430 | 1027 | 173 | 139 | 1196 | 33–40 | 142 | Blessed Aim 18, Jab 24 |
| 8 | Desert Merc Off | 90 | 75 | 2030 | 1447 | 199 | 161 | 1736 | 42–49 | 157 | Blessed Aim 18, Jab 24 |
| 8 | Desert Merc Off | 98 | 75 | 2350 | 1671 | 213 | 173 | 2024 | 47–54 | 165 | Blessed Aim 18, Jab 24 |
| 15 | Iron Wolf Fire | 50 | 49 | 484 | 263 | 93 | 75 | 447 | 18–24 | 86 | Cleansing 17, A3 Merc Fire Ball 11, A3 Mastery Fire 15, A3 Merc Meteor 10 |
| 15 | Iron Wolf Fire | 75 | 49 | 934 | 588 | 124 | 100 | 1047 | 31–37 | 130 | Cleansing 30, A3 Merc Fire Ball 19, A3 Mastery Fire 25, A3 Merc Meteor 17 |
| 15 | Iron Wolf Fire | 90 | 79 | 1303 | 915 | 143 | 115 | 1539 | 38–44 | 149 | Cleansing 30, A3 Merc Fire Ball 24, A3 Mastery Fire 31, A3 Merc Meteor 22 |
| 15 | Iron Wolf Fire | 98 | 79 | 1519 | 1115 | 153 | 123 | 1827 | 42–48 | 157 | Cleansing 30, A3 Merc Fire Ball 28, A3 Mastery Fire 32, A3 Merc Meteor 24 |
| 16 | Iron Wolf Cold | 50 | 49 | 484 | 263 | 93 | 75 | 447 | 18–24 | 86 | Prayer 12, A3 Merc Ice Blast 10, A3 Mastery Cold 15, A3 Merc Blizzard 9 |
| 16 | Iron Wolf Cold | 75 | 49 | 934 | 588 | 124 | 100 | 1047 | 31–37 | 130 | Prayer 18, A3 Merc Ice Blast 16, A3 Mastery Cold 25, A3 Merc Blizzard 13 |
| 16 | Iron Wolf Cold | 90 | 79 | 1303 | 915 | 143 | 115 | 1539 | 38–44 | 149 | Prayer 22, A3 Merc Ice Blast 21, A3 Mastery Cold 31, A3 Merc Blizzard 17 |
| 16 | Iron Wolf Cold | 98 | 79 | 1519 | 1115 | 153 | 123 | 1827 | 42–48 | 157 | Prayer 25, A3 Merc Ice Blast 24, A3 Mastery Cold 32, A3 Merc Blizzard 19 |
| 17 | Iron Wolf Ltng | 50 | 49 | 484 | 263 | 93 | 75 | 447 | 18–24 | 86 | Holy Shock 15, A3 Merc Lightning 12, A3 Mastery Ltng 15, Merc Static Field 12 |
| 17 | Iron Wolf Ltng | 75 | 49 | 934 | 588 | 124 | 100 | 1047 | 31–37 | 130 | Holy Shock 21, A3 Merc Lightning 20, A3 Mastery Ltng 25, Merc Static Field 17 |
| 17 | Iron Wolf Ltng | 90 | 79 | 1303 | 915 | 143 | 115 | 1539 | 38–44 | 149 | Holy Shock 23, A3 Merc Lightning 26, A3 Mastery Ltng 31, Merc Static Field 20 |
| 17 | Iron Wolf Ltng | 98 | 79 | 1519 | 1115 | 153 | 123 | 1827 | 42–48 | 157 | Holy Shock 25, A3 Merc Lightning 30, A3 Mastery Ltng 32, Merc Static Field 22 |
| 24 | Barbarian Mght | 50 | 28 | 684 | 400 | 142 | 90 | 590 | 28–32 | 94 | Might 7, Concentrate 8, Bash 10 |
| 24 | Barbarian Mght | 75 | 58 | 1287 | 1075 | 189 | 122 | 1345 | 44–48 | 138 | Might 14, Concentrate 15, Bash 19 |
| 24 | Barbarian Mght | 90 | 80 | 1872 | 1750 | 218 | 141 | 1970 | 55–59 | 158 | Might 18, Concentrate 17, Bash 21 |
| 24 | Barbarian Mght | 98 | 80 | 2232 | 2150 | 233 | 151 | 2330 | 60–64 | 166 | Might 18, Concentrate 17, Bash 21 |
| 25 | Barbarian Wwnd | 50 | 28 | 684 | 400 | 142 | 90 | 590 | 28–32 | 94 | Battle Orders 10, Battle Cry 10, MonWhirlwind 10 |
| 25 | Barbarian Wwnd | 75 | 58 | 1287 | 1075 | 189 | 122 | 1345 | 44–48 | 138 | Battle Orders 19, Battle Cry 19, MonWhirlwind 19 |
| 25 | Barbarian Wwnd | 90 | 80 | 1872 | 1750 | 218 | 141 | 1970 | 55–59 | 158 | Battle Orders 21, Battle Cry 21, MonWhirlwind 21 |
| 25 | Barbarian Wwnd | 98 | 80 | 2232 | 2150 | 233 | 151 | 2330 | 60–64 | 166 | Battle Orders 21, Battle Cry 21, MonWhirlwind 21 |
| 30 | Ascendant Dark | 50 | 21 | 504 | 284 | 92 | 75 | 358 | 8–10 | 85 | A4 AmpDmg 17, Bone Spear 11, Bone Armor 17, Teeth 13, A4 Mastery 12, CurMas 11 |
| 30 | Ascendant Dark | 75 | 54 | 934 | 588 | 124 | 100 | 910 | 19–21 | 129 | A4 AmpDmg 25, Bone Spear 17, Bone Armor 25, Teeth 20, A4 Mastery 23, CurMas 11 |
| 30 | Ascendant Dark | 90 | 79 | 1303 | 915 | 143 | 115 | 1546 | 33–35 | 148 | A4 AmpDmg 32, Bone Spear 21, Bone Armor 32, Teeth 26, A4 Mastery 31, CurMas 11 |
| 30 | Ascendant Dark | 98 | 79 | 1519 | 1115 | 153 | 123 | 1834 | 39–41 | 158 | A4 AmpDmg 32, Bone Spear 23, Bone Armor 32, Teeth 29, A4 Mastery 32, CurMas 11 |
| 31 | Ascendant Light | 50 | 21 | 504 | 284 | 92 | 75 | 358 | 8–10 | 85 | Sanctuary 15, Holy Bolt 5, Holy Light 10, Fist of the Heavens 8, A4 Mastery 12 |
| 31 | Ascendant Light | 75 | 54 | 934 | 588 | 124 | 100 | 910 | 19–21 | 129 | Sanctuary 27, Holy Bolt 9, Holy Light 17, Fist of the Heavens 15, A4 Mastery 23 |
| 31 | Ascendant Light | 90 | 79 | 1303 | 915 | 143 | 115 | 1546 | 33–35 | 148 | Sanctuary 30, Holy Bolt 15, Holy Light 23, Fist of the Heavens 22, A4 Mastery 31 |
| 31 | Ascendant Light | 98 | 79 | 1519 | 1115 | 153 | 123 | 1834 | 39–41 | 158 | Sanctuary 30, Holy Bolt 19, Holy Light 27, Fist of the Heavens 26, A4 Mastery 32 |
| 36 | Rogue Scout Phys | 50 | 36 | 594 | 489 | 94 | 139 | 742 | 11–13 | 90 | Slow Movement 3, Merc Magic Arrow 16, Strafe 11, Dodge 5, Evade 5 |
| 36 | Rogue Scout Phys | 75 | 67 | 1140 | 920 | 126 | 189 | 1438 | 22–24 | 131 | Slow Movement 5, Merc Magic Arrow 24, Strafe 18, Dodge 5, Evade 5 |
| 36 | Rogue Scout Phys | 90 | 67 | 1590 | 1250 | 144 | 219 | 1978 | 29–31 | 149 | Slow Movement 6, Merc Magic Arrow 29, Strafe 21, Dodge 5, Evade 5 |
| 36 | Rogue Scout Phys | 98 | 67 | 1830 | 1426 | 154 | 235 | 2266 | 33–35 | 159 | Slow Movement 6, Merc Magic Arrow 31, Strafe 23, Dodge 5, Evade 5 |

A Hell-hired merc below its row level uses negative d, e.g. at level 50: Rogue Fire Hell (row 67) life 390 / def 370 /
res 100; A2 Hell (row 75) life 430; A3 Fire Hell (row 79) life 223, def 0; A5 Might Hell (row 80) life 72, def 0.

## 2. Skills and auras

### 2a. Skill level and gear (READ, `auras.md` 0x6FD9FCB0)
Base level from §1b. The effective level (#10306) adds, for a non-player: `item_allskills` (127), element skills (126),
`item_singleskill` (107) and oskills (97, at most +3 when the merc owns the skill). Class-skill (83) and tab (188)
bonuses count only for players. `blvl` in calcs = the merc's base (Hireling) level.

### 2b. How often a merc uses each skill — D2Game 0x6FC3C4E0 (VERIFIED, 39,938 cases)
```
d = max(0, level − row.Level); total = DefaultChance
for slot k in order (stops at the first empty slot):
    skip if effective level ≤ 0; skip aitype-1 skills whose aurastate is on; Inferno also needs dist ≤ lvl/2 + 4
    total += Chance_k + tdiv(ChancePerLvl_k · d, 4);  cum_k = total
r = LCG % (total + 1)
r < DefaultChance → default action (Rogue: MonStats Skill1 RogueMissile; A2/A3/A5: melee attack; A4: idle 10 frames)
else the first slot with r ≤ cum_k → aura skills are set as the active skill (starts the aura), others are used in Mode_k
```
The first usable slot also gets r = DefaultChance, so a Chance-0 slot 0 is still picked 1 time in (total+1). Every
PD2 aura sits in slot 0 with Chance 0, so **auras start through this 1-in-101 pick**. Level 90, Hell rows (per decision):
| merc | default | skills |
|---|---|---|
| A1 Fire | 15/101 | Vigor aura 1, Merc Fire Arrow 60, Exploding Arrow 25 |
| A1 Ice | 15/101 | Meditation aura 1, Merc Cold Arrow 60, Freezing Arrow 25 |
| A1 Phys | 15/101 | Slow Movement 16, Merc Magic Arrow 45, Strafe 25 |
| A2 (all) | 20/101 | aura 1, Jab 80 |
| A3 Fire / Cold / Ltng | 0 | aura 1; Fire Ball 70 + Meteor 30 / Ice Blast 70 + Blizzard 30 / Lightning 75 + Static 25 |
| A4 Dark | 0 | Amp Damage 9, Bone Spear 40, Bone Armor 12, Teeth 40 |
| A4 Light | 0 | Sanctuary aura 1, Holy Bolt 65, Holy Light 10, Fist of the Heavens 25 |
| A5 Might | 20/101 | Might aura 1, Concentrate 40, Bash 40 |
| A5 Whirlwind | 20/101 | Battle Orders 6, Battle Cry 15, Whirlwind 60 |
Passives (Dodge, Evade, A3/A4 Mastery, CurMas) are never picked; they apply through passive states (#10418 → #10056).

**Decision rate** (D2Game 0x6FC3C8A0 / Hireable AI 0x6FC3DD40, READ): with a target within 25, each think (MonStats
aidel 1 = one frame after the previous action ends) first rolls `rand%100 < p`: A2/A5 p = 98; others
p = min(95, 40 + 2·level + counter), counter += 10 on a miss and resets on success. A miss idles 10 frames. Ranged
types (aip1 0) at distance < 4 try to step back half the time.

### 2c. Auras on the merc and the party (READ, `auras.md`)
- Aura level = the merc's effective skill level (+skills count) at the time the AI starts it (0x6FCC1D10 → #10306 with
  bonus). The aura timer then keeps that level until restarted.
- do-65 auras (Might, Prayer, Defiance, Blessed Aim, Thorns, Cleansing, Vigor, Meditation): the merc and every unit in
  range that passes `aurafilter` 0x12003 (players and monsters allied to the merc's owner — the party, other mercs,
  summons) get **aurastat1–6 only**. The merc never gets the passivestats (it has no mana, test mana > cost).
  Meditation uses filter 0x12001 (players only). Range = `aurarangecalc` ln12 = Param1 + Param2·(lvl−1), compared as
  squared distance in map sub-tiles (0x6FCC0C70); e.g. Might 20+3·(lvl−1), Defiance 20+2·(lvl−1).
- While the aura runs, the merc's own passive list (e.g. Prayer's hpregen blvl) is removed; when it is off the merc
  keeps it (#10418).
- do-66 auras (Holy Shock, Sanctuary): the merc gets the passivestats (its own lightning / magic damage); enemies get
  the aurastats; the party gets nothing.

### 2d. PD2's new types and skills (DATA)
Act 4 Ascendant (class 1056 `act4hire`, added to #11104 by PD hook 0x10268E40) with Dark (Amp Damage, Bone Spear,
Bone Armor, Teeth, A4 Mastery, Curse Mastery) and Light (Sanctuary, Holy Bolt, Holy Light, Fist of the Heavens,
A4 Mastery). Act 1 Phys rogue (Ids 36–38: Slow Movement, Merc Magic Arrow, Strafe); Fire/Ice rogues get Vigor /
Meditation, Exploding / Freezing Arrow. Act 3 gets auras (Cleansing/Prayer/Holy Shock), elemental masteries
(A3 Mastery: fire/cold/light mastery %, pierce, heal after kill) and merc versions of Fire Ball/Meteor, Ice
Blast/Blizzard, Lightning/Static Field. Act 5 Might (Might, Concentrate, Bash) and Whirlwind (Battle Orders, Battle
Cry, Whirlwind).

## 3. Items

### 3a. Allowed items — PD 0x102C0BE0 (VERIFIED)
Hireling `WType1/WType2` are not loaded by the game (no such column in the D2Common descriptor). PD2 NOPs the stock type
filter (D2Game 0x6FCF0655) and calls its own rule from the server (0x6FCF070E) and both client checks (0x6FB3E995,
0x6FB4AD10). All mercs: helm (incl. circlets, primal helms, pelts by type), armor, gloves, belt, boots. Then:
| class | weapons / off-hand |
|---|---|
| A1 Rogue 271 | bow or Amazon bow (left hand empty or bow quiver); **crossbow** (left empty or bolt quiver); **bow/bolt quivers** (matching launcher or empty right hand) |
| A2 338 | spear, polearm (includes javelins and PD2 scythes by type) |
| A3 359 | one-handed sword, wand, **orb**, mace, scepter, **auric shield**, any shield (no two-handed items) |
| A4 1056 | staff |
| A5 560/561 | sword, **axe**, **mace**, **hammer** (incl. throwing axes; one- or two-handed) |
Afterwards the normal requirement check (#10244) runs, and D2Common's class-item check is replaced (0x6FD7704D → PD
0x102C0E50, VERIFIED): Rogue may use Amazon items, A3 Sorceress and Paladin items, A5 Barbarian items; other class items
are refused (so A2 cannot use Amazon spears, A5 cannot use the paladin 2H Phase Blade). Unidentified or broken items
are refused (flags 0x10/0x100, READ).

### 3b. How item stats apply (READ)
Items are ordinary stat lists on the merc: resists, FHR, IAS, crit/DS/CB/OW, leech, +skills (§2a), damage, AR, defense.
Per-level stats use the merc level. Stats that do nothing or less on a merc: **vitality, energy** (ops 8/9 are player
only), **+class skills / skill tab**, oskills above +3 on owned skills, **cannot be frozen** (mercs are never frozen;
freezes become chills), slow is capped at 50, stun at 13 frames, mana-related stats (no mana pool),
**FHR on Act 5** (no GH mode in MonStats2) and **block**. `item_addexperience` (85) on merc gear raises the merc's own
experience (0x6FCFC030 reads it from the merc). Merc MF counts on the merc's kills together with the owner's MF (`mf_misc.md`).

## 4. Damage

### 4a. Melee (READ; same fill as players, `dmg_A_attack.md` §1–2)
```
base = weapon is two-handed (grip 2) ? stats 23/24 : stats 21/22        // Hireling Dmg lives in 23/24 only
pct  = skill% + s25 + trunc(str·StrBonus/100) + trunc(dex·DexBonus/100)   (no weapon: + str)
min  = ((base_min + s111)<<8) · (100 + s18 + pct)/100 ; max likewise with s17 ; then crit/DS, elemental, conversion
```
Grip 2 needs the item's 2-handed flag (the Barbarian "1-or-2-handed" rule is for players), so **with a one-handed
weapon (Act 3, Act 5 with a 1H or 1-or-2H sword/axe/mace) the Hireling base damage is not used at all**.
Undead/demon/%-vs-monster bonuses and target-defense bonuses are added when the attacker is a player or has alignment 2 (0x6FCFD4A9).

### 4b. Ranged (READ, `dmg_A_attack.md` §8, D2Common #10413)
Weapon missiles: 2H pair 23/24 for bows (grip 2), pct = str·StrBonus + dex·DexBonus + s25. A missile built without a
weapon item adds **+Dex%** for any merc (#11104). Leech from missiles is halved (PD 0x102700F0).

### 4c. Panel vs server (READ, D2Client 0x6FB3EA20 + server stat push 0x6FC68950)
The server sends the merc's base 21+23 as "21" and 22+24 as "22". The panel then shows
2H: T21 − W21 + T23 (off-weapon +min/+max from `dmg-min`/`dmg-max`, which set both 21 and 23);
1H: T21 (includes the Hireling base damage the server ignores); `s111` after the percent and from the weapon only
(server: before, from the total); mastery term 0; no crit/DS.

### 4d. Crit / DS / CB / OW (READ, `crit_cb.md`)
Mercs use the same PD crit/DS block (0x10270E00: item crit 258 + 337, cap 75, one roll, DS second), CB/OW nodes from
their items, CB divisor by defender (monsters 1/8; a merc *defender* takes CB as a player, 1/10).

### 4e. Leech — D2Game 0x6FCFBA40 (VERIFIED)
```
L64 = lifeLeech% << 6          // players: tdiv(L64, DifficultyLevels.LifeStealDivisor) — mercs skip this
gain = tdiv(MulDiv(drain, MulDiv(L64, phys, 100), 100), 64)   // drain = target MonStats Drain (skip at 100)
```
PD2 divisors are 1/2/3, so a merc in Hell leeches **3× what a player with the same % gets** (mana 2×/…). Missiles halve
first (PD). Merc leech rows 9–11 in the resist loop have no resist stat (no reduction).

## 5. Defense and resist
- **Resist (READ, PD 0x1026F680 monster branch):** `res = total − pierce` (negative halved for player-owned attackers),
  floor −100, **cap 90 because the owner is a player — no 75 + max-res step and no difficulty penalty**. The same cap
  applies to %DR (36) and magic resist, so a merc's physical DR% caps at 90, not 50. The merc panel instead shows
  `clamp(total + penalty(0/−40/−100), −100, min(75 + maxres, 90))` (`minions.md`).
- **Block (READ):** #10212 for monsters = min(stat 20, 75) behind a gate (0x6FD81680): MonStats NoShldBlock (blank for all
  mercs) or a shield graphic in the left hand; **Act 3 (class 359) is hard-coded to "no"**. So no merc ever blocks.
- **Hit recovery (READ):** get-hit thresholds as for monsters (`dmg_B_pipeline.md` §4.1); animation rate 50 + EFHR
  (EFHR = 120·v/(120+v), no other cap). Act 5 mercs have no GH mode (MonStats2 mGH blank): no hit recovery, FHR useless;
  A4/A5 have no KB mode. Stun ≤ 13 frames; never frozen; slow ≤ 50.
- Defense: base 31 + items + dex/4 through #10672 (`minions.md`); monster AR vs merc uses that defense.

## 6. Hire, revive
- Hire (VERIFIED): `max(Gold, tdiv(Gold·(100 + 15·(level − baseLevel)), 100))`; e.g. A2 Hell (15000, base 75) at 90 → 48,750.
- Revive (VERIFIED, #11097, used by D2Game 0x6FCE0DB0 at Kashya, Greiz, Asheara, Tyrael, Qual-Kehk):
  `cost = min(15 · trunc(L²/2), 50000)` → L50 18,750; L75 42,180; ≥ 82 50,000. No PD2 change.

## 7. Experience
```
Exp(L) = L²·(L+1)·ExpLvl (32-bit, row for the current level)        // #10448
per kill of the owner's side (D2Game 0x6FCFEA00, READ):
    e = expGain(monsterExp, mercLevel, monsterLevel, merc ExpRatio, merc stat 85)   // mf_misc.md §3, PD2 penalty
    e = min(e, (Exp(L+1) − Exp(L)) >> 6)                                            // 0x6FCFB6B0, per kill
    if the killer is not the merc: e = tdiv(e·86, 256)                               // 33.6 %
add (0x6FCFDCB0, VERIFIED): nothing if level ≥ playerLevel or level ≥ 98; exp += 2·e;
    level up while Exp(L+1) ≤ exp (stops at 98); stats/skills recomputed (0x6FC68BA0)
```

> **Correction (audit):** in PD2 the kill handler is **PD 0x102CA4B0** (patched in at D2Game 0x6FC97812), not stock 0x6FCFEA00; the merc part is VERIFIED there (`experience.md` §1.4, `harness/exp.c`). The rules are the same as written above, and the merc uses the **player** ExpRatio table at its own level.

Only the killer's owning player's current merc gains, with no range check. Net: 2× its adjusted share for its own
kills, 67 % for the owner's (and the owner's pets') kills. Party members' kills give nothing. Normal-hired mercs have the
lowest Exp/Lvl (e.g. A2 110 vs Hell 130), so they need 15–20 % less experience for identical stats.

## 8. Aura sharing
See §2c: range ln12 (20 + 2 or 3 per level), level = the merc's effective level incl. +skills, only the aurastats reach
the party; Meditation skips monsters; Holy Shock / Sanctuary give the party nothing.

## Left unverified
Kill-share wiring (cap, 86/256, which merc), the attack gate and AI movement, aura level/range application, the monster
resist branch for mercs, block gate, panel damage, the +skills bonus function, the merc-alignment test, and where PD2
realm servers might differ from the client install.
