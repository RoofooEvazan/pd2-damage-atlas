# Summons and mercenaries (PD2 = 1.13c + ProjectDiablo.dll)

Status key as in FINDINGS.md: **VERIFIED** = the game's own code was run natively on generated inputs and matched;
**READ** = read from the disassembly only; **OPEN** = not solved.
Files: `minions.js` (formulas), `minions.json` (tables, built by `extract_minions.py`),
`minions_harness/` (native checks: `python3 run_checks.py`, needs gcc -m32 and node).

## 0. Data and the calc language
- **Tables come from the compiled `.bin` files in PD2's data.zip** (what the game loads), not from the `.txt`:
  skills.bin (0x23C/record, offsets from the Skills.txt descriptor at D2Common 0x6FDB3645), skillscode.bin,
  monstats.bin (0x1A8/record), monlvl.bin (0x78/record), hireling.bin (0x118/record, descriptor at D2Common 0x6FDA8D2B).
  Numeric columns were cross-checked against the `.txt` of the same zip: 0 mismatches.
- **Skills.txt calcs are decompiled from the byte code** into fully parenthesised strings. The game's calc VM is
  Fog #10253 = 0x6FF69E90 (postfix, 64-slot stack):
  0/2/3 end · 1 fn(byte) · 4/5/6 keyword (byte/word/dword index into SkillCalc.txt order) · 7/8/9 const ·
  10 `<` 11 `>` 12 `<=` 13 `>=` 14 `==` 15 `!=` 16 `+` 17 `-` 18 `*` 19 `/` (x/0 = 0) 20 `^` 21 neg 22 `c?a:b`.
  Function table D2Common 0x6FDE9E24: 0 min, 1 max, 2 rand, 3 `skill(id, kw)` (0x6FDA1AF0), 5 `stat(id, mode)`
  (0x6FDA0C30: mode 0 = total #10973), 4/6 unused here. Keywords: D2Common 0x6FDA1070 (table 0x6FDA174C).
  - **VERIFIED:** 7,980 calcs (every calc column of the 187 skills in minions.json × 12 random keyword/skill/stat
    value sets) run through the real VM agree with `minions.js evalCalc` on the decompiled strings: 0 mismatches.
  - Consequence worth knowing: the compiler binds `?:` to the neighbouring operand. `1 + (blvl>19)?1:0 + stat('extra_grizzly')`
    compiles to `1 + ((blvl>19)?1:0) + stat(...)`, so Summon Grizzly gives 2 bears at 20 hard points (+ extra_grizzly).
  - Keywords used: `lvl` (the level passed in), `blvl` (hard points of that skill on the calc unit), `ulvl` (unit stat 12),
    `parN`, `lnXY` = parX + (lvl−1)·parY (0 if lvl ≤ 0, 0x6FD51670), `dmXY` (0x6FD9DC30), `edmn/edmx`
    ((EMin + tier) << HitShift, + EDmgSymPerCalc% when the value > 256 or EMinLev1 ≠ 0, then >> 8; 0x6FDA0460/0x6FDA0360),
    `edln`, `toht` (#10653: ToHitCalc, else ToHit + (lvl−1)·LevToHit).
    `skill('X'.kw)` evaluates kw on skill X at the calc unit's current level of X (hard + bonuses).

## 1. Summons — the pipeline
All player summons are cast through srvdofunc (table D2Game 0x6FD274A8, not patched by PD2).

| do-func | skills | handler |
|---|---|---|
| 16 | Valkyrie | 0x6FC6D710 |
| 31 | Raise Skeleton / Skeletal Mage / Skeleton Archer | 0x6FC718B0 |
| 56 / 57 | Clay, Blood, Fire Golem / Iron Golem | 0x6FC71750 / 0x6FC71540 (both → 0x6FC70070) |
| 58 | Revive | 0x6FC71340 |
| 49 | Shadow Warrior, Shadow Master, **Dopplezon (PD2 Decoy)** | 0x6FCB5980 |
| 114 / 115 / 119 | Raven / Plague Poppy, Vines, Cycle of Life / Oak Sage, HoW, SoB, Spirit Wolf, Fenris, Grizzly | 0x6FC668A0 / 0x6FC66720 / 0x6FC66560 |
| 44 / 45 | Blade Sentinel / all sentries (traps are monsters) | 0x6FCB6310 / 0x6FCB6280 (both → 0x6FCB5D50) |
| 144 | Hydra, Lesser Hydra | 0x6FC63D40 |

### 1a. Spawn (monster init D2Game 0x6FCCFDB0, READ; scaler VERIFIED in FINDINGS)
- The pet is spawned with the **game difficulty** (mercenaries are forced to 0, pets are not).
- Every summoned monster is `noRatio` except hydras and whatever Revive raises. noRatio ⇒ HP = rand[minHP, maxHP] of
  MonStats for that difficulty, defense = MonStats AC, raw (no MonLvl scaling).
- Resistances = MonStats Res{Dm,Ma,Fi,Li,Co,Po}[difficulty] (stats 36/37/39/41/43/45).
- Allied monsters (MonStats `Align` = 1) get no player-count HP/AR bonus (0x6FCCF840 sets players = 1).

### 1b. Level, MonLvl defense and AR — D2Game 0x6FC6E2A0 (VERIFIED, 20,000 cases, 0 mismatches)
`0x6FC6E2A0(eax=owner, ecx=L0, game, pet, lvl)`:
- L0 ≤ 0: **L = min(clvl, max(1, trunc(3·clvl/4) + lvl))**; L0 > 0: L = L0 (no clamp).
- stat 12 = L (Set). Then MonLvl row min(L, 110): **AC and TH are ADDED** (#10551) to stats 31 and 19
  (`L-AC`/`L-TH` columns when game+0x6A or game+0x74 is set, i.e. ladder/online type — OPEN which one PD2 realms use).
- Who calls it with what:
  - L0 = 0 (formula above): Valkyrie, skeletons, golems, sentries, Blade Sentinel.
  - L0 = max(1, calc2): Raven (`ulvl + par1 + lvl`, so above clvl), druid spirits/wolves/bear (`ulvl`).
  - Vines/Poppy/Cycle of Life: only stat 12 = max(1, calc2 = 3·ulvl/4 + lvl); **no MonLvl AC/TH**.
  - Shadows / Dopplezon: stat 12 = owner level; no MonLvl AC/TH. Hydras: not called.

### 1c. Generic stat setter — D2Game 0x6FC6F970 (VERIFIED, 6,000 cases × 16 skills, 0 mismatches)
PD2 redirects every call site (0x6FC610CA … 0x6FCB5F9D, code-init records) to **PD 0x102CA430**, which calls the stock
function through its lazy import (0x1027E060 → table 0x103D2D04[0] = rva 0x4F970) and then, for a player owner,
0x102E9380 stores the skill id/level in the owner's pet list (+0x18/+0x19). No stat change.
`0x6FC6F970(game, owner, pet, skillId, lvl, equipLevel)`, all calcs evaluated **with the owner as calc unit**:
1. `passivestat1–5` += calc (AddUnitStat #10551 on the pet; zero values too).
2. `aurastat1–6`: non-zero values go into one stat list on the pet (Set #10188), tagged with `aurastate` if any.
3. **calc1 = % life**: hp = maxhp + maxhp·calc1/100 (maxhp raw incl. step 1; > 0x100000 divides first), Set 7 and 6.
4. `sumskill1–5` get level = sumskNcalc when > 0 (#10302).
5. `sumumod`, `sumoverlay`; MonEquip gear (0x6FCB2ED0) at level `equipLevel`, or min(max(3·lvl,1), clvl) when 0.
- Summon resist **stat 349 `passive_summon_resist`** (0x6FC6E180, stock): only skeletons (do 31, 0x6FC71A5E) and golems
  (0x6FC70070): fire/light/cold += v unless the pet has absorb% of that element (Fire Golem's aura absorb ⇒ no fire),
  poison += v. No PD2 skill or property grants stat 349 in this data, so it is 0 in practice.

### 1d. Shadow Warrior / Shadow Master / Dopplezon — D2Game 0x6FCB1660 (VERIFIED, 6,000 cases, 0 mismatches)
- **lvl ≤ 1: returns immediately** (no bonus life, no stats at all).
- life += life·(lvl−1)·par1/100.
- Stock 1.13c, confirmed natively: **every aurastat gets the value of `aurastatcalc2`, every passivestat gets
  `passivecalc2`**, evaluated with the **pet** as calc unit. Consequences in PD2's data:
  - Shadow Warrior: tohit, skill_armor_percent, strength, dexterity are all `lvl·par3` = 12·lvl.
  - Shadow Master: tohit, strength, dexterity all `lvl·10`.
  - Dopplezon: its passive `tohit = edln` reads the empty passivecalc2 ⇒ **0**; its four resists are unaffected (same formula).
- Level = owner level, defense/AR = MonStats raw; MonEquip gear (random magic/rare by level) is not modelled.
- Where shadows get their assassin skill levels from: **OPEN** (not in this function or 0x6FCB5980).

### 1e. Revive — D2Game 0x6FC71340 (READ)
- HP = #11089 HP roll at the corpse's own level (MonStats × MonLvl HP, game difficulty); if clvl < mlvl then
  hp = hp·clvl/mlvl (≥ 1) and stat 12 = clvl. Defense stays what the monster spawned with.
- Then the generic setter (calc1 = par1 = 0 in PD2; aurastats damagepercent, masteries, `item_allskills = blvl`).
- PD2: the revive timer's first tick 15 → 8 frames (0x6FC70AD1), PD 0x10267CB0 at 0x6FC7152A (state 0x60 cosmetic).

### 1f. Attack-time damage and AR — D2Game 0x6FC97240 (READ) + physical roll 0x6FCFC530 (VERIFIED in damage.md)
- Each attack rebuilds stats 21/22/19 from the scaler #11089 at the pet's **current** level (flag A1 8, A2 0x10, S1/SC 0x20):
  noRatio ⇒ MonStats A1MinD/A1MaxD/A1TH raw. Revived monsters: scaled at their new level.
- `SkillDamage` pets (Raven, Spirit Wolf, Fenris, Grizzly, Dopplezon): 0x6FCBF2B0 → 0x6FCBE330 adds the owner's
  #10567/#10297 damage (MinDam/MaxDam + level tiers + DmgSymPerCalc, SrcDam·weapon/128) and #10653 ToHit, at the
  owner's level of that skill (#11008).
- No-weapon physical roll: min ≥ 1, max ≥ 2, + stat 111, × (100 + ED% + stat 25 **+ strength**) (−90 floor).
- AR used in the hit roll: stat 19 + 5·dex, × (100 + stat 119)/100 (FINDINGS, monster attacker). Defense #10672.

### 1g. Per skill (what minions.js does)
| skill | level | defense / AR base | life | stats (owner calc unless noted) |
|---|---|---|---|---|
| Valkyrie | 1b formula | MonStats AC + MonLvl | ×(1+calc1%) | aura str/dex/4×res; passive armor%, tohit (toht), ltng pierce; MonEquip gear at calc2 |
| Skeleton, Mage, Archer | 1b formula | MonStats AC + MonLvl | (MonStats + SM.lvl·SM.par1)×(1+calc1%) | aura dmg%/tohit/AC; passive normaldamage, dmg%; stat 349 |
| Clay/Blood/Fire/Iron Golem | 1b formula | MonStats AC + MonLvl | ×(1+calc1%) | GM tohit/speed, dmg% synergies, Iron AC; stat 349; Iron Golem item stats (0x6FCF60D0) not modelled |
| Raven | clvl + par1 + lvl | MonStats AC + MonLvl | MonStats | cold damage aura; SkillDamage Raven |
| Poppy / Vines / CoL | 3·clvl/4 + lvl | MonStats raw | ×(1+calc1%) | poison resist aura on the poppy (negative) |
| Spirits, Wolves, Fenris, Grizzly | clvl | MonStats AC + MonLvl | ×(1+calc1%) | resists min(ln78,80); armor%, tohit, dmg%; SkillDamage |
| Shadow W./M., Dopplezon | clvl | MonStats raw | ×(1+(lvl−1)·par1%) | §1d (calc2 for all), pet calc unit |
| Sentries, Blade Sentinel | 1b formula | MonStats AC + MonLvl | MonStats (100) | damage = their sumskills at `lvl` and synergies |
| Hydra, Lesser Hydra | spawn | – | – | damage = HydraFireball/HydraMissile at `lvl` |
| Revive | min(mlvl, clvl) | spawn defense | §1e | aura dmg%, masteries, allskills |
- **Count** = `petmax` with the owner as calc unit; PD2's `extra_*` item stats (extra_golem 476, extra_skele_war 461,
  extra_skele_mage 462, extra_skele_archer 475, extra_revives 444, extra_valk 464, extra_hydra 463, extra_spirits 459,
  extra_spiritwolf 193, extra_grizzly 509, extra_shadow 192, grims_extra_skele_mage 466, no_wolves 510) enter there.

### 1h. PD2 changes found in the summon code
- 0x102CA430 wrapper around the setter (bookkeeping only). ItemStatCost record size 0x144 → 0x150 in the setter's loops
  (0x6FC6FA0C, 0x6FC6FB39; PD2 has a longer ItemStatCost row).
- 0x6FC6EB65 NOP: the skeleton appearance helper (0x6FC6EB50) sends every class other than 363 to the mage branch,
  so Skeleton Archers use it. Cosmetic.
- Timers: Valkyrie 0x6FC6D879, pet spawn 0x6FCC37B7, revive 0x6FC70AD1. Hooks in the setter's op branch (0x6FC6FA54 →
  PD 0x102ED020) and sumskill start (0x6FC6FE8F → PD 0x102C8190) do not change values.
- PD2-only do-funcs (Desecrate 161, Uber summons 177/179) have no entry in the stock table; not traced (OPEN).
- Everything else (level rule, MonLvl add, setter, do-49 calc2 rule, SkillDamage, spawn) is stock 1.13c.

## 2. Mercenaries
### 2a. Which Hireling.txt row (D2Common #11156 = 0x6FDA32C0, VERIFIED inside the merc check)
- Key = (`Version` = 100 for expansion, `Id`). First row with that key, then later rows while their `Level` ≤ merc level.
- **Save / Armory type:** the save's merc block at 0xAF stores `word type` (0xB5); D2Game 0x6FD0D960 passes it to
  #11156 as the `Id` (adv/SAVE_FORMAT.md). The PD2 Armory `mercenary.type` is almost certainly that saved value, so
  **type = Hireling `Id`** (e.g. 14 = Desert Mercenary "Off-Hell", Blessed Aim; 13 = "Def-Hell", Defiance).
  This can't be proven from client code (the Armory is server-side); the local RoofooSinElev.d2s has 13 where the Armory
  sample has 14 (different snapshots).
- Level from experience (0x6FC75CF1 loop, READ): grow L while exp ≥ (L+1)²(L+2)·ExpLvl of the row for L (#10448).

### 2b. Base stats — D2Game 0x6FC68BA0 (VERIFIED, 20,000 random (row, level), 0 mismatches incl. row choice and skills)
With d = level − row.Level (trunc = toward zero):
| stat | value |
|---|---|
| strength / dexterity | max(10, Str + trunc(Str/Lvl·d/8)) |
| life (7, 6) | max(40, HP + HP/Lvl·d) |
| defense (31) | max(0, Defense + Def/Lvl·d) |
| AR (19) | max(0, AR + AR/Lvl·d) |
| damage (**23/24**, the two-handed pair) | max(0, Dmg-Min + trunc(Dmg/Lvl·d/8)), max(1, Dmg-Max + …) |
| fire/light/cold/poison resist | max(0, Resist + trunc(Resist/Lvl·d/4)) — overwrites the MonStats resists |
| life regen (74) | maxhp_raw / 2000 |
| skill k | if level ≥ Skills.txt reqlevel and Mode < 16: min(32, Level_k + ((LvlPerLvl_k·d) >> 5)), skipped if ≤ 0 |

### 2c. Items, auras, derived values (READ)
- Merc items are ordinary stat lists on the merc. Unit ops (0x6FD89530): per-level stats (op 2) use the **merc level**;
  `item_maxhp_percent` (op 11) is % of the total; **vitality/energy give nothing** (ops 8/9 need owner type 0 = player).
- Skill levels (0x6FD9FCB0, auras.md): base + allskills (127) + element (126) + oskill (97, capped at 3 when base > 0)…;
  class skill bonuses only for players. mercStats adds only `item_allskills` (T is a flat map without stat params).
- Own aura (do-func 65): the merc gets **aurastat1–6 only**. Passivestats need mana > mana cost (0x6FCBAA63, #10090),
  which a mercenary never has; the passivestate list is removed while the aura runs.
- Party auras from the player: aurastats only, evaluated with the caster (auras.md); pass them in T.
- **Merc panel (D2Client 0x6FB3EA20)** fields: level, exp, str, dex, life, damage, defense (#10672), 4 resists.
  - Damage: two-handed weapon (#10326) → base T21−W21+T23 / T22−W22+T24; one-handed or none → T21/T22 only
    (so the hireling base damage in 23/24 only shows with a two-handed weapon).
    % = T25 + StrBonus·str/100 + DexBonus·dex/100 + mastery (PD 0x102728B0 at 0x6FB3ECB3); no weapon: + str
    (**dex for class 271**, the Rogue). min = trunc(base·(100+T18+%)/100) + weapon s111, max with T17; + elemental;
    poison as min/max·length >> 8.
  - Resists: clamp(T + DifficultyLevels.ResistPenalty[current difficulty] (0/−40/−100), −100, min(75 + max-res, **90**)).
    PD2 changes the stock 95 cap to 90 (0x6FB3EFA7 and 0x6FB3EFAD, 0x5F → 0x5A). No server-side cap change was found.
- Server AR for the merc's attacks: T19 + 5·dex (+ skill ToHit), × (100 + T119)/100 (FINDINGS hit roll).

## 3. JS
```js
const M = require('./minions.js');            // browser: PD2Minions.setData(json)
M.summonStats('Clay Golem', {lvl: 20, blvl: 20, clvl: 90, levelsOf: n => ({'Golem Mastery': {lvl: 20, blvl: 20}})[n],
                             T: {extra_golem: 1}, difficulty: 2});
// -> {count: 6, level: 87, life: 3000, dmgMin: 2352, dmgMax: 3360, ar: 4476, def: 1533, res: {fire: 90, ...}, ...}
M.mercStats({type: 14, level: 90}, T_items, {difficulty: 2, weapon: {twoHanded: true, min1: 0, max1: 0, StrBonus: 100}});
```
- summonStats inputs: `lvl` (effective), `blvl`, `levelsOf(name)` for masteries/synergies, `clvl`, `T` (owner totals),
  `difficulty`, `ladder`, `mode` (A1/A2/S1), `revive: {monster, level}`, `ownerWeapon` (Dopplezon SrcDam).
  Output also has `lifeMin/lifeMax` (MonStats ranges), `elem`, `skills` (sumskills override MonStats skills), `stats`, `notes`.
- mercStats: `T` = merc item totals (each item's own ED already inside it) + party auras. Output `res` (server total),
  `resDisplay` (panel), `ar` + `arPct`, `base` (the VERIFIED 2b values), `skills` with levels.

## 4. Verification (minions_harness/run_checks.py)
| check | game code | cases | result |
|---|---|---|---|
| calc VM | Fog 0x6FF69E90 vs decompiled strings + minions.js | 7,980 | 0 mismatches |
| summon level / MonLvl add | D2Game 0x6FC6E2A0 | 20,000 | 0 mismatches |
| merc stats, row pick, skills | D2Game 0x6FC68BA0 + D2Common #11156 | 20,000 | 0 mismatches vs mercStats().base |
| do-func 49 setter | D2Game 0x6FCB1660 | 6,000 | 0 mismatches (calc2-for-all, lvl ≤ 1 exit) |
| generic setter | D2Game 0x6FC6F970 | 6,000 | 0 mismatches (passive add, aura ≠ 0, calc1 %, sumskills) |
Stubs: stat get/set/add, calc evaluator (returns a value derived from the calc offset, so each stat shows which column fed it),
list alloc, skill add, packets. Images: stock D2Game/D2Common (PD2's wrapper only adds bookkeeping, 1c).

## 5. Caveats / OPEN
- Assembly of the final numbers in summonStats (attack-time damage, SkillDamage add, AR, defense) is READ; the
  building blocks above are VERIFIED. Merc panel damage/resist display is READ (D2Client).
- Not modelled: MonEquip gear (Valkyrie, shadows, Dopplezon), Iron Golem item stats, shadow skill levels (OPEN),
  MonStats El1–3 elemental attacks, DifficultyLevels MonsterSkillBonus on MonStats skills, champion/unique revives.
- `L-` MonLvl columns (ladder/online) vs normal: which one PD2 realm games use is OPEN (option `ladder`).
- PD2 realm servers could run different server code; this covers the client install (as FINDINGS).
