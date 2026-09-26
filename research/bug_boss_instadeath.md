# Bug: bosses "instantly die" in groups (monster life overflow)

Scope: stock D2Game/D2Common 1.13c plus ProjectDiablo.dll (PD) hooks, with PD2 `data.zip` excel. Game files and code only.
Confidence tags: **VERIFIED** means the real DLL code ran in the native harness. **READ** means disassembly only. **DATA** means from txt files.

## TL;DR
- Life is stored as 8.8 fixed point in a 32-bit stat. Server code reads it as a signed int.
  - Anything that pushes life/maxhp to 2^31 or above makes it negative.
  - The next damage event then sets life to 0 (`0x6FCFE24E`: `life-dmg < 0x100 → 0`).
- **Spawn is safe.** D2Game clamps spawn life to 8,388,607 HP (`0x6FCD005F`). PD2's player table goes up to +490% at 8 players.
- **Three code paths break the ceiling afterwards.** All three are reachable in PD2:
  1. **Regen overflow (VERIFIED, main cause).** The monster regen tick `0x6FC97CB0` does `life += regen` in signed 32-bit.
     - If life is within `regen` of 2^31, the sum wraps negative.
     - The code then forces life to `0x100`, so the boss is at **1 HP**. The first hit kills it.
     - A monster whose maxhp sits at the ceiling triggers this on the first regen tick. PD2 puts monsters there with its own clamps:
       - the pinnacle scaler (`0x102D2AF0`)
       - map/zone "+% monster life" (`0x102DBCC4`)
       - the spawn cap
     - It fires when that monster has any `hpregen` (stat 74). Stat 74 comes from MonStats `DamageRegen` (recomputed from the new maxhp by the maxhp callback `0x6FCF9626`) or from the map affix `map_mon_hpregen`.
  2. **MonUMod `hpmultiply` (VERIFIED).** Uniques, superuniques and champions get `maxhp += maxhp*pct/100` at `0x6FC41EC0`, with **no clamp**.
     - In Hell pct is 100 (uniques and champions) or 50 (minions).
     - Pre-mod life ≥ **4,194,305** HP (unique/champion) or ≥ **5,592,406** HP (minion) wraps negative. The monster dies on its first damage event.
  3. **PD map/zone +maxhp% (READ, arithmetic emulated).** `0x102DBCC4` computes `maxhp + (maxhp/100)*v` in 32 bits before its `0x7FFFFFFF` clamp.
     - For v > 100 the sum wraps past 2^32 and slips under the clamp.
     - Example: the +200% zone entry turns 5,592,406 HP into **1 HP**. Life is then rescaled to the new maxhp by the callback.
- Joining or leaving after spawn changes nothing. There is no rescale code: stat 100 (players) is written only at spawn, `0x6FCCFF3B`.
  - What matters is the number of living players **in the game** when the boss's room is activated.

## 1. Life pipeline (addresses)
| step | where | what | overflow-safe? |
|---|---|---|---|
| base roll | D2Common #11089 `0x6FDA4A00` (called `0x6FCD0031`) | min/max = MonStats MinHP/MaxHP(H) × MonLvl HP(H) (or L-HP(H) when game+0x6A≠0 / ladder) / 100; noRatio raw; level clamped to 110 | yes (small values) |
| roll | `0x6FCD0036..0x6FCD004F` | rand(min..max) | – |
| players | `0x6FCCF840` → `0x6FCCF5F0` table `0x6FD1B614` | pct = [0,0,70,140,210,280,350,420,490][p] (PD patches the table; vanilla 50/step); p≥9: (p−2)·50. p = living players in game `0x6FC579A0` (max with `/players` global `0x6FD31C1C` only when game+0x6A∈1..3). No difficulty gate | – |
| bonus | `0x6FCD0058` muldiv `0x6FC214D0` (VERIFIED) | life += life·pct/100 | yes |
| **cap** | `0x6FCD005F` | `if life ≥ 0x800000: life = 0x7FFFFF`; then `<<8`, SetStat 7 and 6 | **clamped** (silently loses HP, e.g. Lucion ≥5p) |
| regen | `0x6FCD00F3..0x6FCD0117` | stat74 = maxhp·DamageRegen>>12 (overflow-guarded) | yes |
| stat-change callback | `0x6FCCFF16` registers PD `0x102C6030` → forwards to `0x6FCF9470` (PD table `0x103C6CE8` = rva 0xD9470) | on maxhp change: life = ftol(life/old·new) clamped [1,new]; for monsters stat74 = ((new>>8)·R)>>4 (`0x6FCF9626`) | safe **if new is positive**; if new wrapped negative, life := new (negative) |
| PD post-spawn (all 28 calls of `0x6FD01D90` are hooked to PD `0x102C85A0`) | map mods `0x102DBFE0`→`0x102DBA50`; then pinnacle scaler `0x102C83E0`→`0x102D2AF0` | only in map levels 137–201 (`0x102CE890`) | see §2 |
| umods | `0x6FC448A0` applies inherent mods 1,2,3,4 (table `0x6FD1CE7C`) then the unit's mod list; used by superunique spawn `0x6FC451F0` (flag 0x2), unique/champion spawners, PD's own `0x102CED80/0x102CEE80` | #2 `0x6FC42FA0` → `0x6FC41EC0` | **no clamp** |
| regen tick | `0x6FC99B6D` → `0x6FC97CB0` (PD hook `0x102689B0` on its GetStat(74) only caps regen to 30 in PvP levels 157/159/166) | `life+regen` signed; `>maxhp → maxhp`; `<0x100 → 0x100` | **overflow → 1 HP** |
| damage | `0x6FCFE24E` | life −= dmg; `<0x100 → 0` (signed) | negative life dies on first hit |

Other checks:
- ItemStatCost marks 6/7 as unsigned: no `Signed` flag, Send Bits 32, ValShift 8. `hitpoints` has maxstat=maxhp, and `maxhp` has fMin.
- D2Common SetStat `0x6FD8A280` stores the raw 32-bit value. It does no width or maxstat clamping.
- GetStat `0x6FD88A80` returns it raw. Only the "full" read of an fMin stat is floored to 0.
- The server compares life signed everywhere checked.

## 2. PD2-specific scalers (READ)
- **Pinnacle scaler `0x102D2AF0`.**
  - Applies in map levels to class set `0x104E3508`: 789 (uberdiablonew), 790–794, 886–891, 933–939 (Rathma), 966, 968, 969, 1106, 1107, 1112–1115 (Lucion), 1140, 1141.
  - It is gated once by state 0xF1.
  - n = game byte +0x2629. It is 0 by default and is raised (≤3) by using item 0x49E in the arena (`0x102B8FF0`).
  - maxhp' = n≤0 ? 1.5·maxhp : k·maxhp·(1+0.2n), with k=4 for main bosses {789,933–936,1112} (`0x104E34C0`) and 3 otherwise.
  - The math is 64-bit and **clamped to 0x7FFFFFFF**, so these bosses end up exactly at the ceiling.
  - The callback then sets life = 0x7FFFFFFF. Any later heal or regen overflows.
- **Map/zone mods `0x102DBA50`.**
  - The game list at game+0x1DF8 is built from the map item's stats `0x102D4E70` (`map_mon_*` rows carry the target stat in the `Divide` column: 402→74 hpregen, 405→76 maxhp%, 403→60 lifedrain; DATA). Zone entries are added in `0x102DC777`: type 4 adds `76=+100`, type 14 adds `76=+200`.
  - Stat 76 on a monster is applied directly: `ecx=GetStat7; edx=(ecx/100)*v; ecx+=edx; if(ecx>0x7FFFFFFF u) ecx=0x7FFFFFFF`. Only maxhp is set; life follows through the callback.
  - Other stats, including 74 hpregen, go into the monster's stat list.
  - Emulated: v≤100 never wraps. v=150 wraps from 6,710,887 HP, v=200 from **5,592,406 HP**, and v=300 from 4,194,305 HP. Just above the threshold the result is ~1 HP (5.60M→22,784; 5.76M→502,784).
- Other PD life writes checked are safe: they use unsigned `cmova` clamps or double math. This covers boss phase code `0x102B1CFB/0x102B2B41/0x102B514E/0x102B5214`, heals `0x102F4C4D`, and CB `0x102AF962`.
  - Exceptions:
    - `0x102F2A69`, an item-event heal, uses a signed `cmovg`.
    - `0x102DD37D`/`0x102DD460` (PD spawner `0x102DCFB0`, Outer Void level 180 area logic) does `min(maxhp,0x7FFFFFFF)/100*150` with no output clamp. It wraps negative from 5,592,406 HP.
    - `0x102C7890` (special spawns) does `maxhp*2|*4` unclamped.
    - D2Game skill heal `0x6FCBCF92` does a signed `life+x > maxhp` check.
  - All three overflow only for units already near the ceiling.

## 3. Harness proof (VERIFIED, `/tmp/claude-0/bi/bi.c`, `drv.py`, `thr.py`)
The harness runs the real D2Game image. Stat calls are stubbed.
- Regen `0x6FC97CB0`:
  - life=maxhp=0x7FFFFF00 (spawn cap) with regen 0x7FFFF (R=1) → life **0x100 (1 HP)**.
  - life=maxhp=0x7FFFFFFF (PD clamp) with regen 1 or 0x100 → **1 HP**.
  - Pinned regen for R=1..4 (0x7FFFF…0x1FFFFF) → **1 HP**.
  - Controls: 8,380,423 HP with R=4 stays full; 1000/2000 HP heals normally.
- Overflow thresholds at spawn life = maxhp: R=1 8,386,561; R=2 8,384,514; R=3 8,382,469; R=4 8,380,424 HP.
- `hpmultiply` `0x6FC42FA0` (real MonUMod constants):
  - Hell unique 8,388,607 HP → maxhp=life=0xFFFFFDA4 (**−3 HP**).
  - Hell minion 5,592,406 → 0x800000EE.
  - First overflow: Hell unique/champion **4,194,305**, Hell minion **5,592,406**, Normal unique 2,097,153 HP.
- Spawn bonus via real muldiv `0x6FC214D0` reproduces the table below. For example, Lucion 2.4M: 1p 2.40M, 2p 4.08M, 3p 5.76M, 4p 7.44M, 5p+ capped at 8,388,607.

## 4. Bosses × players (Hell, MaxHP roll, DATA + formulas above)
- Columns: spawn life uses the ladder/L-HP columns (non-ladder differs only for level < 110).
- Player-count cells give "first player count that crosses" as non-ladder/ladder. "–" means never at 1–8 players; "n/a" means the scaler does not apply.
- Instant death still needs the trigger in §5.
| boss (hcIdx) | lvl | regen | Hell HP nonladder / ladder | spawn life 1..8 players (ladder cols, HP) | spawn cap (8,388,607) at | x1.5 pinnacle (n=0) hits 2^31 at | n=3 hits 2^31 at | unique/superunique +100% overflow (>=4,194,305) at |
|---|---|---|---|---|---|---|---|---|
| uberandariel (707) | 110 | 1 | 660,000 / 660,000 | 0.66M 1.12M 1.58M 2.05M 2.51M 2.97M 3.43M 3.89M | -/- | n/a | n/a | -/- |
| uberduriel (708) | 110 | 1 | 660,000 / 660,000 | 0.66M 1.12M 1.58M 2.05M 2.51M 2.97M 3.43M 3.89M | -/- | n/a | n/a | -/- |
| uberizual (706) | 110 | 1 | 660,000 / 660,000 | 0.66M 1.12M 1.58M 2.05M 2.51M 2.97M 3.43M 3.89M | -/- | n/a | n/a | -/- |
| ubermephisto (704) | 120 | 0 | 569,500 / 569,500 | 0.57M 0.97M 1.37M 1.77M 2.16M 2.56M 2.96M 3.36M | -/- | n/a | n/a | -/- |
| uberdiablo (705) | 120 | 0 | 642,700 / 642,700 | 0.64M 1.09M 1.54M 1.99M 2.44M 2.89M 3.34M 3.79M | -/- | n/a | n/a | -/- |
| uberbaal (709) | 120 | 0 | 633,600 / 633,600 | 0.63M 1.08M 1.52M 1.96M 2.41M 2.85M 3.29M 3.74M | -/- | n/a | n/a | -/- |
| uberdiablonew (789) | 110 | 0 | 1,053,000 / 1,053,000 | 1.05M 1.79M 2.53M 3.26M 4.00M 4.74M 5.48M 6.21M | -/- | 8/8 | 2/2 | 6/6 |
| diabloclone (333) | 110 | 2 | 642,700 / 642,700 | 0.64M 1.09M 1.54M 1.99M 2.44M 2.89M 3.34M 3.79M | -/- | n/a | n/a | -/- |
| rathmaBone (933) | 110 | 0 | 697,500 / 697,500 | 0.70M 1.19M 1.67M 2.16M 2.65M 3.14M 3.63M 4.12M | -/- | -/- | 3/3 | -/- |
| rathmaPoison (934) | 110 | 0 | 855,000 / 855,000 | 0.85M 1.45M 2.05M 2.65M 3.25M 3.85M 4.45M 5.04M | -/- | -/- | 2/2 | 7/7 |
| Lucion (1112) | 110 | 0 | 2,400,000 / 2,400,000 | 2.40M 4.08M 5.76M 7.44M 8.39M 8.39M 8.39M 8.39M | 5/5 | 3/3 | 2/2 | 3/3 |
| uberancientbarb1 (989) | 90 | 0 | 383,140 / 510,829 | 0.51M 0.87M 1.23M 1.58M 1.94M 2.30M 2.66M 3.01M | -/- | n/a | n/a | -/- |
| uberancientbarb2 (990) | 90 | 0 | 337,022 / 449,340 | 0.45M 0.76M 1.08M 1.39M 1.71M 2.02M 2.34M 2.65M | -/- | n/a | n/a | -/- |
| uberancientbarb3 (991) | 90 | 0 | 305,093 / 406,771 | 0.41M 0.69M 0.98M 1.26M 1.55M 1.83M 2.12M 2.40M | -/- | n/a | n/a | -/- |
| andariel (156) | 85 | 0 | 68,210 / 90,937 | 0.09M 0.15M 0.22M 0.28M 0.35M 0.41M 0.47M 0.54M | -/- | n/a | n/a | -/- |
| duriel (211) | 88 | 0 | 63,390 / 84,524 | 0.08M 0.14M 0.20M 0.26M 0.32M 0.38M 0.44M 0.50M | -/- | n/a | n/a | -/- |
| mephisto (242) | 87 | 0 | 70,740 / 94,320 | 0.09M 0.16M 0.23M 0.29M 0.36M 0.42M 0.49M 0.56M | -/- | n/a | n/a | -/- |
| diablo (243) | 94 | 0 | 135,325 / 180,425 | 0.18M 0.31M 0.43M 0.56M 0.69M 0.81M 0.94M 1.06M | -/- | n/a | n/a | -/- |
| baalcrab (544) | 99 | 0 | 370,275 / 493,701 | 0.49M 0.84M 1.18M 1.53M 1.88M 2.22M 2.57M 2.91M | -/- | n/a | n/a | -/- |
| willowispboss (743) | 92 | 4 | 838,400 / 1,117,920 | 1.12M 1.90M 2.68M 3.47M 4.25M 5.03M 5.81M 6.60M | -/- | n/a | n/a | 7/5 |
| KanemithBoss (1103) | 92 | 4 | 681,200 / 908,310 | 0.91M 1.54M 2.18M 2.82M 3.45M 4.09M 4.72M 5.36M | -/- | n/a | n/a | -/7 |
| RadamentBoss (1104) | 92 | 4 | 864,600 / 1,152,855 | 1.15M 1.96M 2.77M 3.57M 4.38M 5.19M 5.99M 6.80M | -/- | n/a | n/a | 7/5 |
| SharpToothBoss (1105) | 90 | 4 | 760,200 / 1,013,550 | 1.01M 1.72M 2.43M 3.14M 3.85M 4.56M 5.27M 5.98M | -/- | n/a | n/a | 8/6 |
| siegebeastMapBoss (1000) | 92 | 4 | 524,000 / 698,700 | 0.70M 1.19M 1.68M 2.17M 2.66M 3.14M 3.63M 4.12M | -/- | n/a | n/a | -/- |
| CityofUrehBoss (1174) | 91 | 1 | 773,100 / 1,030,800 | 1.03M 1.75M 2.47M 3.20M 3.92M 4.64M 5.36M 6.08M | -/- | n/a | n/a | 8/6 |
| DjinnDarkBoss (1196) | 90 | 2 | 481,460 / 641,915 | 0.64M 1.09M 1.54M 1.99M 2.44M 2.89M 3.34M 3.79M | -/- | n/a | n/a | -/- |
| ImperialPalaceBoss (1166) | 91 | 0 | 425,205 / 566,940 | 0.57M 0.96M 1.36M 1.76M 2.15M 2.55M 2.95M 3.34M | -/- | n/a | n/a | -/- |
| KyovoshadBoss (1192) | 91 | 2 | 412,320 / 549,760 | 0.55M 0.93M 1.32M 1.70M 2.09M 2.47M 2.86M 3.24M | -/- | n/a | n/a | -/- |
| NaKrulBoss (1194) | 91 | 2 | 360,780 / 481,040 | 0.48M 0.82M 1.15M 1.49M 1.83M 2.16M 2.50M 2.84M | -/- | n/a | n/a | -/- |
| Iskatu (1094) | 90 | 0 | 354,760 / 472,990 | 0.47M 0.80M 1.14M 1.47M 1.80M 2.13M 2.46M 2.79M | -/- | n/a | n/a | -/- |
| DemonRoadBoss (1121) | 90 | 2 | 354,760 / 472,990 | 0.47M 0.80M 1.14M 1.47M 1.80M 2.13M 2.46M 2.79M | -/- | n/a | n/a | -/- |
| SkovosBoss (1122) | 90 | 2 | 347,158 / 462,854 | 0.46M 0.79M 1.11M 1.43M 1.76M 2.08M 2.41M 2.73M | -/- | n/a | n/a | -/- |
| leoricMapBoss (998) | 89 | 0 | 336,285 / 448,335 | 0.45M 0.76M 1.08M 1.39M 1.70M 2.02M 2.33M 2.65M | -/- | n/a | n/a | -/- |
| ZharTheMad (1058) | 91 | 0 | 335,010 / 446,680 | 0.45M 0.76M 1.07M 1.38M 1.70M 2.01M 2.32M 2.64M | -/- | n/a | n/a | -/- |
| CowBoss (826) | 91 | 0 | 360,780 / 481,040 | 0.48M 0.82M 1.15M 1.49M 1.83M 2.16M 2.50M 2.84M | -/- | n/a | n/a | -/- |

Map bosses need extra map or zone "+monster life" to reach the regen-overflow ceiling. All of them are MonStats `boss`, and several have `DamageRegen` > 0. The table gives the life % needed, as one entry (entries compound), at 1..8 players, ladder:

| boss | DamageRegen | ladder HP | +life% needed at 1..8 players |
|---|---|---|---|
| RadamentBoss | 4 | 1,152,855 | 627 328 203 134 91 62 40 23 |
| willowispboss (superunique "Wisp Boss") | 4 | 1,117,920 | 650 341 212 142 97 67 44 27 |
| SharpToothBoss | 4 | 1,013,550 | 727 386 245 167 118 84 59 40 |
| CityofUrehBoss | 1 | 1,030,800 | 714 379 239 162 114 81 56 38 |
| KanemithBoss | 4 | 908,310 | 823 443 284 198 143 105 77 56 |
| siegebeastMapBoss | 4 | 698,700 | 1099 … 131 103 |
| DjinnDark / Kyovoshad / NaKrul / DemonRoad / Skovos | 2 | 463k–642k | ≥121% even at 8p |
| diabloclone (if spawned in a modded map level) | 2 | 642,700 | 121% at 8p |
| uber Lilith/Duriel/Izual | 1 | 660,000 | not reachable: levels 133–135 are not map levels, so no map mods and no pinnacle scaler |

## 5. Exact instant-death conditions
- **A. Regen overflow → 1 HP** (VERIFIED mechanism). Needs life + stat74 ≥ 2^31, i.e. life ≳ 8,380,000–8,386,560 HP depending on regen, **and** stat74 > 0.
  - **Pinnacle bosses in map arenas.** The PD scaler pins maxhp=life=0x7FFFFFFF:
    - Lucion (arena 188/189): ≥3 players at n=0, ≥2 at n=3.
    - Uber Diablo new (137): 8 players at n=0, ≥2 at n=3.
    - Rathma (161–163): ≥2–3 players at n≥1.
    - Their DamageRegen is 0, so they die only when regen comes from elsewhere. The realistic source is the `map_mon_hpregen` affix, or any hpregen stat in the game's map list at game+0x1DF8 when the arena applies map mods.
    - Rathma's golems/minions in the set (937/938: R=3) regenerate by default, but their HP is small.
  - **Map bosses with DamageRegen > 0** (Radament/Kanemith/SharpTooth/Wisp/Ureh/Siege…) are pushed to the ceiling by map `+% monster life` or zone +100/+200% life. The 0x7FFFFFFF clamp plus the recomputed regen (≥524,287 raw) then kills on the first regen tick.
    - Example: RadamentBoss at 8 players, ladder, with +23% monster life. Or 4–5 players with the +100% zone entry.
- **B. Negative life at spawn** (VERIFIED arithmetic). A Hell unique, superunique or champion whose pre-umod life (after players and map life%) is ≥4,194,305 HP; minions ≥5,592,406.
  - Wisp Boss (superunique row 67): ≥5 players ladder / ≥7 players non-ladder, with no map mods; fewer with map life%.
  - Any PD map boss spawned as unique/champion (e.g. by PD's wave spawner `0x102B63E9/0x102B6503`) above that life also qualifies.
- **C. Map/zone maxhp% wrap** (READ). The +200% zone entry (or a stacked entry >100%) on a monster with 5.59M–~5.8M HP before that entry leaves ~1–600k HP.
  - Example: Lucion with 3 players (5.76M) plus the +200% zone modifier → ~0.50M (×1.5 pinnacle → 0.75M).
- **Not a cause:**
  - Uber Tristram/Pandemonium ubers and act bosses: max 3.9M at 8p, no PD scaler outside map levels.
  - Vanilla DClone outside modded maps.
  - Players joining after spawn: no rescale exists.
  - Crushing blow (PD, double math).
  - #11027 hp% (uses HP units, max 839M).

## 6. Why it is intermittent
- It depends on the number of living players in the whole game at the moment the boss room activates, not who enters the arena. Uniques also get a random MinHP–MaxHP roll.
- It depends on the map/zone mods active for that game (`+% life`, `hpregen`, zone types 4/14). The list is rebuilt per map item.
- It depends on the pinnacle empower counter n (game+0x2629) and on ladder L-HP versus non-ladder HP.
- Champion/unique rolls vary.
- The fatal regen tick runs every regen interval, so the boss silently drops to 1 HP right after spawn or after a heal. Players see it die to the first hit.
- Side effect: the same tick computes the HP-bar byte as `life<<7 / maxhp` (`0x6FC97D4C`), which overflows for >65,536 HP. That is display only, but it makes big-HP bars erratic.

## 7. Fixes
1. **Regen** `0x6FC97CDC`: saturate by testing `if (regen > maxhp - life) life = maxhp` (unsigned) instead of `add edi,ebx; cmp edi,eax; jle`. Or do the add in 64-bit.
2. **Headroom:** clamp every PD maxhp result to e.g. 0x7F000000 (8,323,072 HP), not 0x7FFFFFFF. Sites: `0x102D2C3F`, `0x102DBCE7`, and add an output clamp after `0x102DD378`/`0x102DD45B`. Or cap boss HP at ~8.0M in data.
3. **Map maxhp%** `0x102DBCD0`: use the 64-bit mul already used in `0x102D2AF0` before clamping.
4. **hpmultiply** `0x6FC41F45`: clamp the sum to ≤0x7FFFFF00, e.g. with a PD hook on the SetStat at `0x6FC41F4D`.
5. **Generic guard:** hook SetStat for stats 6/7 on monsters and clamp to [0, 0x7F000000].
6. **Longer term:** scale pinnacle toughness with damage reduction instead of life beyond 8.3M.

## Files
- Harness: `/tmp/claude-0/bi/bi.c` (build: `gcc -m32 -nostdlib -static -O1 -fno-pie -no-pie -ffreestanding -fno-stack-protector`, run from `harness/`), `drv.py`, `thr.py`, `table.py`.

## 8. Single player /players
**1. game+0x6A in single player is 3, so `/players` is honoured.** READ, traced end to end.
- **Server side.**
  - `0x6FC4C3BD` stores arg3 of the game-create function `0x6FC4C2D0`.
  - Its only caller is `0x6FCEAF9C`, the handler for internal client message 0x67. It passes byte **P[0x11]** of that message.
- **Client side.** D2Client builds message 0x67 at `0x6FAC4C20`. `P[0x11]` comes from the client connection type `0x6FBCC394`, which is set from the launch struct at `0x6FAF1F8C`:
  - type 0 → **3**
  - type 6 → 1
  - type 8 → 2
  - any other type → 0
- **Which type is single player.** Types 0 and 1 are the in-process local server: `0x6FAF4CF5` calls D2Game #10040 and #10008 directly. Type 0 is the single-player case; that label is inferred from this behaviour.
- **The `/players` command.** The client parser `0x6FB20A5F` accepts it only for types 0, 1, 6 and 8, clamps it to 2..8, and calls D2Game #10049 `0x6FC57430`, which writes `0x6FD31C1C`.
- **The player count.** `0x6FC579A0` returns max(living players in the game, `/players` value) when 1 ≤ game+0x6A ≤ 3.
  - Single player (3) and TCP/IP host (1 or 2) use `/players`.
  - Type 1 (0) does not.
  - Realm games get the byte from the realm server.
- **HP columns.** Because game+0x6A ≠ 0, #11089 is called with the L-flag (`0x6FCCFFFB`), so **single player uses the L-HP columns**, the same numbers as the "ladder" figures above. This corrects the OPEN note in FINDINGS.md.

**2. Which player count each life path uses.**
| path | player count used |
|---|---|
| spawn +% (`0x6FCCF840` → `0x6FC579A0`) | living players or `/players` (single player: `/players`). Bosses have MonStats byte +0x4C = 0; townsfolk (1) and cows/camels (2) are forced to 1 |
| PD2 pinnacle scaler `0x102D2AF0` | none. It uses only game+0x2629 (empower n) and the class set; it multiplies the already player-scaled maxhp |
| MonUMod hpmultiply `0x6FC42FA0` / `0x6FC41EC0` | none. It uses difficulty and unique/champion/minion; it multiplies the player-scaled maxhp |
| map/zone life % `0x102DBA50` | none. It uses the map item's stats; it multiplies the player-scaled maxhp |
| regen tick `0x6FC97CB0` | none |

So `/players N` in single player is exactly equivalent to N players online for all of these paths.

**3. Single player, `/players 8`, Hell, no map/zone mods, n = 0.**
- **Ubers.**
  - Lilith, Uber Duriel and Uber Izual reach at most 3.89M HP (DamageRegen 1). The Uber Tristram ubers reach at most 3.79M (DamageRegen 0). DClone reaches 3.79M.
  - They are nowhere near either threshold. **No death.**
- **Act bosses.** At most 2.9M (Baal). **No death.**
- **Lucion** (from `/players 3` up) and **Uber Diablo new** (`/players 8`) are pinned at exactly 0x7FFFFFFF.
  - Both have DamageRegen 0 and nothing else heals them, so they stay at full ceiling HP. **No death.**
  - Rathma: 5.04M × 1.5 = 7.57M, not pinned.
- **Map bosses without map mods.** At most 6.80M (Radament, L-HP) with DamageRegen 4. That is below 8,380,424, so **no death.**
- **Exception: the Wisp Boss superunique** (SuperUniques row 67, `willowispboss`).
  - Its superunique +100% Hell bonus wraps negative from **`/players 5`**.
  - Harness, real `0x6FC42FA0`: p5 4,248,096 → −8,281,025 HP; p6 −6,715,937; p7 −5,150,849; p8 −3,585,761; p4 stays positive at 6,931,103.
  - It dies on the first damage event.

**Verdict.** The player's observation matches the code for the ubers, act bosses and pinnacle bosses. With no map mods and n = 0, none of them can reach 1 HP or negative life in single player; they need an outside regen source or a +life map/zone mod.

**Single-player recipes that should reproduce it.** Run in the harness, VERIFIED on the arithmetic.
1. **Wisp Boss.** Enter the level that holds the Wisp Boss with `/players 5`–`8` **before** its room is activated. It dies to the first hit.
2. **RadamentBoss** (DamageRegen 4). Use `/players 8` and a map with **≥ +24% monster maximum life** (`map_mon_maxhp_percent`).
   - Harness: +23% → 8,366,268 HP (survives); +24% → clamped at 0x7FFFFFFF, regen 2,097,151 → **life 0x100 = 1 HP** at the first regen tick.
   - Fewer players need more life %. See the §4 table: the same approach works for Kanemith, SharpTooth, Ureh and Siege with the listed percentages.
3. **Lucion.** Use `/players ≥3` (n = 0) so it is pinned, plus a "monsters regenerate" (`map_mon_hpregen`) affix in the game's map list.
   - This works only if the arena applies map mods (the level +0x4C gate at `0x102DC018`, not resolved).
   - Treat this one as likely, not proven.

`/players` must be set **before** the boss spawns. Changing it afterwards has no effect, because monster HP is never rescaled.

## 9. What can wrap Diablo Clone
Diablo Clone is `uberdiablonew`, hcIdx 789. Its data: DamageRegen 0, Drain(H) 5, no MonProp rows.
- Skills: UberDiabWall/Cold/Fire/Light/SuperFire/Summon/Run and MonTeleport.
- None of those skills has an aura/passive stat or heals (Skills.txt, DATA).
- Lucion and Rathma are the same: no life-returning skill, DamageRegen 0, Drain 5.

**Spawn and ceiling (READ).**
- Diablo Clone spawns in level **137 "UberDiabloLvl"**. That is inside the 137–201 map-level range (`0x102CE890`), so both PD post-spawn steps apply.
- **Map/zone list.**
  - `0x102DBFE0` applies game+0x1DF8 to every monster in a map level whose MonStats byte +0x4C is 0 (`0x102DC018`). Diablo Clone, Lucion and Rathma are all 0.
  - There is no uber exclusion. The only guard is state 0xC6, which stops a second application.
  - The list persists for the game. It is (re)built when a map is opened (handler `0x102D4820`, registered at `0x102BE30A`) and by the zone builder `0x102DC520`. It is empty in a fresh game.
- **Pinnacle scaler.** `0x102D2AF0` pins maxhp = life = 0x7FFFFFFF when:
  - players ≥ 8 at n = 0, or
  - players ≥ 2 at n ≥ 1 (§4).
  - Main bosses also get absorb % = 7n + 2t for fire, light, cold and magic (stats 142/144/146/148, `0x102D2C73`).

**Every path that adds life, and whether it wraps.** Harness `/tmp/claude-0/bi/heal.c` runs the real `0x6FCFB570` with PD's SetStat hook `0x10268600`.
| path | code | math | at life = 0x7FFFFFFF |
|---|---|---|---|
| regen tick | `0x6FC97CB0` | `life+regen` signed, then `<0x100 → 0x100` | **1 HP** (VERIFIED). Needs stat 74 > 0: DamageRegen is 0, so only `map_mon_hpregen` (402→74) from the game's map list can supply it |
| defender heal from **elemental absorb** | PD `0x1026F820` adds absorbed amount to dmg+0x48; `0x6FCFE23D` → `0x6FCFB570` | `lea eax,[heal+life]; cmp eax,maxhp; jl` signed; PD wrapper does not block a negative result (stat 488 test is on ratios) | wraps negative (VERIFIED: 0x7FFFFFFF + 700 HP → −8,387,909 HP). The same hit's subtraction `0x6FCFE24E` runs on the wrapped value; heal ≤ that hit's damage, so the modular result equals life+heal−dmg and the boss **survives with ≈ full life** (VERIFIED: after a 1000 HP hit → 8,388,307 HP). Net effect is harmless |
| attacker **leech** (Diablo Clone hitting players, mercs or summons) | PD `0x102700F0` → stock `0x6FCFBA40` → `0x6FCFB570` | same signed add | wraps negative by g (the leech gain) and **stays negative** until the next damage event. The next hit with raw damage ≤ g−1 leaves life < 0x100 → **dies**; any larger hit wraps back to ≈ full. Needs stat 60 on Diablo Clone, which only comes from `map_mon_lifedrainmindam` (403→60). g = trunc(5% × L% × phys) (Drain(H) 5), a small number, so death is possible but unlikely |
| maxhp change → callback | `0x6FCF957D` | double math, `life/old*new`, clamped [1,new] | safe |
| PD boss phase / heal scripts | `0x102B1CFB`, `0x102B2B41/2C93/2F78`, `0x102B514E`, `0x102B5214`, `0x102F4C4D` | double via `0x102CE840`, or unsigned `cmova` | safe |
| other heals | `0x6FCBCF92` (skill heal), `0x102F2A69` (item-event heal) | signed | not used by Diablo Clone's skills or items |
| shrines, potions | – | player-only | n/a |
| life% | #11027 `0x6FD82330` (HP units ≤ 839M), HP-bar byte `life<<7/maxhp` (`0x6FC97080`, `0x6FC97860`, `0x6FC97D4C`) | 32-bit | bar byte overflows for > 65,536 HP. **Display only** (erratic bar); never writes life |

**Correction to §5.** A negative life does not always die on the next hit. `0x6FCFE24E` subtracts the hit's total damage from the wrapped value.
- If life = 0x80000000 + x, a hit ≤ x kills it. A larger hit wraps life back to a positive value.
- The unique/superunique case (x ≈ 0.1–4.8M HP) and the regen 1-HP case still die to ordinary hits.

**Conclusion for Diablo Clone in a group game.**
- **Most likely (VERIFIED mechanism, READ reachability).** Diablo Clone pinned at 0x7FFFFFFF (8 players, or ≥2 players after an empower use) in a game whose map list carries a **monsters-regenerate** affix. The first regen tick sets it to **1 HP**, and the next hit kills it.
- **Possible but rare.** A **life-drain** affix in the map list lets Diablo Clone leech while pinned. It dies if the next incoming hit is smaller than its leech gain.
- **Not fatal.**
  - Elemental absorb (from empower or map absorb affixes) wraps life but is cancelled within the same hit.
- **No life gain possible.**
  - There is no regen or drain source in a fresh game, so Diablo Clone cannot gain life.
  - Its skills and PD scripts cannot wrap it.

## 10. Fresh 8-player game, no maps: remaining paths
Setup: online, 8 players, empty map list, n = 0. The pinnacle scaler pins Diablo Clone at maxhp = life = 0x7FFFFFFF (§2).
- Realm games get game+0x6A from the server. Level 110 HP(H) equals L-HP(H), so the value is the same either way.

**1. The pinnacle scaler and the callback it triggers.** `/tmp/claude-0/bi/fpu.c` runs the real D2Game `_ftol` `0x6FD15724` on the callback's exact x87 sequence.
- The scaler's own math is SSE2/64-bit and clamped (`0x102D2BF3..0x102D2C4F`). It runs once per unit: it is gated by state 0xF1 at `0x102C8413` and called only from the SpawnMonster hook.
- Nothing re-runs spawn life, the scaler, or a player-count rescale. Stat 100 is read only by crushing blow; there is no join/leave hook and no portal handler touching life.
- The scaler's `SetStat(7, 0x7FFFFFFF)` fires the vanilla maxhp callback `0x6FCF957D`. That callback computes **life = ftol((x87) life/old × new)**, then clamps it to [1, new]. Harness results:
| x87 precision control | spawn life = old = 6,212,700 HP, new = 0x7FFFFFFF | at spawn cap, old = 0x7FFFFF00 |
|---|---|---|
| 64-bit (0x37F) | 0x7FFFFFFF ✓ | 0x7FFFFFFF ✓ |
| 53-bit (0x27F, MSVC/Windows default for new threads) | 0x7FFFFFFF ✓ | 0x7FFFFFFF ✓ |
| **24-bit (0x07F, single precision; what Direct3D sets on its thread unless FPU_PRESERVE)** | ftol → 0x80000000 → clamp → **life = 1 (1/256 HP)** | **life = 1** |
- In single precision, 1.0 × 2147483647 rounds to 2^31, and `_ftol` returns 0x80000000. The callback's `max(1, x)` then leaves life at 1/256 HP, and **the first hit kills it**.
  - D2Game, D2Common and PD contain no `fldcw` that selects single precision; only the CRT sets 64-bit temporarily. So this happens **only if the server thread runs with precision control 24**, which depends on the host process or another loaded module.
- The same applies to every unit PD clamps to 0x7FFFFFFF, i.e. Lucion (≥3p) and Rathma once empowered. It never happens with 53/64-bit precision.
- In single player, the game-server thread is a separate thread, whose FPU state starts at the Windows default (53-bit). That fits "does not reproduce offline".

**2. Life or maxhp increases on a monster in a fresh game.** All of these were checked:
- **Regen.** Stat 74 is 0: DamageRegen is 0, and the callback recomputes regen only when DamageRegen ≠ 0 (`0x6FCF9618`). There is no other base, out-of-combat or leash regen in D2Game or PD. PD's regen hook `0x102689B0` only caps regen in PvP.
- **Skills.** Diablo Clone's skills (§9) and those of its summons have no heal, aura stat or passive.
- **Player auras and heals** (Prayer/Meditation/party heals) target the party only. **Shrines and potions** affect players only.
- **Leech.** It needs stat 60, which Diablo Clone has 0 of: no MonProp row and no skill stat.
- **Absorb.**
  - Scaler absorb is 7n + 2t with t = byte game+0x2628 − 1, so it is non-zero only if that per-game field is set.
  - The heal wraps but cancels within the same hit (§9, VERIFIED). Absorb % is capped at 40, so the net change is life − D + 2A ≤ life.
- **PD boss scripts and heals**: SSE double math or unsigned `cmova`, so safe.
- **Event buff `0x102B6E8D` (NEW, READ).** For every monster in the level, PD does `maxhp += (maxhp/100)·pct` in 32 bits with **no clamp**, then SetStat 7.
  - On a pinned unit this wraps maxhp negative. The callback then sets life = new (negative), so the boss dies to the first hit smaller than ≈ 83,886·pct HP.
  - It runs only when game+0x1DF4 == 0x2C, a map/event type set by the map-open handler `0x102D4B4C`. So it does not apply to a fresh game.

**3. 32-bit life% and "life combined with other values".** All safe:
- Crushing blow (PD `0x102AF610`) does everything in SSE doubles. Life is read as unsigned double, and prime-evil hp% comes from #11027.
- #11027 uses HP units: 8,388,607·100 < 2^31.
- Open wounds uses a fixed amount.
- Blood Warp `0x102FC440` uses doubles, on the caster.
- Rathma share `0x1026F5F5` uses `cmovns`, clamped to 0, and applies to Rathma only.
- The HP-bar byte (`life<<7/maxhp`) is display only.

**4. The damage function at 0x7FFFFFFF.** The damage clamp `0x6FCFE205`, the subtraction `0x6FCFE24E` and the leech/absorb heals are all safe for a positive, pinned life (VERIFIED §9). There is no %-of-maxhp damage or execute effect in PD against uniques.

**Conclusion.**
- The only mechanism found that kills a pinned Diablo Clone with no map mods and no regen or drain is the x87 rescale in the maxhp callback. It turns 1.0 × 0x7FFFFFFF into 0x80000000 and so sets life to 1/256 HP. The trigger is simply ≥8 players plus the pinnacle ×1.5 (Lucion: ≥3 players).
- **Confidence:**
  - VERIFIED that the code produces 1/256 HP in single-precision mode and is safe in 53/64-bit.
  - UNVERIFIED that the realm server's game threads run in single precision. The realm may also run different server code.
- Fix either way: clamp PD's maxhp results to ≤ 0x7F000000, which also leaves headroom for §5.
  - For this path, even 0x7FFFFF00 rounds correctly: `fpu.c` shows 0x7FFFFF00 → 0x7FFFFF00 in all three precision modes. 0x7FFFFFFF alone rounds up.
  - Alternatively, compute the callback's rescale in integer/64-bit.

## 11. Diablo Clone life gain at fight start, by tier
Scope: `uberdiablonew` (789) in level 137, Hell. "Tier" n = game byte +0x2629. Harness: `/tmp/claude-0/dc/abs.c` + `drv.py`. It runs the real PD absorb `0x1026F820`, the real D2Game heal `0x6FCFB570` with PD's SetStat hook `0x10268600`, and the damage-apply tail `0x6FCFE242..0x6FCFE2CE`, emulated.

**Corrections to §9/§10.**
- §9/§10 said Diablo Clone has no MonProp rows. That is wrong: `MonProp.txt` row `uberdiablonew` has three Hell rows. They match `excel_live` (DATA).
  - `curse-res` 95
  - `res-pois-len` 50
  - **`abs-fire` 8**, which is stat **143 `item_absorbfire` (flat)** per Properties.txt
  - D2Game applies these at spawn (`0x6FCD024B..0x6FCD02F2`, chance 100, not patched by PD; READ).
- §9 said the absorb heal is harmless. **It is not** for small fire hits (see 11.3).

### 11.1 Where the tier comes from (READ)
- **Portal open.** `0x102D4AAA` sets +0x2629 = item stat 185 `uber_difficulty` of the `dcma` item.
  - CubeMain rows `dcma difficulty 0 -> 1`, `1 -> 2`, `2 -> 0` (mods +1, +1, −2) cycle it. So **only tiers 0, 1 and 2 exist for the Clone** (DATA).
  - The same handler sets game type +0x1DF4 = 0x2A.
  - It sets +0x2628 = P, the value returned by `0x102DB680`. That function teleports the opener plus eligible players standing near the town spot (0x13D2, 0x13FA; dist² < 150), and P counts those that arrive in 137.
- **The increment at `0x102B8FF0`** is not a Clone path. It sits in the AI function `0x102B77B0` (AI table slot +0x788, installed at `0x102BABE3`), in the branch for **monster class 0x49E = hcIdx 1182 `InvaderPortal`**.
  - That branch runs only in game type **0x14** (an invasion event). It needs phase +0x262A < 5, n ≤ 2 and +0x2628 ≤ 3.
  - It spawns InvaderNecromancer/Paladin/Sorceress (0x4A3–0x4A5) and advances the phase.
  - It then raises n when (n=0 and phase ≥ 1), (n=1 and phase ≥ 3) or phase ≥ 5, so n can reach 3.
  - The Clone portal overwrites +0x2629 and sets type 0x2A, so tier 3 cannot apply to a Clone spawned from `dcma`. **Tier 3 is theoretical** for 789.

### 11.2 What each tier gives Diablo Clone (READ unless noted)
Every reader of +0x2629:

| reader | what it does for 789 |
|---|---|
| `0x102C8406` → scaler `0x102D2AF0` (once per unit, gated by state 241 `uberscaling`) | maxhp: see the table below. It also attaches a stat list to the Clone: <br>• 127 `item_allskills` +4n <br>• main-boss block, active at **every** tier: fire/cold/light absorb % (142/148/144) = **7n + 2t**; magic absorb % (146) = **3n + 2t**; phys DR % (36) = **3t**; poison-length resist (110) = 3n + t, on top of MonProp 50 |
| Clone AI `0x102B4F73` | AI delay = max(25, MonStats byte +0x51 − 8n + 8). Skill-use roll `rand(100) < word+0x84 + 8n` (+25 when no target is near) (`0x102B50EF`, `0x102B54CC`, `0x102B55F1`) |
| soul AI `0x102B5C96` (790–794) | n>0: delay = max(20, byte+0x51 − 4n + 4), else byte+0x51 + 10. Cast gate word+0x78 − 5n + 5 (min 5). Count gate word+0x6C + 12(n−1). No life effect on the Clone (see 11.4) |
| death `0x102C194C`, `0x102C1A10`, `0x102C1A1F` | n>0: 1/200 bonus drop. `0x102CFA20(n)`. n==2: extra counters +0x383/+0x3FF and flag bit 0 at +0x375. Kill time only |
| `0x102F4485`, `0x102C9322`, `0x102C1E74`, `0x102C1FA6/1FB5`, `0x102C2706`, `0x102C2827/2836` | reward and drop logic for other events or kills. No life |
| `0x102B25B8`, `0x102B7896`, `0x102B8529`, `0x102B8905`, `0x102B8CA2` | Rathma / Lucion / other AI. Not 789 |
| `0x102F4D4E` | Rathma-level branch of `0x102F4670`. Not 789 |

- **t = max(+0x2628 − 1, 0) at spawn** (`0x102D2BAA`). After the altar, +0x2628 = P (players teleported), so normally **t = P − 1 (0..7)**.
  - `0x102F4670` increments +0x2628 once per player moved (`0x102F4C00`) before the count overwrites it. If the Clone's room activates inside the altar loop, t can be higher (not resolved).
  - So even tier 0 with 8 players gives **14 % fire/cold/light/magic absorb and 21 % physical DR**.
- **Empowered absorb % (cap 40 at use, `0x1026F846`).**

| n | fire / cold / light % | magic % |
|---|---|---|
| 0 | 2t | 2t |
| 1 | 7+2t | 3+2t |
| 2 | 14+2t | 6+2t |
| 3 | 21+2t | 9+2t |

  - The fire rows also get the flat 8 HP from MonProp.
- **maxhp after the scaler.** Emulated with IEEE doubles, the same as SSE2. Spawn life is 1.053M × player%, L-HP. "CEIL" means pinned at 0x7FFFFFFF.

| players | n=0 | n=1 | n=2 | n=3 (theoretical) |
|---|---|---|---|---|
| 1 | 1.58M | 5.05M | 5.90M | 6.74M |
| 2 | 2.69M | CEIL | CEIL | CEIL |
| 3 | 3.79M | CEIL | **7.44M** | **6.11M** |
| 4–7 | 4.90M–8.21M | CEIL | CEIL | CEIL |
| 8 | CEIL | CEIL | CEIL | CEIL |

  - **New scaler bug (READ + emulated).** For n>0 the code computes `X = k·maxhp` in 64 bits, then adds `n·trunc(int32(low32(X))·20/100)`. The low half is read as **signed** (`cvtdq2pd` in `0x102CE840`).
    - When 2^31 ≤ X < 2^32, the bonus is negative. For the Clone that is 3–5 players.
    - At 3 players, tier 2 then has *less* HP than tier 1 (7.44M vs pinned). §2/§4 said "≥2 players at n≥1 → pinned"; that is not exact at 3 players.

### 11.3 Everything that can add life to it at fight start
| source | present at tier | math | effect at full / pinned life | confidence |
|---|---|---|---|---|
| **Flat fire absorb 8** (MonProp, stat 143) | 0, 1, 2 | `0x1026F88B`: `A = min(fire, 8<<8)` added to the heal field dmg+0x48 after resistance | see the heal-only rule below | **VERIFIED** |
| **Empowered absorb %** (7n+2t fire/cold/light, 3n+2t magic, cap 40) | 0 (t>0), 1, 2 | `0x1026F862`: `A = trunc(E·min(p,40)/100)` | for cold, light and magic alone, A ≤ 0.4E < remaining 0.6E, so the hit never nets a heal | **VERIFIED** (40 % cold 1000 HP, 60→40 % magic) |
| Clone AI: player count in level drops (death, TP out, leave) | all | `0x102B511B..0x102B514E`: `life = min(life + maxhp/100·20, maxhp)` with **unsigned** `cmova` | +20 % maxhp heal. Sum ≤ 1.2·0x7FFFFFFF < 2^32, so it cannot wrap. The first AI tick only stores the count | READ |
| Clone AI phase floors | all | `0x102B51CB..0x102B5214`: phase stat 447 ∈ {2,4,6} and life% < 75/50/25 → life = maxhp·(100 − 25·phase/2)/100 (double) | raises life to 75/50/25 %. Not at fight start | READ |
| Altar teleport `0x102F4670` → `0x102F4C00` (dest 0x89 or 0xBC) | all | +0x2628++ and the boss at **+0x2600** gets `min(life + 20 %, maxhp)` (unsigned) | at open, +0x2600 is the previous event's boss (stale pointer, possibly freed) or null, so this Clone gains nothing. A **second `dcma` opened mid-fight** heals the live Clone 20 % per player moved and rewrites n and t | READ |
| regen (stat 74) | – | DamageRegen 0 | only from a map list at game+0x1DF8 left by an earlier map in the same game (§9) | DATA / READ |
| leech (stat 60) | – | Drain(H) 5 but stat 60 = 0 | only from the map list (§9) | DATA |
| skills | – | Skill1–8 are all level 1: UberDiabWall 166→PD `0x102FD320`; UberDiabCold 100; UberDiabFire 22; UberDiabLight 152; UberDiabSuperFire 22; UberDiabSummon 177→PD `0x102FE760`; UberDiabRun 103; MonTeleport 98 | no aurastat/passivestat. The two PD do-funcs contain no SetStat calls. The stock do-funcs are unpatched | DATA / READ |
| minions and summons | – | dclone skeletons, mages and archers (886–891): SkeletonRaise, no stats. Bloodlord 892: `BloodLordFrenzy` aura, which gives speed only. Souls 790–794: Meteor/Boulder fire | no heal aura. The souls' missiles are monster-side (Align blank), so they should not hit the Clone (stock collision rules, not traced) | DATA |
| states at spawn | – | 241 `uberscaling` (the scaler's stat list); 198 `map` (map-mod gate) | no life stats | DATA |

**The heal-only rule (VERIFIED).**
- **How a hit is applied.** `0x6FCFE237` heals life by A with a signed add: `lea eax,[heal+life]; cmp eax,maxhp; jl`. PD's hook lets a negative value through.
- **Remaining damage.** `0x6FCFE242` subtracts the rest only if `total > 0`, then applies `<0x100 → 0`. `0x6FCFE2C7` flags the kill when `life ≤ 0`, via flag bit 0x2 (`0x6FCFE33E` → events 0xA/9).
- **So with life + A ≥ 2^31 and A > D'** (D' = the hit's remaining damage), the Clone **dies to that hit**. D' = 0 is the heal-only case.
- **Harness results at life = maxhp = 0x7FFFFFFF:**

| fire hit (post-res) | absorb | heal | remaining | result |
|---|---|---|---|---|
| 5 HP | flat 8 | 5 | 0 | life 0x800004FF, **KILLED** |
| 12 HP | flat 8 | 8 | 4 | **KILLED** |
| 15.996 HP | flat 8 | 8 | 7.996 | **KILLED** |
| 16 HP | flat 8 | 8 | 8 | alive, full |
| 12 fire + 100 phys | flat 8 | 8 | 104 | alive, 8,388,512 |
| 18 HP | 7 % + 8 (n=1) | 9.26 | 8.74 | **KILLED** |
| 19 HP | 7 % + 8 (n=1) | – | – | alive |
| 22 HP | 14 % + 8 | – | – | **KILLED** |
| 23 HP | 14 % + 8 | – | – | alive |
| 27 HP | 21 % + 8 | – | – | **KILLED** |
| 28 HP | 21 % + 8 | – | – | alive |
| 79 HP | 40 % + 8 | – | – | **KILLED** |
| 81 HP | 40 % + 8 | – | – | alive |

- **Controls:**
  - Not pinned (6,212,700 HP, full): the heal is capped at maxhp, so there is no gain.
  - 256 HP below the ceiling: +5 HP, fine.
  - 4 HP below the ceiling: **KILLED**.
  - Any unit sitting at the spawn cap 0x7FFFFF00 is killed by a 5 HP fire hit.
- **Rule.** A fire-dominated hit whose post-resistance fire E satisfies **E < 16/(1−2p) HP**, with small other damage, nets a heal.
  - p = min(40, 7n+2t)/100.
  - Thresholds: 16 HP (n=0, t=0); 18.6 (n=1); 22.2 (n=2, or n=0 with 8 players); 27.6; 36.4; 53.3; up to 80 HP at the 40 % cap.
- **Pinned Clone** (8p at n=0; 2 or 4–8p at n=1–2): **the first such hit kills it outright.**
- **Not pinned:** each such hit heals it by up to A − D' (≤ 8 HP + % part) until it reaches maxhp.

**Which player damage does this (DATA).**
- **Fire-only small hits:**
  - per-frame ground fire: Blaze (HitShift 2), Firestorm (2), Fire Wall (3), Inferno (5), and low-level or unsynergised versions of them
  - **Holy Fire aura** (HitShift 6). Dragon and Hand of Justice give level 12: listed 7–11 HP per pulse, about 5–8 HP after the Clone's 30 % fire resist, which is **at or under the flat 8 → heal-only**. This is an estimate: the aura damage path was not traced.
  - weak Fire Bolt or charges
- **Cold, lightning and magic alone can never net a heal** (the % cap is 40, so absorbed < remaining).
- Hits with a real physical part (melee, Enchant, fire-plus-physical arrows) keep D' ≫ A and are safe.
- Poison and physical have no absorb entry (`0x6FD22AB0` table).

### 11.4 Player-side effects that could heal a monster (READ)
- **No player effect heals a monster.** No negative damage exists:
  - The resist multiplier is (100 − min(res,100)) ≥ 0 (`0x1026F4A9`).
  - Results are floored at 0 (`0x1026F4E4`).
  - The absorb % cap is 40 (`0x1026F846`).
- Life Tap and leech heal the attacker only. Prayer, Meditation and the other auras affect the owner and party only (auras.md); hostile targets get only aurastats (curses and -res). Static Field skips 789 (`0x1026F150`). Crushing blow uses doubles.
- **Answer:** the only player-driven life gain on Diablo Clone is its own **fire absorb**: flat 8 at all tiers plus 7n+2t %. That gain is what wraps a pinned Clone.

### 11.5 Summary by tier
- **Tier 0.** maxhp ×1.5; pinned only with 8 players. Fire absorb = 8 flat + 2t % (t = P−1). Cold, light and magic 2t %; DR 3t %.
- **Tier 1.** ×4·1.2, pinned from 2 players. +4 all skills. Fire/cold/light 7+2t %, magic 3+2t %, plus flat fire 8. Faster AI and souls.
- **Tier 2.** ×4·1.4, pinned at 2 and 4–8 players (3 players: 7.44M). +8 all skills. Fire/cold/light 14+2t %, magic 6+2t %, plus flat fire 8. 1/200 bonus drops and the tier-2 kill flag.
- **Tier 3.** Not reachable through `dcma`: only the InvaderPortal event, which a Clone portal overrides.
- **At every tier the Clone can gain life at fight start** through fire absorb, and through the +20 % heal when a player dies or leaves the arena. Neither wraps an unpinned Clone.
- **When it is pinned** (the same conditions as §2), one low-damage fire-only hit (below 16–80 HP post-resist, by tier and t) wraps its life negative and **kills it on that hit** (VERIFIED). The +20 % heals are clamped with unsigned math and are safe.
- **Fix:** clamp PD maxhp results to ≤ 0x7F000000 (§7). That leaves room for the 8 HP + 40 % absorb heal.
