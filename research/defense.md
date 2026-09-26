# Player damage taken, monster attack rating and damage, and monster hit chance (PD2 = 1.13c + ProjectDiablo.dll)

Code: `adv/re/defense.js`. Monster data: `adv/re/extract_monsters.py` writes `adv/re/monsters.json`.

Status labels:
- **VERIFIED**: the real DLL code ran natively and matched `defense.js` on every case.
- **READ**: read from the disassembly only.

Other conventions:
- `tdiv` is C division (truncates toward zero).
- Damage and life are 8.8 fixed point (×256) inside the game. JS functions whose names end in `8` use those raw values.
- Stat ids are ItemStatCost ids.

**Data source.** The default is `excel_live/`, the launcher's `data.zip`, which is what the game loads. The local `/mnt/user-data/uploads/.../excel` copy has:
- identical MonLvl, MonUMod and DifficultyLevels;
- identical stats for every shared MonStats row;
- 223 extra `d1_*` monsters;
- edited Levels.txt spawn lists.

Use `--src` to switch.

## 0. Native verification done for this page

The harness sources are in the session scratchpad (`dmg.c`, `hit.c`, `gen.py`, `genh.py`, `cmp.js`, `cmph.js`). They follow `harness/pdual.c` and load the real `harness/D2Game.img` and `PD.img`.

| Test | What ran | Cases | Result |
|---|---|---|---|
| One damage type, PD2 | `0x102ED620 → 0x1026F410` (+`0x1026F680` resist, `0x1026EA70` pierce, `0x1026F820` absorb, `0x102CE840` multiply) | ~78k | 0 mismatches against `applyType8` |
| One damage type, stock | `0x6FCFBD00` (+`0x6FCFB3C0`, `0x6FCFAD00`, `0x6FC214D0`) | ~42k | 0 mismatches against `applyType8(…, {stock:true})` |
| Whole resist pass | `0x6FCFC0B0(game, att, def, dmg)` with PD2's call patch at `0x6FCFC4E9` applied | 60k | 0 mismatches against `applyHit8` |

Notes on the tests:
- The two single-type tests used random stats: every resist, max resist, pierce, absorb, DR and MDR value; difficulties 0–2; the 0x400 bypass flag; PvP on/off; damage from −500 to 3,000,000 in 8.8 units.
- The stock and PD2 results differ in 28.5k of the cases, so the tests exercise both code paths.
- The whole-pass test covers all 9 damage fields, `damage_framerate`, the bypass flag, and the total at +0x4C and absorbed amount at +0x48.

Stubs:
- The stat getter reads a per-unit table.
- State checks return 0.
- PD `0x102CB180` (owner lookup) returns 0, which means a monster attacker.
- PD `0x102CEAE0` (PvP-level test) returns the test flag.
- In the whole-pass test only: the damage-percent call at `0x6FCFC17F` returns 100 (see §1.2, READ), and the freeze call at `0x6FCFC477` is a no-op.

## 1. Damage taken by a player (server, D2Game)

### 1.1 Call chain (READ, except where marked)

1. **Hit roll** `0x6FCFDE90` (VERIFIED earlier, FINDINGS). Then block or avoid (`0x6FCFB790` → PD `0x1026FB60`, see block_regen.md).
2. **Attacker damage roll** `0x6FCFD450`:
   - Physical: `0x6FCFC530`, see §2.5.
   - PD2 crit/deadly strike (§1.6).
   - Elemental rolls `0x6FCFCD80`.
   - Then **MonStats `Crit`** `0x6FC970E0` (call at `0x6FCFDA6C`, not patched): with `Crit`% chance (5 for almost every monster), **every damage field is doubled** (phys, fire, light, magic, cold, poison).
3. **Apply** `0x6FCFE0C0` (called from `0x6FCFDE2D`):
   - **resist pass** `0x6FCFC0B0` (§1.2);
   - events `damagedinmelee` (1) / `damagedbymissile` (2) at `0x6FCFE1C4..F2`, which run damage-to-mana;
   - leech `0x6FCFBA40` (PD `0x102EE040`);
   - **absorbed amount healed** `0x6FCFB570(dmg+0x48)`, capped at max life (PD `0x10268600` adds a stat 488 `lifedrain_percentcap` gate);
   - **life −= total** (`0x6FCFE242..0x6FCFE264`): `life8 = life8 − total8; if (life8 < 256) life8 = 0`, which is death.
   - Then the stun, cold, freeze and poison handlers run. Poison: `0x6FCFCAB0` → **PD `0x10271A70`**, not traced.

### 1.2 Resist pass `0x6FCFC0B0(game, attacker, defender, dmg)` (VERIFIED as a whole)

1. **Flat reductions** (`0x6FCFC125..0x6FCFC17B`):
   - `DR8 = stat34 << 8` and `MDR8 = stat35 << 8`, whole points.
   - If the damage struct's `+0x54` > 0, each becomes `muldiv(dmg+0x54, DR8, 1024)`. That field is the attacking missile's stat 327 `damage_framerate` (set at `0x6FC5AFB6`).
2. **Damage percent** (`0x6FCFC17F`: stock `0x6FCFAC10`; **PD `0x102ED040 → 0x1026D1D0`**):
   - Stock: PvP 17%, some merc cases 25%, otherwise 100.
   - PD2 applies the percent to the 12 fields itself and returns 100.
   - For a plain monster hitting a player the result is **100%** (READ).
   - Correction (VERIFIED natively, `harness/pvp.c`, see `dmg_D_incoming.md` §4): a monster owned by another monster deals **100%**; the 50% branch needs the attacker to be a hireling or carry unit flag +0xC4 bit 31 while owned by a monster. All PvP / pet / merc percentages are tabulated in `dmg_D_incoming.md`.
   - The PD2 PvP tables are not covered here.
3. Event 11 `absorbdamage` (`0x6FCFC46E`). Then cold/freeze-length immunities (`0x6FCFC477`: stock `0x6FCFA7B0`, PD `0x102EFA90`). These don't change damage.
4. **Bypass flag**: when the defender is a player, damage flag `0x400` = "ignore resist/absorb" (`0x6FCFC4BF`). Normal monster hits don't set it.
5. The per-type step (§1.3) runs over the 12-row table **D2Game `0x6FD22AB0`** (stride 0x2C). The call at `0x6FCFC4E9` is **replaced by PD2** (`0x102ED620 → 0x1026F410`).

| # | row | dmg offset | resist | max resist | pierce | absorb % | absorb flat | flat reduction |
|---|---|---|---|---|---|---|---|---|
| 0 | Dam (physical) | +0x08 | 36 damageresist | – | – | – | – | stat 34 DR |
| 1 | Fire | +0x10 | 39 | 40 | 333 | 142 | 143 | stat 35 MDR |
| 2 | Ligt | +0x1C | 41 | 42 | 334 | 144 | 145 | MDR |
| 3 | Cold | +0x24 | 43 | 44 | 335 | 148 | 149 | MDR |
| 4 | Magc | +0x20 | 37 magicresist | 38 | – | 146 | 147 | MDR |
| 5 | CLen (cold length) | +0x30 | 43 | 44 | 335 | – | – | – |
| 6 | FLen (freeze length) | +0x34 | 43 | 44 | 335 | – | – | – |
| 7 | PLen (poison length) | +0x2C | 110 item_poisonlengthresist | – | 336 | – | – | – |
| 8 | Pois (per-frame rate) | +0x28 | 45 | 46 | 336 | – | – | – |
| 9–11 | life / mana / stamina leech | +0x38/+0x3C/+0x40 | – | | | | | |

6. **Total** (`0x6FCFC4F7`) = cold + poison + magic + light + fire + phys, stored at `+0x4C`.
   - The absorbed sum is at `+0x48`.
   - The poison field is the **per-frame** rate, so one frame of poison lands with the hit. The rest is applied over the length by the poison handler.

### 1.3 One damage type: PD2 `0x1026F410` (VERIFIED) and stock `0x6FCFBD00` (VERIFIED)

**Resistance** (PD `0x1026F680`, stock `0x6FCFB3C0`), for a player defender:
```
res  = defender[resStat]                                  (gear/skill total; NOT including the difficulty penalty)
res -= attacker[pierceStat]            if a pierce stat exists and the level is not PvP
       PD 0x1026EA70: if the result is < 0 and the attacker is player-owned, halve it (tdiv 2); monsters: no halving
res += DifficultyLevels.ResistPenalty  (0 / -40 / -100)   unless resStat is 36 or 37; PD2: not in PvP levels
if res <= 0: res = max(res, -100)
else: cap = (maxStat == -1) ? (resStat == 36 ? 50 : 75)
            : min(75 + defender[maxStat], HARD)           HARD = 90 in PD2 (stock 95); PD2 PvP: 75 Normal, 80 NM/Hell
      res = min(res, cap)
```
- **%DR (36) is capped at 50** and gets no penalty.
- **Magic resist (37)**: no penalty. Its max is 75 + stat 38, hard cap 90.
- **Poison length (110) gets the difficulty penalty** and its cap is 75. With 0 PLR in Hell the poison **length is doubled** (−100%).
- Cold and freeze *length* use cold resist, including the penalty.

**Damage** (PD2):
```
d = dmg[off]; if d < 1 -> 0
if !bypass:
   f = (flat==1 ? DR8 : flat==2 ? MDR8 : 0); PvP: f = min(f, 25<<8)
   d = max(0, d - f)                                                (flat reduction FIRST)
   if d > 0 and res != 0: d = trunc(d * (100 - min(res,100)) / 100) (double math, 0x102CE840)
   absorb (0x1026F820), only when the row has an absorb-% stat and d > 0, and not PvP:
      p = min(defender[absPct], 40); if p > 0: a = trunc(d*p/100); absorbed += a; d -= a
      F = defender[absFlat] << 8;    if F > 0: a = min(d, F);     absorbed += a; d -= a
   d = max(d, 0)
else (bypass): only a negative resist is applied, with no flat reduction and no absorb.
```

**Stock differences** (for comparison, `{stock:true}`):
- Hard cap 95.
- No PvP rules.
- The integer `muldiv` (`0x6FC214D0`) is used instead of double math.
- **A component is not clamped at 0**. DR larger than the physical damage leaves a negative physical value that is subtracted from the other types in the total. PD2 clamps each type at 0.

**Order, summarised:** flat DR/MDR → resistance / %DR → absorb % (cap 40) → absorb flat → clamp 0. All values are 8.8, and there is **no minimum damage** (a hit can do 0).

### 1.4 Damage-to-mana (stat 114) (READ)

- ItemStatCost row 114 has `itemevent damagedinmelee` / `damagedbymissile` with func 13. The event-func table is D2Game `0x6FD277A8`; func 13 is `0x6FCCCDA0`. PD2 does not patch it.
- If the defender is a player and `total8 > 0`: `mana8 = min(maxMana8, mana8 + muldiv(stat114, total8, 100))`.
- It uses the **post-resist total**, including the one frame of poison.
- It does **not** reduce the life damage.

### 1.5 Effective life (JS `effectiveLife(L, {S, difficulty})`, `rawToKill8`)

A hit kills from full life when `total8 ≥ 256·L − 255`. At full life the absorb heal is wasted, because it is capped at max life before the subtraction. Per type, in points (the exact 8.8 value comes from `rawToKill8`):

```
r = effective resistance (1.3), a = min(absorb%,40), A = absorb flat, F = DR (phys) or MDR (fire/light/cold/magic)
raw needed = F + (L - 255/256 + A) / ((1 - r/100) * (1 - a/100))
phys  : F = stat34, r = min(stat36, 50), no absorb
fire  : F = stat35, r = min(stat39 + pen, min(75 + stat40, 90)), a = stat142, A = stat143
light : F = stat35, r from 41/42, a = 144, A = 145
cold  : F = stat35, r from 43/44, a = 148, A = 149
magic : F = stat35, r = min(stat37, min(75 + stat38, 90)), a = 146, A = 147
poison: no DR/MDR/absorb; total = rate8 * (100 - r)/100 * len * (100 - min(plr + pen, 75))/100 / 256
        with r from 45/46 (+pen), plr = stat 110 (+pen)
pen = 0 / -40 / -100 (Normal/NM/Hell)
```

- When the player is not at full life, the absorbed part also heals before the subtraction. The net life change per hit is `total − absorbed`, but life never goes above max.
- Example (`node`, Hell): stats 34=15, 35=10, 36=30, fire res 175 (75 on the character screen), 142=10, 143=15, L = 2500.
  - Physical kill needs **3585.0** raw.
  - Fire needs **11183.3**. The closed form and the exact 8.8 search agree.

### 1.6 PD2 crit and deadly strike, all attackers (READ)

- Stock `0x6FCFD52C` (player weapon-mastery crit) is replaced by a call to PD `0x102ED930 → 0x1026F8C0 → 0x10270E00`, with 131 NOPs after it.
- The decision is made in `0x10270D20`:
  - **Crit**: chance = mastery crit (players) + stat 337 + stat 258, capped at 75. Physical × (200 + stat 256)/100.
  - **Deadly strike**: stat 141, capped at 75 + stat 210. Physical × (150 + stat 257)/100.
- Monsters get these only through stats (for example the `map_mon_deadlystrike` map mod).
- The separate MonStats `Crit` doubling (§1.1) is unchanged.

## 2. Monster attack rating, defense, damage, life

### 2.1 Level (READ)

- **Spawn** (`0x6FCCFDB0`):
  - Normal difficulty uses MonStats `Level`.
  - In NM/Hell (expansion), monsters that are not `noRatio` and not `boss` use **Levels.txt `MonLvl{2,3}Ex`** of the area (`#10894`).
  - Mercs use difficulty 0.
- **PD2 replaces that `#10894` call** (`0x6FCCFF0B` → `0x102EEE40 → 0x10268D00`):
  - **Map levels (Levels Id 137–201)**: level += the game's map mod with ISC `Divide` code 2001 = **`map_glob_arealevel` (stat 374)**. Maps store these in `game+0x2630` {u16 layer, u16 mod id, i32 value}; the lookup is `0x102DBA00` with id 1.
  - Other levels: if the level is in the zone set chosen by `game+0x26F2`, the level is **85**. That id is Hell-only and time-seeded, redrawn every 900 s (PD `0x1026C3C2`); it works like a terror-zone rotation.
- **Unique-monster modifiers** (MonUMod handlers, table `0x6FD2E550`):
  - Uniques and champions get mods 1–4 (rndname, hpmultiply, light, **leveladd**) from `0x6FD1CE7C` (`0x6FC448A0`).
  - Minions get the same 4 at `0x6FC44829` with arg 0.
  - leveladd `0x6FC41E80` = **+3 level**, XP ×5.
  - champion `0x6FC42DA0` = **−1 level**, XP ×3/5.
  - Net: **champion +2, unique +3, minion +3**.
- HP, defense and XP use the **spawn** level. Attack rating and damage are computed at attack time (§2.3) from the **current** level, which includes the +2/+3.

### 2.2 MonLvl scaling: D2Common #11089 `0x6FDA4A00` (VERIFIED for AC/TH, FINDINGS; same routine and indexing for HP/DM/XP)

- `value = muldiv(MonStats[col][diff], MonLvl[level][col][diff], 100)`. `noRatio` monsters use the raw MonStats value.
- The level is clamped to the last MonLvl row.
- **MonLvl column order** was recovered from the D2Common column table (`0x6FDA59E6`): AC +0, L-AC +0xC, TH +0x18, L-TH +0x24, HP +0x30, L-HP +0x3C, DM +0x48, L-DM +0x54, XP +0x60, L-XP +0x6C. That matches the txt order.
- The L- columns apply when the game-type byte `game+0x6A` ≠ 0 or `game+0x74` (ladder) is set.
- DifficultyLevels column offsets were recovered the same way: ResistPenalty +0, MonsterSkillBonus +0x10, LifeStealDivisor +0x28, UniqueDamageBonus +0x30, **ChampionDamageBonus +0x34**.

### 2.3 Attack rating and damage at attack time: `0x6FC97240` (READ)

- The attack mode picks the columns:
  - A2 (mode 5): A2TH / A2MinD / A2MaxD.
  - SC or S1 (modes 7–8): S1TH / S1MinD / S1MaxD.
  - Otherwise: A1TH / A1MinD / A1MaxD.
- The values come from the scaler at the current level (TH and DM columns) plus MonStats `SkillDamage` extras (`0x6FCBF2B0`).
- **Player count** (expansion, NM/Hell, p ≥ 2): AR, min and max each get `x += tdiv(x·f,128)`, with f = [0,0,8,16,24,32,40,48,56][p], or 8p−16 when p ≥ 9 (`0x6FC970A0`).
- The results are set as stats 21, 22 and 19.
- Elemental El1–3: when `ElMode` equals the attack mode and the `ElPct` roll succeeds, `ElMinD/ElMaxD` are scaled by DM and `ElDur` is taken raw. Both get the player factor. The result goes to stats 48/49 (fire), 50/51 (light), 52/53 (magic), 54/55/56 (cold, length), 57/58/59 (poison: min×10, max×10, length×2), 60/61 (life drain), 62/63 (mana), 64/65 (stamina), 66 (stun), and 315–317 (burn). `rand` picks one of fire, light, magic, cold or poison, with a length of 25 if the length is 0.

### 2.4 Champion, unique and minion bonuses (READ, constants from MonUMod `constants`)

- **HP** (hpmultiply `0x6FC42FA0` → `0x6FC41EC0`, `life += life·pct/100`):
  - champion `c[4+d]` = 200 / 150 / 100;
  - unique `c[7+d]` = 300 / 200 / 100;
  - minion `c[1+d]` = 100 / 75 / 50.
  - This comes after the player-count HP% (`0x6FCCF5F0`: PD2-patched table [0,0,70,140,…,490][p] (stock 50/step), or 50(p−2) when p ≥ 9).
- **Champion** (`0x6FC42DA0`), with CDB = DifficultyLevels ChampionDamageBonus (90 / 75 / 66):
  - stat 25 `damagepercent` += `tdiv(c[11]·CDB,100)` = **+66% in Hell**;
  - stat 119 `tohit%` += `tdiv(c[10]·CDB,100)` = **+49% in Hell**.
  - Damage is halved for base class 118.
- **Strong** (umod 5, `0x6FC42BC0`):
  - unique: dmg `c[15]` (150), tohit `c[13]` (100);
  - minion: dmg `c[14]` (75), tohit `c[12]` (50);
  - each × CDB/100.
  - A unique without `strong` gets **no** AR% or damage% bonus.
- Berserk / possessed champions: HP −75% / +100% plus the champion handler. Ghostly: stat 36 = 80 (not modelled).

### 2.5 Physical damage per swing (`0x6FCFC530`, roll helper `0x6FCFBED0`) (READ)

```
min8 = max(stat21,1)<<8, max8 = max(stat22,2)<<8 (+ stat111<<8 each); if max8 <= min8: max8 = min8 + 256
pct = dmg.EnDmgPct + stat25 (>= -90); min% = pct + stat18, max% = pct + stat17
X = min8 + muldiv(min8, min%, 100), Y = max8 + muldiv(max8, max%, 100)
phys8 = base + X + rand(Y - X)          (rand in [0, n), so the top value is Y - 1/256)
× damage-rate/128 (0x80 = 100% for a normal monster attack); then PD2 crit/DS (1.6); then MonStats Crit ×2 (1.1)
```

### 2.6 Defense and life at spawn (`0x6FCCFDB0`) (READ)

- Defense: stat 31 = scaler AC at the spawn level. No other change except umods such as stoneskin, and PD2 map mods (`map_mon_ac%`).
- Life = rand(MinHP..MaxHP) after scaling, × (1 + player%/100), capped at 0x7FFFFF, then the umod HP%.

### 2.7 JS

- `monsterAt(monId, difficulty, level, kind, opts)` → `{lvl, spawnLvl, ar, arBase, arPct, def, dmgMin, dmgMax, dmgPct, elem[], life:{min,max}, xp, res:{phys,magic,fire,light,cold,poison}}`.
  - `level` is the area level. For Normal difficulty, `noRatio` or `boss` monsters it is ignored and MonStats `Level` is used.
  - `kind` is `normal`, `champion`, `unique` or `minion`.
  - `opts`: `{players, ladder, attack:'A1'|'A2'|'S1', strong, dmgPct, arPct}`. `dmgPct` and `arPct` are for PD2 map mods, which are not traced.
  - `ar` is the value that enters the hit roll (it includes stat 119).
  - `dmgMin` and `dmgMax` are in points after the damage% and player factor, before crits.
- `typicalHellTable(areaLevel, player, opts)`: rows for `HELL_SET`, with `summary.{kind}.median` and `.max`.
  - `HELL_SET` holds the melee monsters that recur in Hell `nmon` lists of areas with MonLvl3Ex ≥ 80: Devilkin `fallen3`, Hungry Dead `zombie2`, Blood Clan `goatman3`, Preserved Dead `mummy4`, Steel Weevil `scarab4`, Unraveler `unraveler3`, Ghoul Lord `vampire1`, Gorbelly `blunderbore2`, Doom Knight `doomknight1`, Minion of Destruction `minion1`, Blood Lord `bloodlord1`, Balrog `megademon1`.
  - `MAP_SET` holds PD2 map-only rows (`minionmap`, `goatmanmap`, `doomknightmap`, `zombieSiege`, `cr_archermap`). These have their own, higher MonStats values. Map Levels rows already have MonLvl3Ex 85–90, and `map_glob_arealevel` is added on top.
- Area level 85, Hell, 1 player. Hit % is against a level-90 player with 2000 defense.

| monster | AR | damage | hit % | champion AR | champion damage | hit % | unique AR | unique damage | hit % | defense | life |
|---|---|---|---|---|---|---|---|---|---|---|---|
| fallen3 | 3067 | 43-86 | 58 | 4671 | 73.0-146.1 | 68 | 3169 | 45-90 | 60 | 982 | 1159-2550 |
| zombie2 | 3578 | 57-134 | 62 | 5450 | 96.3-227.4 | 71 | 3698 | 60-140 | 63 | 1122 | 4868-6955 |
| goatman3 | 2896 | 62-115 | 57 | 4411 | 104.6-194.2 | 66 | 2993 | 65-120 | 58 | 841 | 4637-6028 |
| unraveler3 | 5452 | 105-134 | 70 | 8305 | 177.6-227.4 | 78 | 5635 | 110-140 | 72 | 1683 | 7882-9737 |
| doomknight1 | 6134 | 67-144 | 72 | 9343 | 112.9-244.0 | 80 | 6339 | 70-150 | 75 | 1613 | 5564-6955 |
| bloodlord1 | 6134 | 76-144 | 72 | 9343 | 129.5-244.0 | 80 | 6339 | 80-150 | 75 | 1823 | 11128-13911 |
| megademon1 | 5452 | 96-153 | 70 | 8305 | 162.7-259.0 | 78 | 5635 | 100-160 | 72 | 1613 | 9737-11592 |

## 3. The player's defense in the monster's hit roll (VERIFIED, FINDINGS / charscreen.md)

The hit roll is `0x6FCFDE90`:

```
def = #10672(player) + (missile ? stat32 armorclass_vs_missile : stat33 armorclass_vs_hth)
      #10672: base = stat31 + tdiv(dex,4); pct = stat16 + stat171 (+ Holy Shield); def = base ± tdiv(base·pct,100)
def += tdiv(def · stat182, 100)                           (armor_override_percent)
monster: AR = stat19 + 5·dex + skill ToHit; AR += tdiv(AR · stat119, 100); no target-specific adjustments
chance = clamp(5, 95, tdiv(tdiv(AR·100, AR+def) · 2·mlvl, mlvl + clvl))   (negative def/AR handled as in FINDINGS)
```

- Confirmed: the defense a monster rolls against is the character-screen defense (#10672) plus stat 33 against melee or stat 32 against missiles, then stat 182.
- A player in Run mode (3) is hit without a roll (caller `0x6FCFE5A0`, READ).
- None of these functions are patched by PD2.
- `mlvl` is the monster's current level, which includes the champion +2 and unique +3.
- JS: `playerDefense(S, missile)` and `hitChance(ar, mlvl, def, clvl)`.

## 4. PD2 patch records touching this code

| Address | Patch | Effect |
|---|---|---|
| D2Game `0x6FCFC4EA` | → `0x102ED620` | per-type resist/absorb (VERIFIED) |
| `0x6FCFC180` | → `0x102ED040` | damage percent |
| `0x6FCFC478` | → `0x102EFA90` | freeze/cold immunities |
| `0x6FCFD52C` | call `0x102ED930` + 131 NOPs | crit/deadly strike |
| `0x6FCFE219` | → `0x102EE040` | leech |
| `0x6FCFE2AF` | → `0x10271A70` | poison application |
| `0x6FCFE288` | → `0x102ED5A0` | handler at `0x6FCFCC20`, not traced |
| `0x6FCFB5B8` | → `0x10268600` | absorb-heal life set, stat 488 |
| `0x6FCCFF0C` | code-init → `0x102EEE40` | monster level |

- The following are not patched: the MonLvl scaler, the hit roll, #10672, monster attack-time AR/damage `0x6FC97240`, the MonStats Crit doubling `0x6FC970E0`, the MonUMod handlers used here (PD2 patches only calls to `0x6FC468E0` inside `0x6FC41820/30/80`), and damage-to-mana `0x6FCCCDA0`.

## 5. Caveats / OPEN

- **Resist inputs** are totals **before** the difficulty penalty. The character screen shows total + penalty, so screen 75 = stat 175 in Hell.
- **PD2 map monster mods** (`map_mon_*`: tohit, ED%, pierce, AC%, max HP%, phys-as-extra-element, …) are applied by PD2 code that was not traced. Pass them as `opts.dmgPct` / `opts.arPct` or `attackerPierce`.
- **Map player mods** (`map_play_*resist`, ISC codes 1039–1046 = stat id + 1000; `map_play_damageresist` 1036; `map_play_ac%` 1016) appear to be added to the player's own stats. Include them in `S`.
- The poison DoT and stacking (PD `0x10271A70`) is not traced. The monster poison unit conversion (×10 rate, ×2 length) is READ only.
- The damage percent for monster → player being 100 (PD `0x1026D1D0`) is now VERIFIED (`dmg_D_incoming.md` §4).
- Champion +2 relies on champions running the auto mod list 1–4 (`0x6FC448A0`). The hpmultiply champion branch (MonsterData flag 4) supports this, but the spawn path for champions was not followed end to end.
- `kind:'superunique'` is treated like `unique` (+3 level, unique HP%). The SuperUniques spawn path (`0x6FC6FD00`…, which applies the row's Mod1–3) was not followed for the level bonus.
- PD2 realm servers may run different server code. This covers the client install's D2Game.
