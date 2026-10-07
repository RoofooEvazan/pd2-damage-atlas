# Experience in PD2 (1.13c + ProjectDiablo.dll), end to end

This covers how much experience a kill gives, how a party splits it, the death penalty, the level cap, and whether the in-game Advanced Stats panel (BH.dll) shows the right "XP %".

**Status labels**
- **VERIFIED**: the real game code was run natively (`harness/exp.c`, driver `harness/exp_check.js`) and matched `adv/re/experience.js` on every case.
- **READ**: read from the disassembly only.
- **DATA**: from PD2's tables (`pd2data.mpq` == `data.zip`).

> **Correction (audit):** the two copies are not identical in general (Misc, Levels row 193, CubeMain, AutoMap, LvlPrest, LvlTypes and string ids differ; see `data_sources.md`). They are identical for every table this page uses (MonStats numbers, MonLvl, Experience, Levels outside row 193), so the numbers here stand.


**Native run:** `node harness/exp_check.js 50000` gives 50,000 cases and **0 mismatches**:
- 30,000 kills, 36,572 exp awards, 9,372 multi-member party splits and 12,801 merc awards;
- 5,000 deaths;
- 5,000 exp applies;
- 5,000 champion/unique modifier runs;
- 5,000 spawn player-count runs.

Each deliberate model mutation fails. The mutation names and mismatch counts at 20k cases:

| Mutation | Mismatches |
|---|---|
| `party89` | 3,793 |
| `range` | 453 |
| `merc86` | 2,747 |
| `gate` | 146 |
| `death` | 451 |
| `champ` | 976 |
| `players` | 2,000 |
| `zero1` | 96 |
| `double` (float64 instead of float32 in the split) | 19 |

## 0. Summary

```
monster stat 13 (at spawn) = MulDiv(MonStats Exp[d], MonLvl XP[mlvl][d], 100)       (#11089; noRatio: raw)
                           + MulDiv(pct(players), that, 100)     pct = 0,0,50,100,...,350 (p=1..8), 10p+260 (p>=9)
                           x5 (unique, superunique, minion, berserker champion)   x3 (champion/ghostly/fanatic/possessed)
kill (PD 0x102CA4B0):  merc gets gain(monsterXp at its own level) x86/256 (unless it killed), then x2 into its exp
                       not in a party / 1 member in range -> killer gets gain(monsterXp)
                       n >= 2 in range (80 subtiles): E = X + ((n-1)*X*89 >>> 8);  share_i = trunc(level_i * float32(E / sumLevels))
gain (0x6FCFC030):     x = min(x, 8,388,607); x <= 0 -> 1;  clvl 99 -> 0
                       x = levelPenalty(x, clvl, mlvl);  x = x*ExpRatio[clvl] >> 10;  x += MulDiv(stat85, x, 100)
apply (PD 0x102C9AF0 -> 0x6FCFE6D0): nothing below clvl 80 in Uber/Pandemonium zones; total capped at 3,520,485,254
death (0x6FC57730):    lose 0/5/10 % of the current level's span (N/NM/H), never below the level's start
```

**PD2 changes.** PD2 replaces:
- the whole kill handler (`0x6FCFEA00` → PD `0x102CA4B0`);
- the level-gap penalty (`0x6FCFAA40` → PD `0x102CACF0`, player levels 20–24 only);
- the apply step (`0x6FCFE6D0` → PD `0x102C9AF0`, which adds the level-80 gate).

PD2 does **not** change these:
- the player-count exp bonus;
- the champion/unique multipliers;
- ExpRatio, stat 85, the party range, the 89/256 party bonus, the death penalty and the level cap.

**BH panel verdict: approximate, and wrong where it matters most.** The formula shape (level factor × ExpRatio) is right. But the level factor is one fixed number per act and difficulty. It does not match PD2's area levels:
- In Hell Acts 1–2 at clvl 80–90 it is **2–12× too low**. For example, it shows 0.48% at clvl 90 in Hell Act 1, where a level-85 area really gives 5.96%.
- It shows **0.00% in every map and Uber zone** (level id ≥ 137).

Details are in §6.

## 1. Monster kill XP

### 1.1 Base value (DATA + VERIFIED routine)

- D2Common `#11089` (`0x6FDA4A00`) computes `MulDiv(MonStats Exp/Exp(N)/Exp(H), MonLvl XP/XP(N)/XP(H)[level], 100)`.
  - `noRatio` monsters use the raw MonStats value.
  - The L-XP (ladder) column is used when `game+0x6A != 0` or the game is ladder.
  - This is the same routine and column table as AC/TH/HP, and it is VERIFIED for those in defense.md.
  - The XP column is +0x60 / +0x6C (READ, defense.md §2.2).
- The level is the **spawn** level:
  - MonStats Level in Normal;
  - Levels.txt MonLvl{2,3}Ex in NM/Hell;
  - plus PD2 map/zone changes (defense.md §2.1).
- Worked examples with PD2 data (`defense.js monsterAt(...).xp`), Hell, area level 85, 1 player:

| Monster | XP |
|---|---|
| Doom Knight (`doomknight1`) | 51,195 |
| Minion of Destruction (`minion1`) | 40,956 |
| Fallen (`fallen3`) | 33,276 |

### 1.2 Player count (READ; formula VERIFIED)

The spawn code at D2Game `0x6FCD0094..0x6FCD00B0` computes:

```
stat13 = xp + MulDiv(pct, xp, 100)          MulDiv 0x6FC214D0 (eax=pct, edx=xp, ecx=100)
pct    = 0x6FCCF5D0(p):  p < 9 ? [0,0,50,100,150,200,250,300,350][p] : 10*p + 260     (table D2Game 0x6FD1B5F0)
```

- **p** is the number of players in the game **when the monster spawns**, not when it dies.
  - The count comes from `0x6FC579A0`, called via `0x6FCCF840`.
  - It counts players who are not in death/dead mode (0 / 0x11) and whose `+0xC6` bit 0 is clear.
  - In game types 1–3 (single player / open) the count is at least the `/players` setting at `0x6FD31C1C`.
- Monsters with MonStats **Align** ≠ 0 (byte +0x4C of the record) always use p = 1.
- **PD2 patches the HP table next to it** (`0x6FD1B614`: +70% HP per player; defense.js `playersHP`).
  - It does **not** patch the exp table `0x6FD1B5F0` (no patch record in 0x6FD1B5F0..0x6FD1B613, DATA).
  - So PD2 keeps stock **+50% exp per extra player** while monsters get +70% HP.
- `0x6FCCF840` puts the HP factor in slot 0 and the exp factor in slot 1. The spawn code reads slot 1 (`[esp+0x20]`) for XP and slot 0 for HP (READ).

VERIFIED: `0x6FCCF5D0` + MulDiv run natively for p = 0..30 and XP up to 5·10⁷ (5,000 cases).

### 1.3 Champion and unique multipliers (VERIFIED arithmetic, READ dispatch)

The MonUMod handler table is at `0x6FD2E550`.
- **leveladd** (umod 4, `0x6FC41E80`): `level += 3; stat13 = stat13 * 5` (32-bit).
- **champion** (umod 16, `0x6FC42DA0`): `level -= 1; stat13 = x - 2*floor(x/5)` (unsigned). After ×5 this is exactly ×3.

Who gets which:
- **Uniques and superuniques** get umods 1–4 from the list at `0x6FD1CE7C` (`0x6FC448A0`). So they get **×5 exp, +3 levels**.
- **Minions** of uniques get the same umods 1–4 (`0x6FC44850` loop). So they also get **×5 exp, +3 levels**.
- **Champions** get umods 1–4 plus their champion type:
  - Champion (16) calls `0x6FC42DA0`. Ghostly (36, `0x6FC43850`), fanatic (37, `0x6FC43810`) and possessed (38, `0x6FC437C0`) also call it. Net **×3 exp, +2 levels**.
  - **Berserker** (39, `0x6FC42D00`) does **not** call it. Berserker champions keep **×5 exp and +3 levels**, like a unique.
- Act bosses and Uber bosses get no umods. Their exp is all in MonStats.
- PD2 patches none of these handlers (no records in 0x6FC41000..0x6FC45000 touch them, DATA).

Doom Knight, Hell area 85:

| Type | 1 player | 8 players |
|---|---|---|
| Normal | 51,195 | 230,377 |
| Champion | 153,585 | 691,131 |
| Unique / minion / berserker | 255,975 | 1,151,885 |

VERIFIED: the real leveladd and the champion exp step (`0x6FC42E11..0x6FC42E4C`) were run natively on 5,000 values, including values that overflow 32 bits.

### 1.4 Who gets the kill (PD `0x102CA4B0`, VERIFIED)

- `0x6FC977F0` calls the handler for every kill, unless the dead unit has flag `0x04000000` in `+0xC4`.
- The PD2 code-init patch at `0x6FC97812` sends that call to **PD `0x102CA4B0`** instead of stock `0x6FCFEA00`.
- It works in this order:
  1. Stop if the monster's base stat 13 is 0.
  2. Find the owning player:
     - a player attacker counts itself;
     - a monster attacker counts its owner, and that owner's owner (PD `0x102CB180`);
     - the stock "statlist 0x800 owner" override is also checked.
     If the result is not a player, nobody gets exp.
  3. **Merc** (PD `0x102D1700`): the first pet whose class is one of 271, 338, 359, 1056, 560 or 561 (`0x102D1200`).
     - Its gain is `gain(monsterXp, mercLevel, mlvl, merc stat 85)` (§2).
     - Unless the merc landed the kill, it gets `(g*86) >>> 8` (×0.336).
     - The merc award `0x6FCFDCB0` then adds **2 × that** to the merc's exp, only while merc level < player level (READ).
     - For mercs, `0x6FCFB6B0` caps each gain at `(HireExp(L+1) − HireExp(L)) >> 6`, where `HireExp(L) = L²(L+1)·Exp/Lvl` (D2Common `#10448`) (READ).
     - The merc uses the **player** ExpRatio table for its own level (VERIFIED).
  4. **Party** (party id `0x6FCFFA60` ≠ 0xFFFF): see §1.5.
  5. Not in a party: the killer gets `gain(monsterXp, killerLevel, mlvl, stat85)`. There is no range check.

### 1.5 Party split (VERIFIED)

The member list comes from D2Game `0x6FCBC070`, which walks the party members in the game (the killer included). The real callback is `0x6FCFAED0`. A member counts when all of these hold:
- it is not in player mode 0 (death) or 0x11 (dead);
- `+0xC6` bit 0 is clear;
- its x is not 0, and the dead monster's x is not 0;
- `dx² + dy² ≤ 6400` (unsigned), i.e. within **80 subtiles** of the dead monster.

With n members in range and S = the sum of their levels (stat 12):

```
n == 0 or S <= 0 : nobody gets anything
n == 1           : the KILLER gets gain(X, killerLevel)        (whoever the one in-range member was)
n >= 2           : E  = (X + (((n-1)*X*89) mod 2^32 >>> 8)) mod 2^32       ; +34.8% per extra member
                   f  = float32(float32(E) / float32(S))                   ; divss, single precision
                   share_i = (unsigned) trunc(level_i * f)                   ; PD CRT __dtoui3 0x103030B0
                   member i gets gain(share_i, level_i, mlvl, stat85_i)
```

- Stock `0x6FCFE8F0` (no longer reached in PD2) used signed arithmetic and x87 division stored to float32. The result is the same except with overflow.
- The float32 step matters. Computing f in double changes 19 of 3,745 splits by 1 exp (mutation `double`).
- Example, Doom Knight Hell 85 (X = 51,195), all members at the monster:

| Party | Share (level 90 / 85 / 80) | Gain (level 90 / 85 / 80) |
|---|---|---|
| Levels 90, 85, 80 | 30,632 / 28,930 / 27,228 | 1,824 / 7,232 / 12,412 |
| Same, but the level 85 is 141 subtiles away | 36,525 / – / 32,467 | 2,175 / – / 14,801 |

### 1.6 Per-player gain (D2Game `0x6FCFC030`, VERIFIED with the PD penalty)

```
x = int32(share);  if x > 0x7FFFFF: x = 0x7FFFFF  elif x <= 0: return 1     # yes, 1
if clvl >= 99 (MaxLvl #10066): return 0
x = levelPenalty(x, clvl, mlvl)            # PD 0x102CACF0 via the patched call at 0x6FCFC075
x = x*ExpRatio[clvl] >> 10                 # 0x6FCFA9E0 (overflow-safe path for huge x)
x += MulDiv(stat85, x, 100)                # stat 85 item_addexperience, read with #10973
```

The level-gap penalty (mf_misc.md §3, VERIFIED there and again here through the full handler):

```
mlvl <= clvl                 : x * T_le[min(clvl-mlvl,10)] / 256    T_le = 256x6, 207, 159, 110, 61, 13   (0x6FD1A01C)
mlvl >  clvl, clvl <= 19     : x * T_gt[min(mlvl-clvl,10)] / 256    T_gt = 256x6, 225, 174, 92, 38, 5     (0x6FD1A048)
mlvl >  clvl, clvl 20..24    : f = clamp(tdiv(3c, c+5d)*256 - 64, 13, 256)      (PD2 only; stock uses T_gt)
mlvl >  clvl, clvl >= 25     : trunc(x * clvl / mlvl)
```

- Experience.txt ExpRatio (DATA) is 1024 up to level 69.
- From 70 up it is: 976, 928, 880, 832, 784, 736, 688, 640, 592, 544, 496, 448, 400, 352, 304, **256 (85)**, 192, 144, 108, 81, 61 (90), 46, 35, 26, 20, 15, 11, 8, 6, 5 (99).
- Stat 85 is the sum of every statlist on the player:
  - gear;
  - the Experience shrine (+50 via state 137 for 144 s; shrines_movement.md);
  - the map player mods (§1.7).

  It is additive between sources and multiplies the gain **after** penalty and ExpRatio.

Doom Knight (51,195) solo, Hell area 85:

| clvl | Gain | % of monster XP |
|---|---|---|
| 70 | 40,183 | 78.49% |
| 80 | 23,338 | 45.59% |
| 85 | 12,798 | 25.00% |
| 88 | 5,399 | 10.55% |
| 90 | 3,049 | 5.96% |
| 95 | 38 | 0.07% |

### 1.7 Shrine and map bonuses (READ / DATA)

- The **Experience shrine** gives +50 stat 85 (shrines_movement.md).
- **Maps:**
  - ISC `map_play_addexperience` (stat 373) has `Divide = 1085`. PD's map builder (`0x102D511E`: `Divide / 1000 == 1` → player list, remainder = target stat) turns it into **stat 85** on the player list at `game+0x21FC`.
  - PD `0x102DBA50` attaches that list to the player as a state-198 (0xC6) statlist.
  - So a map's XP bonus is **plain stat 85, additive with gear and shrines**, and applied after the level penalty and ExpRatio.
- DATA ranges:
  - MagicPrefix/MagicSuffix map affixes (`t1m`/`t2m`/`t3m`): 3–24% each;
  - unique maps: 8–40%;
  - cube recipes: e.g. "map heroed" +10, dungeon upgrade +40.
- The map monster **level** (`map_glob_arealevel`, defense.md §2.1) also changes the monster's XP through MonLvl and the level penalty.
- No map mod multiplies monster stat 13.

## 2. Death XP penalty (D2Game `0x6FC57730`, VERIFIED; PD2 unchanged)

```
if clvl <= 1: nothing
lo = Exp[clvl-1]  (start of the current level),  hi = Exp[clvl]
loss = (DeathExpPenalty[diff] * (hi - lo)) / 100        unsigned; DifficultyLevels +4 = 0 / 5 / 10 (DATA)
if loss == 0: nothing
new = cur - loss;  if new <= lo: new = lo + 1, recorded loss = max(0, cur - lo)
stat13 = new;  playerdata+0x9C -> +0x508 = recorded loss
```

- **When it applies:** only when the killer is not a player and not a player-owned unit (caller `0x6FC583A0`). PvP deaths cost no exp.
- **Hell example:** a level 90 character loses 10% of 146,072,446 = **14,607,244** (Normal 0, Nightmare 7,303,622). It can never drop a level.
- **PD2's only change on death** is the gold-loss step (`0x6FC583A8` → PD `0x102C9B50`). It skips the gold loss in PvP levels 157, 159 and 166 (`0x102CEA70`). The exp step is stock.
- **Offsets:** `DeathExpPenalty` at +4 was confirmed from D2Common's field descriptor list (name `0x6FDE0694` with offset 4, at `0x6FDB0418`).

## 3. Level cap and XP table (DATA, apply VERIFIED)

- **Experience.txt** (PD2 mpq == data.zip):
  - `MaxLvl` is 99 for all 7 classes, with identical thresholds.
  - Row L is the total needed for level L+1: 500, 1,500, … 3,520,485,254 (level 99).
- **Apply** (`0x6FCFE6D0`, reached through PD `0x102C9AF0`):
  - `new = min(cur + gain (unsigned), Exp[98] = 3,520,485,254)`;
  - stat 29 = new − cur;
  - level up when `#10670` gives a new level (PD hooks the level-up call at `0x6FCFE736` → `0x102ED420`).
- No stock copy of Experience.txt is in the repo to diff against. The PD2 values are the ones above. No code in the chain changes the cap.
- **PD2 gate (`0x102C9AF0`, VERIFIED with the sets read from init code):** no exp when playerdata `+0x264` is set. No exp for characters **below level 80** standing in:
  - Pandemonium levels 133–135 (set `0x104E3484`);
  - levels 136, 137, 161, 162, 163, 168, 185, 188, 189 (set `0x104E352C`): Tristram/Pandemonium Finale, Uber Diablo, the three Rathma maps, Uber Ancients, PD2 Tristram, and the Lucion Arena/Vault.

## 4. What a character really gets: a quick reference

`realXpPct(clvl, mlvl)` in experience.js = levelFactor × ExpRatio, for one solo kill with no stat 85.

| clvl | Monster level 85 | Monster level = clvl | Monster clvl − 5 |
|---|---|---|---|
| 60 | 70.59% | 100% | 100% |
| 70 | 78.49% | 95.31% | 95.31% |
| 80 | 45.59% | 48.44% | 48.44% |
| 85 | 25.00% | 25.00% | 25.00% |
| 90 | 5.96% | 5.96% | 5.96% |

Beyond 5 levels below you, the factor drops to 81/62/43/24/5% at 6/7/8/9/10+ levels below.

## 5. JS model (`adv/re/experience.js`, UMD, `window.PD2Experience`)

```js
spawnXp({xp, players, align, kind})           // monster stat 13; kind normal|champion|berserker|unique|superunique|minion
gainXp(x, clvl, mlvl, stat85, {mercExpPerLvl, stock})   // 0x6FCFC030
killXp({monsterXp, mlvl, killer:{level,stat85,levelId,noExpFlag}, merc:{level,stat85,isKiller,expPerLvl}|null,
        party:[{level,stat85,x,y,dead,blocked,levelId}]|null, monsterPos:{x,y}})
      -> {merc:{gain,added}, players:[{who, level, share, gain, applied}]}
levelPenalty(x, clvl, mlvl, stock) / levelFactor / applyExpRatio / applyXp(cur, gain) / deathXp(cur, clvl, diff)
bhPanelXp(clvl, levelId, diff) / bhActColumn / realXpPct(clvl, mlvl)
EXP_THRESHOLD, EXP_RATIO, BH_XP_TABLE, BH_XP_RATIO, GATED_LEVELS
```

## 6. The BH panel's XP line (BH.dll `0x1007C299..0x1007C366`, READ + DATA)

**What it computes.**
- `esi` = the current level id, from the BH accessor `0x1003FE00`. That this is the level id is inferred from the act thresholds.
- `al` = difficulty (`0x1003D010`).
- The column is `difficulty*5 + act`. Acts are ids 1–39 → 0, 40–74 → 1, 75–102 → 2, 103–108 → 3, 109–136 → 4. Anything else (0, or **≥ 137**) gives −1, and the line shows **0.00%**.
- `value = T[clvl*16 + col] * T[clvl*16 + 15] / 100`, with the double table at `0x1011DE28` (100 rows × 16).
- The format string is `"XP: %.2f%% / Additional XP:ÿc: %d%%"`. The second number is stat 85.

**What the table is (DATA, dumped with tools/pe.py).**
- **Column 15 is ExpRatio in percent.** It matches Experience.txt (ExpRatio·100/1024) to 2 decimals, except two rows: level 83 shows 34.9 (true 34.375) and level 88 shows 10.6 (true 10.55).
- **Columns 0–14 are an integer "level factor %" per act and difficulty**, with a floor of 5.
  - Its shape is the level-gap penalty averaged over each act's monster levels.
  - Averaging the real penalty over the PD2 Levels.txt areas of each act reproduces it closely: the mean absolute difference per act/difficulty group is 0.3–3.9 points in 12 of the 15 groups, and 2.7 points overall.
  - It diverges exactly where PD2 raised areas to level 85: Hell Act 1 (mean difference 9.3 points), Normal Act 5 (6.0).
  - So it was built from an older area-level list, not PD2's current one.

**Comparison with the server.** BH shown vs a per-act average of the real factor vs a level-85 area. Hell, no stat 85:

| clvl | Act | BH shows | Real, act average | Real in a level-85 area |
|---|---|---|---|---|
| 70 | A1 | 93.39% | 86.28% | 78.49% |
| 80 | A1 | 11.62% | 30.44% | 45.59% |
| 80 | A5 | 45.50% | 46.39% | 45.59% |
| 85 | A1 | 2.75% | 14.61% | 25.00% |
| 85 | A2 | 14.75% | 18.67% | 25.00% |
| 85 | A4 | 25.00% | 25.00% | 25.00% |
| 88 | A1 | 0.95% | 5.19% | 10.55% |
| 90 | A1 | 0.48% | 2.44% | 5.96% |
| 90 | A5 | 3.10% | 4.23% | 5.96% |

**Verdict: approximate at best, wrong in common PD2 cases.**
- The structure is right: level factor × ExpRatio, with stat 85 shown separately (it really does multiply the result). ExpRatio is right to 0.5%.
- **Wrong: one number per act.**
  - The real factor depends on the level of the monster you kill. That depends on the area, and is +2 for champions and +3 for uniques and minions.
  - In PD2's Hell Acts 1–2, with many level-85 areas, the panel is 2–12× too low from clvl 80 up.
- **Wrong: maps and Uber zones show 0.00%.**
  - Every PD2 map, the Uber/Rathma/Lucion zones and the PD2 levels 137+ show 0.00%, although exp is normal there.
  - The level-80 gate in Pandemonium zones is not shown either: below 80 you really get 0, but the panel shows a number for levels 133–136.
- **Not modelled by the panel** (fine for a "% of base" line, but worth knowing):
  - the party bonus and split;
  - the 8,388,607 per-share cap;
  - the player-count bonus, which is in the monster's exp, not the %.

## 7. Not verified natively

- the player-count routine `0x6FC579A0` and the `/players` override;
- the merc cap and ×2 award (`0x6FCFB6B0` for mercs, `0x6FCFDCB0`), which were stubbed in the harness;
- the BH level-id accessor;
- the map stat-85 statlist attach (`0x102DBA50`);
- the contents of the two PD level-id sets. They are read from init code `0x10137470`/`0x10137540`; the harness uses them but does not build the std::sets itself.
