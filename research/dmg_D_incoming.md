# Part D: damage dealt TO players, mercenaries and summons (monsters → you, PvP, map/zone modifiers)

PD2 = Diablo II 1.13c (`D2Game.dll` base 0x6FC20000, `D2Common.dll` 0x6FD50000) + `ProjectDiablo.dll` (PD, 0x10000000)
runtime hooks. Everything here comes from the game code and the PD2 `data.zip` tables (`/tmp/dz/data/global/excel`,
the `.bin` files are what the game loads). No community information was used.

Machine-readable flow: `adv/re/flow/flow_D.json` (schema `adv/re/flow/SCHEMA.md`). The generic apply pipeline
(resist / pierce / crit / CB / OW / absorb / DR / leech / thorns) belongs to part B; this file links to it with the
shared ids `B.*` and only adds what is specific to incoming damage.

**Status keys.** VERIFIED = the game's own code ran natively in the harness and matched the model on every case.
READ = read from the disassembly. DATA = a table value. PLAUSIBLE = follows from READ facts but a guard elsewhere was not ruled out.

**New native harnesses (this part)**

| harness | code run | cases | result |
|---|---|---|---|
| `harness/pvp.c` + `pvp.py` | PD 0x1026D1D0 (damage-percent / PvP scaler), with PD's own 0x102CE840, 0x102CEAE0, 0x102CEB10 and D2Game's type table 0x6FD22AB0 | 6,588 (every skill id 0–619 × 3 difficulties × levels 1/157/166, 14 pet classes, mercs, prime evils, bosses, safe zone) | table in §4 (the output *is* the code's) |
| `harness/monatk.c` + `monatk.py` | stock D2Game 0x6FC97240 (monster attack-time AR / damage / elemental set-up) with the real D2Common scaler #11089 0x6FDA4A00 and the real PD2 `monstats.bin` / `monlvl.bin` | 30,000 random (1,240 monster rows, all modes, levels 0–120, 1–12 players, ladder/L-flag, classic/expansion, RNG seeds) | 0 mismatches against the model in §2.3 |

Stubs: pvp.c stubs only D2Common #11104/#10064/#10278/#10511, PD's owner lookup 0x102CB180 and PD's safe-zone test
0x102CF760. monatk.c stubs #10195 (stat list), #10343, #10830 (alignment), #10973 (GetStat: level, players) and
#10188 (SetStat recorder).

---------------------------------------------------------------------------------------------------------------------

## 1. The incoming pipeline, in order

A monster (or player, pet, merc) attack that reaches a player goes through these steps. D.* ids are the flow nodes.

1. **Monster set-up** (spawn, once): level (`D.mon_level`), map/zone mods (`D.map_mods_monster`), MonUMod bonuses
   (`D.umods`), skill levels (`D.mon_skill_level`).
2. **Attack time** (every attack): D2Game 0x6FC97240 sets stats 21/22 (phys min/max), 19 (AR) and the elemental
   stats from MonStats A1/A2/S1 and El1–3 (`D.mon_attack_stats`, VERIFIED).
3. **Hit roll** PD 0x10270EB0 → 0x10271740 (VERIFIED in `hit_pd2.md`), then block / weapon block / evade / dodge /
   avoid PD 0x1026FB60 (VERIFIED). No auto-hit on running players in PD2 (`D.hit_roll_vs_player`, `D.block_avoid`).
4. **Damage roll** D2Game 0x6FCFD450 (`D.mon_damage_roll`): physical 0x6FCFC530 (+stat 25 damage%), PD crit /
   deadly strike (`B.crit`, `B.deadly`), elemental rolls with masteries, drains (PD ×16 for monster mana drain),
   poison, conversion, then **MonStats `Crit`** doubling (`D.monstats_crit`).
5. **Resist pass** D2Game 0x6FCFC0B0: flat DR/MDR values read → **damage percent** PD 0x1026D1D0 (`D.damage_percent`:
   PvP, pets, mercs, prime evils; 100% for a plain monster) → **event 11 `absorbdamage`**: Energy Shield, Bone
   Armor, Cyclone Armor (`D.absorb_skills`, **before** resistances) → cold/freeze-length immunities → per-type step
   PD 0x1026F410 (`B.resist`, `B.pierce`, `B.dr`, `B.mdr`, `B.absorb`).
6. **Apply** D2Game 0x6FCFE0C0: events `damagedinmelee` / `damagedbymissile` (damage-to-mana, `D.damage_to_mana`),
   physical capped at the defender's current life, leech / monster drain (`B.leech`, `D.monster_drain`), absorb heal,
   `B.life_subtract`, mana/stamina drain subtraction, stun (PD 0x1026F190), chill (PD FCR change), freeze, poison
   (PD 0x10271A70), burning (`B.hit_effects`).
7. **After the hit**: MonUMod "on hit" events (Cursed → PD2 MonAmplifyDamage), item events of the attacker
   (CB/OW on players via `map_mon_crushingblow`/`openwounds`, splash), thorns etc. (`B.thorns`, `B.cb`, `B.ow`).

---------------------------------------------------------------------------------------------------------------------

## 2. Monster damage build-up

### 2.1 Level (READ; details in `defense.md` §2.1, `hit_pd2.md` §4)
- Normal difficulty: MonStats `Level`. NM/Hell: Levels.txt `MonLvl{2,3}Ex` of the area for monsters that are neither
  `noRatio` nor `boss`. PD2 replaces the lookup (`0x6FCCFF0B → 0x102EEE40 → 0x10268D00`):
  - map levels **137–201** (PD 0x102CE890): `+ map_glob_arealevel` (game key 1);
  - the rotating Hell "desecrated/terror" zone set (`game+0x26F2`, redrawn every 900 s): **level 85**. No other damage
    change was found for those zones (the only other readers of `+0x26F2` are drop/logging code).
- MonUMods applied right after: uniques / superuniques / their minions **+3** (leveladd, umod 4), champions **+2**
  (+3 then −1 in the champion handler). AR and damage use the level at attack time (with the bonus); HP/defense use
  the spawn level.

### 2.2 Player count (READ + DATA)
- Damage and AR: `x += tdiv(x·f,128)`, f = [0,0,8,16,24,32,40,48,56][p] (8p−16 for p ≥ 9), **NM/Hell only,
  expansion only**. p = the monster's **stat 100**, written once at spawn (living players in the game, or `/players`
  in single player). Joining/leaving later does not change a monster that already exists. Monsters with the
  alignment state (#10830 ≠ 0, e.g. converted) get no factor. This table is **not** patched by PD2.
- HP (for completeness): PD2 patches the 2–8 player entries of D2Game 0x6FD1B614 (records 0x103D0C68–0x103D0D30) to
  **+70 % per extra player** (70…490). p ≥ 9 keeps the stock `(p−2)·50`, so 9 would be lower than 8 (unreachable
  online). `defense.js playersHP` still had the stock 50/step table: fixed (§12).

### 2.3 Attack-time stats: D2Game 0x6FC97240 (VERIFIED, 30,000 cases)
`stdcall(unit, mode)`, called from the monster attack code (0x6FC95E10 and the AI) before each attack. Not patched by
PD2.
```
cols  = mode 5 (A2): A2TH/A2MinD/A2MaxD; modes 7–8 (SC/S1): S1*; anything else: A1*
L     = (game+0x6A != 0) || ladder(game+0x74)          -> "L-" MonLvl columns (single player has +0x6A = 3 -> L-)
TH    = scale(TH, MonLvl TH), min = scale(MinD, MonLvl DM), max = scale(MaxD, MonLvl DM)  (#11089, muldiv/100,
        level clamped to the last row, noRatio = raw)   + MonStats SkillDamage extras for summons (0x6FCBF2B0, part C)
classic (game+0x70 == 0), NM/Hell, Align != 1:  min = tdiv(10·min,12), max = tdiv(10·max,12), TH = tdiv(10·TH,15);
        and no player factor
expansion: f = player factor (stat 100); min/max/TH += (x·f)>>7 (rounded toward zero)
SetStat 21 = min, 22 = max, 19 = TH (base stat list)
for slot i = 1..3: if ElMode(i) == mode and ElPct(i)[d] > 0 and (ElPct >= 100 or rand(100) < ElPct):
     emin/emax/edur = #11089 with flag 0x40+i
     expansion: all three get the player factor
     ElType 10 ('rand'): type = rand(5)+1 (fire, ltng, mag, cold, pois); edur = 25 if 0
     type 1 fire -> 48/49; 2 ltng -> 50/51; 3 mag -> 52/53; 4 cold -> 54/55 + 56 length;
     5 pois -> 57 = 10·min, 58 = 10·max, 59 = 2·dur; 6 life drain -> 60/61; 7 mana drain -> 62/63;
     8 stamina drain -> 64/65; 9 stun -> 66 = dur; 11 burning -> 316/317, 315 = dur
```
**noRatio rows (VERIFIED).** For `noRatio` rows the scaler writes the raw El values to other
output slots, so min/max read back as 0 (only the duration survives). Of the 35 noRatio rows only the Fire Golem has
an El type; its fire does not come from El1.

### 2.4 MonStats `Crit` (READ)
D2Game 0x6FC970E0, called at 0x6FCFDA6C after all rolls: `rand(100) < Crit` → **every damage field ×2** (phys,
fire, light, magic, cold, poison). Crit is 5 for almost every row; blank (0) for the uber Ancients. Not patched.
Independent of the PD2 crit/deadly-strike roll (stats 337/258/141, used by monsters only via map mods, `B.crit`).

### 2.5 Monster skill levels (READ + DATA)
Spawn (0x6FCD01B0–0x6FCD020D): each of Skill1–8 with `SkNlvl > 0` gets level `SkNlvl + MonsterSkillBonus`
(DifficultyLevels 0 / 3 / 7), then 0x6FC2AD90 stores the bonus. Skill/missile damage from those levels is part C.

### 2.6 Monster missiles and spells (pointer)
A missile uses the owner's rolled damage × Missiles.txt `SrcDamage`/128 (the damage roll's `src` byte, 0x80 = 100 %)
plus the missile's own damage; skills use Skills.txt at the level from §2.5. Formulas: part C. Only 108 of the 1,057
missiles roll to hit (`hit_pd2.md`).

---------------------------------------------------------------------------------------------------------------------

## 3. Champions, uniques and minions: MonUMod

### 3.1 Spawn handlers (table D2Game 0x6FD2E550, one dword per umod id; READ)
CDB = DifficultyLevels ChampionDamageBonus (90/75/66). `c[k]` = MonUMod `constants` (PD2 = stock values).
DM = MonLvl DM (or L-DM) at the monster's current level; "unique" = the handler's arg (uniques/superuniques),
otherwise the minion constants.

| umod | handler | effect on damage the player takes |
|---|---|---|
| 4 leveladd | 0x6FC41E80 | +3 levels (AR/damage at attack time) |
| 16 champion | 0x6FC42DA0 | −1 level; stat 25 += CDB (Hell +66 % damage), stat 119 += tdiv(75·CDB,100) (Hell +49 % AR); damage halved for base class 118 |
| 5 strong | 0x6FC42BC0 | unique: +150 %·CDB damage, +100 %·CDB AR; minion: +75 %·CDB / +50 %·CDB (Hell: +99 %/+66 %, +49 %/+33 %) |
| 39 berserk | 0x6FC42D00 | life −75 %; stat 25 and stat 119 += tdiv(300·CDB,100) (Hell +198 %; damage halved for class 118) + champion handler |
| 37 fanatic | 0x6FC43810 | defense −70 % (stat 16) + champion handler (speed from the AI) |
| 38 possessed | 0x6FC437C0 | life +100 %, flag 0x20 + champion handler |
| 36 ghostly | 0x6FC43850 | stat 36 = 80 (80 % physical DR), flag 0x40, champion handler, **cold damage** min = tdiv(c[22+d]·DM,100) (33 %), max = tdiv(c[25+d]·DM,100) (50 %), cold length 150 |
| 9 fire enchanted | 0x6FC42740 | fire 48/49 += unique: 66 %·DM – 100 %·DM; minion: c[16+d] / c[19+d] = 0/33/33 % – 0/50/50 %; +fire resist (0x6FC41BE0) |
| 17 lightning enchanted | 0x6FC42600 | lightning 50/51, same numbers |
| 18 cold enchanted | 0x6FC42490 | cold 54/55, same numbers, cold length 56 += 100 + 5·x |
| 25 mana burn | 0x6FC421D0 | mana drain 62/63 = same numbers **<< 1** in PD2 (stock << 8; byte patches 0x6FC422DF, 0x6FC422F4 = 1) |
| 23 poisonhit | 0x6FC42320 | poison 57/58 = same numbers (per-frame units), length 59 += 2·(5x+150) |
| 28 stoneskin, 8 resist, 27 spectral hit | 0x6FC41BE0 | resist / defense handler (stoneskin: defense ×2) |
| 30 aura enchanted | 0x6FC44CD0 → PD 0x102C62B0 | see §3.3 |

Normal-difficulty minions of an enchanted unique get min 0 → **no elemental bonus at all** (0x6FCFCD80 skips a type
whose min stat is < 1 even if the max is 50 % DM) (READ).

### 3.2 Event handlers (D2Game 0x6FD2F448, 6 events × 43 umods; dispatcher 0x6FC468E0, READ)
Event 0 = attack, 1 = spawn/init, 2 = death, 3 = the monster hit something, 4 = the monster was hit, 5 = missile.
**PD2 replaces 11 dispatcher calls** (0x6FC41825/35/95, 0x6FC469C1/D5/E5, 0x6FC95F2B/FC0, 0x6FCFD13A/D179,
0x6FCFE233) with PD 0x10253DE0: 10 umod slots instead of 9, and **event 4 ("was hit") runs at most once per frame**
per monster (state 86 set for one frame). Lightning Enchanted also keeps its stock 10-frame cooldown.

| umod | event | handler | incoming damage |
|---|---|---|---|
| 9 fire enchanted | death | 0x6FC466B0 | missile 117 `monstercorpseexplode` + area damage 0x6FCFE380, radius difficulty+4. Base = the row's **scaled MaxHP** (#11089 flag 1, not the unit's real life) × MonsterCEDamagePercent (50/35/20) / 100, then ×¾ (N) / ×⅔ (NM) / ÷8 (Hell), then 60–100 % roll; stored `<<6`, i.e. **¼ of that value as physical and ¼ as fire** (READ, low confidence on the output slot) |
| 18 cold enchanted | death | 0x6FC462D0 | casts skill 194 `DiabCold` at level max(1, mlvl/2) (Skills EMin 15 +9/lvl …, part C) |
| 17 lightning enchanted | was hit, death | 0x6FC468C0 → 0x6FC46320 | fires missile 0x21 at level max(1, mlvl/2), ≥ 10 frames between procs (MonsterData+0x18), + PD once-per-frame guard |
| 7 cursed | hit something | 0x6FC44250 | 3 in 4 hits: stock would cast Amplify Damage (level mlvl/5+1); **PD2 replaces the callback (0x6FC442EB → PD 0x102C0EB0) with skill 424 `MonAmplifyDamage`: −50 % physical DR (stat 36 −Param5 = −50) for 300 frames**, through PD's curse routine (curse resistance / length reduction, `uber_review.md`) |
| 27 spectral hit, 23 poisonhit, 14 spcdamage, 19 hireable | attack / missile | 0x6FC42B60, 0x6FC45CC0, 0x6FC418A0, 0x6FC41B30 | spectral hit's random element roll lives here (not decoded, OPEN) |
| 31 goboom, 32 firespike, 33 suicide minion, 42 lightningdeath, 20 scarab, 10 poisondead | death | various | monster-specific death attacks (part C for skill damage) |

### 3.3 Aura enchanted (PD2 rewrite, READ)
Stock picked from a level-gated table (Might mlvl/6, Holy Fire /6, Blessed Aim /5, Holy Freeze /7, Conviction /8,
Fanaticism /8, Holy Shock /8 from mlvl 20). PD2 replaces the "set skill + start aura" code (0x6FC44E4E → PD
0x102EF9D0 → 0x102C62B0, 18 NOPs; 0x6FC44E05 → 0x102C8190):
- skill = random pick from {472 Mon Conviction, 473 Mon Fanaticism, 474 Mon Holy Shock, 416 MonHolyFreeze,
  476 Mon Holy Fire, 477 Mon Might, 478 Mon Concentration, 479 Mon Vigor} (built at 0x10126C80); superunique 37
  gets skill 473; Uber Mephisto (704) Conviction 20 as stock.
- **level = floor(mlvl/7)** (1 when below 2), **max 13**. mlvl 88 → 12.
- Relevant aurastats (DATA): Mon Conviction −min(27+4(L−1),150) fire/cold/light resist and −(20…) % defense
  (at L12: **−71 % resists**); Mon Might damage% = edmn (40 +10/lvl → L12 150 %); Mon Fanaticism attack rate /
  AR% / damage% (ln56/2); Mon Concentration damage% ln34 (30 +5/lvl); MonHolyFreeze −speed.

---------------------------------------------------------------------------------------------------------------------

## 4. Damage percent: PvP, pets, mercs, prime evils (PD 0x1026D1D0, VERIFIED)

Stock 0x6FCFAC10 returned one percent (PvP 17 %, merc vs merc 25 %, merc vs boss HireableBossDamagePercent, prime
evil vs player-owned units 400 % / 200 % vs mercs). PD2 hooks the call (0x6FCFC180 → 0x102ED040 → 0x1026D1D0), applies
per-field percentages itself and returns 100. Fields with the table flag (0x6FD22AB0 row+0x20): physical, fire,
lightning, cold, magic, poison rate and the three leech/drain fields. Lengths are never scaled. Each field:
`d = trunc(d · pct / 100)` in double math (0x102CE840). A field-specific value replaces the general one when non-zero.

**Defender is a player.** Attacker = owner via 0x102CB180 (a pet or merc is replaced by its owning player; a
player is itself).
- Plain monster (no owner): **100 %**. A monster owned by another monster: also 100 % (the 50 % branch needs the
  attacker to be a hireling or have unit flag +0xC4 bit 31; practically unused). *Corrects `defense.md`.*
- Player (or a player's pet/merc) → player: first the **safe-zone test** 0x102CF760: level 157 rectangle
  x 0x50B0–0x50DA, y 0x281C–0x2852; level 159 x 0x4FF9–0x5025, y 0x2D79–0x2D98 (either unit inside), playerdata+0x190
  set, or game type 0x3D within 12,000 frames of +0x2628 → **all damage fields zeroed** (0x102CF660) and 0 returned.
- Otherwise **P = 16 % (Normal) / 12 % (Nightmare) / 6 % (Hell)**, −2 in level 166 (14/10/4). This applies to any
  hostile player damage, not only in the PvP levels.
- Skill id used: a pending id stored in the defender's playerdata+0x194 by PD missile/aura hooks (0x1026E9CF,
  0x1026EA1F, 0x1026FB22, 0x102710E6; Holy Fire aura = 102), else the damaging missile's skill (dmg+0x5C), else the
  **owner's currently selected skill** (#10511). Skill 337 is read as 383.
- Pet classes (the pet itself, [esp+0x24]): 351–353 Hydra → skill 62; 738–740 Lesser Hydra → skill 383;
  932 dopplezonnew: physical ×0.3; 425 plague poppy: poison ×0.15; 416 death sentry: ×0.8 (Hell ×1);
  357 Valkyrie: ×0.7; 417 Shadow Warrior ×0.1, 418 Shadow Master ×0.33 (both then **also** get the owner's skill
  factor); every other pet or merc: P × the owner's current skill factor.

Per-skill factors (multiply P; "all" = every scaled field incl. leech; "—" = none). N and NM are always equal.

| id | skill | Normal / Nightmare | Hell |
|---|---|---|---|
| 6 | Magic Arrow | magic ×0.6 | magic ×1.3 |
| 7 | Fire Arrow | fire ×1.5 | fire ×1.25 |
| 10 | Jab | phys ×0.4 | phys ×0.8 |
| 11 | Cold Arrow | cold ×1.5 | cold ×1.25 |
| 12 | Multiple Shot | phys ×0.85 | — |
| 14 | Power Strike | phys ×0.17 | — |
| 15 | Poison Javelin | phys ×0.75, pois ×0.85 | pois ×0.8 |
| 20 | Lightning Bolt | light ×0.55 | — |
| 21 | Ice Arrow | cold ×2 | cold ×1.5 |
| 22 | Guided Arrow | all ×0.9 | all ×1.2 |
| 24 | Charged Strike | light ×0.85 | — |
| 25 | Plague Javelin | phys ×0.75, pois ×0.9 | pois ×0.55 |
| 26 | Strafe | phys ×0.35 | — |
| 27 | Immolation Arrow | fire ×2 | fire ×1.55 |
| 31 | Freezing Arrow | cold ×2 | cold ×1.6 |
| 32 | Valkyrie | all ×0.7 | all ×0.7 |
| 35 | Lightning Fury | phys ×0.75 | — |
| 36 | Fire Bolt | fire ×0.75 | fire ×1.35 |
| 38 | Charged Bolt | — | light ×0.75 |
| 39 | Ice Bolt | cold ×0.9 | cold ×1.45 |
| 41 | Inferno | fire ×1.25 | — |
| 43 | Telekinesis | light ×0.1 | light ×0.1 |
| 44 | Frost Nova | cold ×1.5 | cold ×1.4 |
| 45 | Ice Blast | cold ×1.5 | cold ×1.45 |
| 46 | Blaze | fire ×0.5 | — |
| 47 | Fire Ball | fire ×0.9 | fire ×1.3 |
| 48 | Nova | light ×1.75 | light ×1.5 |
| 49 | Lightning | light ×0.9 | light ×2 |
| 51 | Fire Wall | fire ×0.6 | fire ×1.2 |
| 53 | Chain Lightning | light ×1.2 | — |
| 55 | Glacial Spike | cold ×1.5 | cold ×1.45 |
| 56 | Meteor | all ×0.65 | all ×0.9 |
| 57 | Thunder Storm | light ×0.3 | light ×0.35 |
| 59 | Blizzard | — | cold ×1.3 |
| 62 | Hydra | fire ×0.3 | fire ×0.8 |
| 64 | Frozen Orb | cold ×1.5 | cold ×1.3 |
| 67 | Teeth | magic ×0.2 | magic ×0.3 |
| 73 | Poison Dagger | pois ×1.5 | pois ×1.25 |
| 74 | Corpse Explosion | all ×0.65 | — |
| 83 | Desecrate | pois ×0.85 | pois ×0.6 |
| 84 | Bone Spear | magic ×0.35 | magic ×0.4 |
| 92 | Poison Nova | pois ×1.5 | pois ×1.25 |
| 93 | Bone Spirit | magic ×0.3 | magic ×0.15 |
| 97 | Smite | phys ×0.5 | phys ×0.75 |
| 101 | Holy Bolt | magic ×0.5 | magic ×0.65 |
| 102 | Holy Fire | fire ×1.25 | fire ×0.5 |
| 106 | Zeal | phys ×0.75 | phys ×0.85 |
| 107 | Charge | phys ×0.7 | phys ×0.7 |
| 111 | Vengeance | fire ×2, light ×2, cold ×2 | — |
| 112 | Blessed Hammer | magic ×0.6 | magic ×0.4 |
| 121 | Fist of the Heavens | all ×0.55 | all ×0.65 |
| 133 | Double Swing | all ×0.9 | — |
| 139 | Stun | all ×0.9 | — |
| 140 | Double Throw | — | phys ×1.15 |
| 143 | Leap Attack | phys ×0.16 | phys ×0.12 |
| 144 | Concentrate | phys ×0.9, magic ×0.45 | magic ×0.5 |
| 147 | Frenzy | all ×0.9 | phys ×1.25 |
| 151 | Whirlwind | phys ×0.55 | phys ×0.88 |
| 152 | Berserk | phys ×0.9 | — |
| 154 | War Cry | phys ×1.2 | phys ×1.05 |
| 221 | Raven | all ×0.6 | all ×0.6 |
| 222 | Plague Poppy | pois ×0.15 | pois ×0.15 |
| 225 | Firestorm | fire ×1.5 | fire ×1.2 |
| 229 | Molten Boulder | all ×0.9 | all ×0.8 |
| 230 | Arctic Blast | cold ×1.5 | — |
| 234 | Eruption | fire ×0.9 | — |
| 238 | Rabies | pois ×1.25 | pois ×1.25 |
| 239 | Fire Claws | fire ×3 | — |
| 240 | Twister | phys ×0.8 | phys ×0.8 |
| 243 | Shock Wave | phys ×1.2 | — |
| 244 | Volcano | all ×0.45 | all ×0.45 |
| 245 | Tornado | phys ×0.5 | phys ×0.35 |
| 249 | Armageddon | phys ×0.3, fire ×0.3 | phys ×0.35, fire ×0.35 |
| 250 | Hurricane | cold ×1.5 | — |
| 253 | Psychic Hammer | magic ×0.2 | magic ×0.1 |
| 254 | Tiger Strike | all ×0.9 | — |
| 255 | Dragon Talon | all ×0.7 | — |
| 257 | Blade Sentinel | phys ×0.2 | phys ×0.25 |
| 260 | Dragon Claw | all ×0.9 | — |
| 266 | Blade Fury | phys ×0.9 | phys ×0.75 |
| 268 | Shadow Warrior | all ×0.1 | all ×0.1 |
| 273 | Mind Blast | phys ×0.3 | phys ×0.35 |
| 276 | Death Sentry | all ×0.8 | — |
| 279 | Shadow Master | all ×0.33 | all ×0.33 |
| 281 | Wake Of Destruction Sentry | fire ×0.4 | fire ×0.8 |
| 311 | mon inferno sentry | fire ×0.6 | — |
| 313 | Sentry Chain Lightning | light ×0.45 | light ×0.4 |
| 337 | HydraMissile (read as 383) | fire ×0.4 | fire ×0.8 |
| 364 | Holy Nova | magic ×0.5 | magic ×0.5 |
| 368 | Deep Wounds | phys ×0.33 | phys ×0.33 |
| 369 | Ice Barrage | cold ×1.6 | cold ×1.4 |
| 371 | Holy Light | magic ×0.05 | magic ×0.15 |
| 376 | Combustion | fire ×0.35 | fire ×0.9 |
| 378 | Joust | all ×0.2 | all ×0.2 |
| 381 | Dark Pact | magic ×0.5 | magic ×0.5 |
| 383 | Lesser Hydra | fire ×0.4 | fire ×0.8 |
| 392 | Sentry Lightning | light ×0.37 | light ×0.4 |
| 488 | DoppleZonStrafe | phys ×0.3 | phys ×0.3 |

Example: Hell Blizzard → cold ×(6 % × 1.3) = 7.8 %; Hell Fist of the Heavens → every field 3.9 %.

**Defender is a monster (merc, summon, boss).**
- Merc attacker: vs a merc **25 %**; vs a MonStats `boss` **HireableBossDamagePercent 50 / 35 / 25 %**; else 100 %.
- MonStats `primeevil` attacker vs a player-owned unit (+0xC4 bit 31): **300 %** vs summons, **100 % vs mercs**
  (stock 400 / 200).
- Any non-merc attacker in a PvP level (157/159/166) vs a player-owned **non-merc** unit (summons): **85 %**
  (safe-zone → 0). Enemy mercs take 100 %.

---------------------------------------------------------------------------------------------------------------------

## 5. Map and zone modifiers

### 5.1 Routing (DATA + READ)
The map item's stats are copied into game lists by the map-open handler (0x102D4E70). ItemStatCost `Divide` is the
target: **< 1000** = monster stat id, **1000–1999** = player stat (id − 1000), **2000+** = game key (id − 2000),
**3000** = physical-as-extra-element (handled by stat id). Monster list: game+0x1DF8; player list: game+0x21FA; keys:
game+0x2630 {u16 value, u16 key, i32 level}.

| map stat | → | effect on incoming damage |
|---|---|---|
| map_mon_ed% (426) | monster stat 25 | +% physical damage (same stat as champion/berserk, added) |
| map_mon_att / tohit (277/394) | 119 | +% AR |
| map_mon_{fire,light,magic,cold,poison}{min,max}dam, coldlength, poisonlength (376–387) | 48–59 | flat elemental on **every** attack roll (only if the min stat ≥ 1) |
| map_mon_passive_{fire,ltng,cold,pois}_mastery (388–391) | 329–332 | +% to that element in the roll (0x6FCFCD80 / 0x6FCFBED0) |
| map_mon_passive_*_pierce (414–417) | 333–336 | −player resist (`B.pierce`; no halving for monster attackers) |
| map_mon_deadlystrike (449) | 141 | PD crit/DS roll (`B.deadly`, cap 75 + stat 210) |
| map_mon_crushingblow (408), openwounds (407) | 136, 135 | CB / OW **on players** (own item events 16/15; CB divisor for players 1/10, `B.cb`, `B.ow`) |
| map_mon_splash (427) | 359 item_splashonhit | melee splash (event 20) |
| map_mon_lifedrainmindam (403) | 60 | monster "life drain": a flat amount per hit (§6.4), heals the monster |
| map_mon_fasterattackrate / fastercastrate / velocity (392/393/401) | 93 / 105 / 67 | speed |
| map_mon_pierce (406) | 156 | missile pierce |
| map_mon_cannotbefrozen (450) | 153 | |
| map_mon_absorb*_percent, normal_damage_reduction, hpregen, maxhp_percent, ac% | 142–148, 34, 74, 76, 16 | monster defense (parts A/B) |
| map_play_{fire,light,cold,poison}resist (428–431) | player 39/41/43/45 | −resist for players in map levels |
| map_play_max*resist (418–421) | player 40/42/44/46 | −max resist |
| map_play_ac% (410), damageresist (456), toblock (412), maxhp% / maxmana% (454/455), hpregen (413), FHR (411), speeds | 16, 36, 20, 76/77, 74, 99, … | player defenses |
| map_glob_arealevel (374) | key 1 | +monster level (137–201) |
| map_glob_density / monsterrarity / skirmish (372/375/493) | keys 0 / 2 / 4 | see 5.3 |

### 5.2 The applier PD 0x102DBA50 (READ)
Called after every monster spawn (0x6FD01D90 hooks → 0x102C85A0 → 0x102DBFE0; also 0x102C7F10 from 0x6FC3FF58) and
for players on entering a map level (0x102DC050 with the player list). Monsters get the list only when the level is a
map level and MonStats `Align` (+0x4C) = 0; the stats go into a new stat list tied to state 198 (`map`), which also
blocks a second application.
- Generic: add the stat (value 0 → 1).
- Stat 76 (maxhp%) on monsters: applied directly to stat 7.
- **PvP levels (157/159/166), every unit:** stat 443 extra_bonespears −4, 481 extra_holybolts −4, 491 pvp_disable 1,
  482 pvp_cd 25 (NM/Hell) or 492 pvp_lld_cd 25 (Normal). Players in **166**: +400 stat 27 (manarecoverybonus), +400
  life, +200 mana.

### 5.3 Zone keys (0x102DC520, READ)
Key 0 density ×(1+v/100); 1 area level += v; 2 champion/unique counts ×(1+v/100); 3 add a monster type; **4 skirmish**:
density/rarity ×(1+v/100) and monster list += {76 maxhp% +100, 25 damage% **+40**, 329/330/331/332 elemental
masteries **+40**, 357 magic mastery +20, 493 = v}; 12 drop bonus; 14: {76 +200 % life, 276 treacherous = v}.

---------------------------------------------------------------------------------------------------------------------

## 6. The monster damage roll (D2Game 0x6FCFD450, READ except where noted)

### 6.1 Physical
`defense.md` §2.5: min8/max8 from stats 21/22 (+111), `pct = EnDmgPct + stat 25 (≥ −90)`, +stat 18/17,
roll, × src/128. PD crit/DS (stats 337+258 / 141) then MonStats Crit ×2 at the end (§2.4).

### 6.2 Elemental (0x6FCFCD80 / 0x6FCFBED0)
`min8 = stat_min << 8`; **if min8 < 8 (min stat 0) the type is skipped even when max > 0**; `max8 = stat_max << 8`;
both × (100 + mastery + extra)/100 (fire 329, light 330, cold 331, magic 357, poison 332; drains none); result =
X + rand(Y−X) (Y ≤ X → X); × src/128. Cold length += stat 56 × src/128 only if cold > 0; stun 66; poison rate from
57/58 (no << 8), length = stat 101 (override) or += stat 59, ÷ stat 326 when > 1.
**Conversion** (dmg+0x65 type, +0x68 %): muldiv(phys, %, 100) moves physical into the element; poison gets
**÷8** and length ≥ 50; cold/freeze length ≥ 50.

### 6.3 Drains (PD2 change at 0x6FCFD746 → PD 0x102ED2A0 → 0x1026ED30, READ)
Life drain 60/61 and stamina 64/65: stock `stat << 8`. **Mana drain 62/63: PD2 uses `<< 4` (×16) for monster
attackers** and `<< 8` for everyone else. Combined with the Mana Burn umod's `<< 1` (§3.1):
- Mana Burn unique, Hell, DM 100: stats 132–200 → **8.25–12.5 mana per hit** (stock: stat `<<8` twice → 16,896+ mana,
  i.e. all of it).
- El type 7 'mana' monsters (35 rows): stats are raw points → PD2 drains **x/16 mana** (stock x).

### 6.4 Monster drain on the target (stock leech 0x6FCFBA40, monster branch 0x6FCFBBCB, READ)
Fields are amounts, not percents: life = min(life field, physical dealt), mana = min(mana field, target's mana),
stamina = min(stam field, target's stamina); **the sum heals the monster's life** (so mana/stamina drain heal it too),
× the target's Drain % (100 for players, MonStats `Drain` for a monster target), capped at max life. Afterwards the
apply routine subtracts the mana field from the player's mana (0x6FCFA970, floor 0) and stamina (0x6FCFA930).
PD 0x102700F0 wraps the leech call: **no leech/drain in PvP levels** (fields zeroed), and **both leech fields halved
when its 5th argument (the apply routine's arg 3) is non-zero** — see Open questions (B's domain).

---------------------------------------------------------------------------------------------------------------------

## 7. Player defenses, in the order they act

| # | defense | where | rule (status) |
|---|---|---|---|
| 1 | Defense / AR | PD 0x10271740 | `hit_pd2.md` (VERIFIED). Monster AR = stat19 + 5·dex + skill ToHit, × (1 + stat119). Running players are rolled (stock auto-hit removed). Cap 100 %. |
| 2 | Block | PD 0x1026FB60 | (toblock + BlockFactor)·(dex−15)/(2·clvl), cap 75, no moving penalty, ÷3 vs `wraithMapMod` (Lucion) (VERIFIED) |
| 3 | Weapon block, evade, dodge, avoid | PD 0x1026FCF0 | weapon block cap 75 (2hs/ht2); moving: evade then dodge/avoid; 4-frame lockout; PvP levels halve all (VERIFIED) |
| 4 | Energy Shield | event 11, func 24 0x6FC61FB0 | **before resistances**: for phys, fire, light, cold, magic (and the 3 drain fields vs non-merc monsters; never poison): absorbed = min(d·calc1/100, mana·16/calc2); mana −= absorbed·calc2/16. calc1 = min(edmn, 90), calc2 = max(34 − blvl − es_efficiency (507), 14) (DATA) → ≥ 0.875 mana per life. State removed at 0 mana (READ) |
| 5 | Bone Armor | func 22 0x6FC6DFE0 | pool (stat bonearmor) absorbs **physical** before DR% (READ) |
| 6 | Cyclone Armor | func 25 0x6FC64600 | pool absorbs fire, lightning, cold (0x6FC645E0 ×3), before resistances (READ) |
| 7 | Resistances | PD 0x1026F680 | `B.resist`: total − pierce (monsters: no halving) + penalty 0/−40/−100 (not %DR/magic; not in PvP levels), floor −100, cap min(75 + max, 90); %DR cap 50; PvP levels 75 (N) / 80 (NM/H), no penalty, no pierce (VERIFIED, `defense.md`) |
| 8 | Flat DR / MDR | stats 34 / 35 | subtracted first, × missile damage_framerate/1024; PvP cap 25 (VERIFIED) |
| 9 | Absorb % / flat | stats 142–149 | % cap 40, flat after; not in PvP (VERIFIED) |
| 10 | Damage to mana | stat 114, event func 13 0x6FCCCDA0 | mana += stat114 % of the post-resist total (incl. 1 frame of poison); does not reduce damage (READ) |
| 11 | Life | 0x6FCFE242 | life −= total; < 1 HP → dead. Absorbed amount healed first (capped at max life) (`B.life_subtract`) |
| 12 | Chill / freeze | 0x6FCFC780 / 0x6FCFDAC0 | lengths use cold resist incl. penalty; stat 153 cannot be frozen, 118 half freeze (0x6FCFC478 → PD 0x102EFA90). PD2 chill also sets **FCR = v (not below −own FCR)** and stat 423 = 3v (0x6FCFC88E → PD 0x102C0AD0) (READ) |
| 13 | Poison | PD 0x10271A70 | length × (100 − PLR(110) − penalty)/100, cap 75 (part C) |
| 14 | Curses | PD 0x102BFFB2 | duration × (100 − curse_res)/100, player curse_res cap 75; stat 504 scales curse value (`uber_review.md`) |
| 15 | Hit recovery | FINDINGS | FHR EIAS table (VERIFIED) |
| 16 | Holy Shield / Fade / Cloak / Shiver / Chilling | Skills.txt aurastats | +block & defense (HS), resists + curse res (Fade), −enemy defense (Cloak), +defense (armors) — plain stats (DATA) |

---------------------------------------------------------------------------------------------------------------------

## 8. PvP summary (PD2)
- Any hostile player damage: P = 16 / 12 / 6 % × per-skill factor (§4), level 166 −2; safe-zone rectangles → 0.
- In PvP levels 157/159/166: no leech/drain (PD 0x102700F0); resist cap 75/80, no difficulty penalty, no pierce, no
  absorb, flat DR/MDR cap 25 (`B.resist`); block/avoid/evade/dodge halved (`hit_pd2.md`); monster/summon regen capped
  at 30 per tick unless state 205 (PD 0x102689B0 on 0x6FC97CBB); −4 extra bone spears / holy bolts; pvp cooldown
  stats; level 166 players +400 life, +200 mana, +400 % mana recovery.
- CB vs players divisor 10 (`B.cb`); OW (`B.ow`).
- **Leech outside the PvP levels scales twice** (READ + VERIFIED part): the percent also
  multiplies the leech fields (they carry the attacker's leech % at that point), and leech is computed from the
  already-scaled physical damage, so life leech per hit ∝ P² (Hell: 0.36 % of normal).

## 9. Monsters hitting mercenaries and summons
- Same roll and apply; defender type 1. Damage percent 100 % except prime evils (300 % vs summons, 100 % vs mercs) and
  the PvP-level 85 % for summons (§4).
- Resist: no difficulty penalty for monster defenders, cap 90 when player-owned (`B.resist`); merc resists come
  from Hireling.txt per level (`combat.js mercView`).
- Block: #10212 returns MonStats/Hireling `ToBlock` for monsters; no dex formula.
- Monster drain from a merc/summon uses the defender's MonStats `Drain` (§6.4).
- Chill length on monster defenders ÷ MonsterColdDivisor (1/2/4), freeze ÷ MonsterFreezeDivisor.

## 10. Bosses and ubers (pointers)
- Per-boss numbers: Lucion phase ×2+3 / ×3+6 skill values (0x10300141), Tristram minion spawns,
  Static Field exclusions; pinnacle scaler (+4n all skills and 7n+2t absorbs for Clone, Rathma, Lucion).
- Uber Mephisto: Conviction level 20 (stock special case kept in PD 0x102C62B0).
- The uber Ancients have MonStats Crit 0 and Drain(H) 0.

---------------------------------------------------------------------------------------------------------------------

## 11. Addresses and PD2 patch records on these paths
Records are `{module, rva, value, relative, size}` (module 3 = D2Game). "rt" = built at runtime (0x100F9839…),
"st" = static `.rdata`.

| site | → | what |
|---|---|---|
| D2Game 0x6FCCFF0C | rt → 0x102EEE40 → 0x10268D00 | monster level (map +arealevel, desecrated 85) |
| 0x6FD1B61C…0x6FD1B635 | rt data | player-count HP% 70/140/…/490 |
| 0x6FD01D90 (28 callers) | → 0x102C85A0 | post-spawn: map mods 0x102DBFE0 → 0x102DBA50, pinnacle scaler |
| 0x6FC3FF59 | st → 0x102EDC00 → 0x102C7F10 | second spawn path, map mods |
| 0x6FC422DF, 0x6FC422F4 | st byte = 1 | Mana Burn `shl 8` → `shl 1` |
| 0x6FC442EB | st data → 0x102C0EB0 | Cursed umod callback → MonAmplifyDamage via PD curse routine |
| 0x6FC44E4E (+18 NOPs) | st → 0x102EF9D0 → 0x102C62B0 | aura enchanted pick and level |
| 0x6FC44E05 and 26 more calls of 0x6FCC2100 | rt → 0x102C8190 | aura/skill start wrapper |
| 0x6FC41825/35/95, 0x6FC469C1/D5/E5, 0x6FC95F2B/FC0, 0x6FCFD13A/D179, 0x6FCFE233 | st → 0x10253DE0 | umod event dispatcher (10 slots, event 4 once per frame) |
| 0x6FC97CBB | st → 0x102689B0 | regen stat read: cap 30 in PvP levels |
| 0x6FCFC180 | st → 0x102ED040 → 0x1026D1D0 | damage percent (§4, VERIFIED) |
| 0x6FCFC478 | st → 0x102EFA90 | cold/freeze immunities |
| 0x6FCFC4EA | st → 0x102ED620 → 0x1026F410 | per-type resist/absorb (`B.resist`, VERIFIED) |
| 0x6FCFC88E | st → 0x102C0AD0 | chill: attackrate + FCR + stat 423 |
| 0x6FCFD41B | st → 0x102ED580 | hit-processing call (not traced) |
| 0x6FCFD52C (+131 NOPs) | st → 0x102ED930 | crit / deadly strike (`B.crit`) |
| 0x6FCFD746 | st → 0x102ED2A0 → 0x1026ED30 | mana-drain roll ×16 for monsters |
| 0x6FCFE219 | st → 0x102EE040 → 0x102700F0 | leech: none in PvP levels, halving flag |
| 0x6FCFE288 | st → 0x102ED5A0 → 0x1026F190 | stun |
| 0x6FCFE2AF | st → 0x10271A70 | poison |
| 0x6FCFB5B8 / 0x6FC6E8AD | st → 0x10268600 | heal-life setter (stat 488 lifedrain_percentcap gate) |
| 0x6FCFB0C0 | st byte 0x33 | stock weapon-block value forced to 0 (PD does its own) |
| 0x6FCFB7A0/7C2/846, 0x6FCC10F1 | st → 0x102EDAE0 | block routine hooks |
| 0x6FCFE559, 0x6FC5AFF7 | st → 0x102EFA50 | area-damage missile call |
| 0x6FCFC076 | rt → 0x102EF9A0 | call in 0x6FCFC030 (not traced) |
| 0x6FC64862 | rt data 0x150 | record-size constant (0x144 → 0x150) in Cyclone Armor's table walk |

Unpatched and relevant: 0x6FC97240 (attack stats), 0x6FC970E0 (Crit), MonUMod spawn handlers other than the byte and
callback patches above, ES/Bone Armor/Cyclone absorb functions, damage-to-mana 0x6FCCCDA0, stock leech core
0x6FCFBA40, #11089.

## 12. Discrepancies with our JS models (and what was fixed)
| model | issue | action |
|---|---|---|
| `defense.js playersHP` | stock 50 %/player; PD2 table is 70 %/player (2–8) | **fixed** |
| `defense.js monsterAt` elemental | El min/max/dur per slot did not match the attack-time code (§2.3) | **fixed** (`monsters.json` regenerated) |
| `defense.md` §1.2 | "monster owned by a monster deals 50 %" | corrected to 100 % |
| `combat.js monstersOnYou` | physical only: ignores El damage, MonStats Crit ×2 (5 %), PD crit/DS, map mods, ES/Bone Armor, dodge/avoid/evade; block uses the displayed chance | not changed (a model limit); worth adding El + Crit |
| `defense.js monsterAt` | poison El reported as raw MonStats min/max/dur; the game's stats are ×10/×10/×2 per-frame units | documented only |
| `combat.js effectiveLife` | ignores ES/Bone Armor (which act before resistances) | documented only |

Tests after the fix: `adv/engine/test_combat.js`, `test_derive.js`, `test_armory.js`, `test_roofoo.js`,
`adv/re/test_hit.js` (0 failures), `test_drops.js` (0 mismatches) all run. The only output difference in
`test_combat.js` (dps 7606 → 7572) comes from part C's concurrent poison change in `combat.js`, not from these fixes.

## 13. Open questions
1. **Leech halving** (for part B): PD 0x102700F0 halves life and mana leech when its 5th argument is non-zero. Via
   the 0x6FCFE219 wrapper that argument is 0x6FCFE0C0's arg 3 (the "run resist pass" flag, 1 at 24 of 26 callers).
   If that stack reading is right, leech from the stock apply path is halved everywhere; PD's own apply (0x10271F64)
   passes 0. Worth a native test of the wrapper frame.
2. Spectral Hit's random elemental add (umod 27 event 0, 0x6FC42B60) not decoded.
3. Fire Enchanted death explosion: the output slot (MaxHP) and the `<<6` split are READ only; the resulting numbers look
   small (≈ 0.4–0.6 % of the row's scaled max life as physical + the same as fire in Hell).
4. The PD skill-id writers at 0x1026E9CF/0x1026EA1F/0x1026FB22/0x102710E6 (which missiles/areas set the PvP skill)
   were only skimmed.
5. `0x102ED580` (0x6FCFD41B) and `0x102EF9A0` (0x6FCFC076) not traced.
6. "Corrupted ears presets" were not found as damage code; no ear-related branch appears in these paths.
