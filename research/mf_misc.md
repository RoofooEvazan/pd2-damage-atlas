# Magic find, gold find, experience and run speed (PD2 = 1.13c + ProjectDiablo.dll)

Every formula below was taken from the disassembly. JS is in `mf_misc.js`.

Status key:
- **VERIFIED**: the real game code ran natively in the harness and matched the JS/Python model on every case.
- **READ**: taken from the disassembly but not executed.

Harness: `harness/mfq.c` + `harness/mfq.py`, 250,000 cases (50k per group), 0 mismatches.

PD2 patch check: I scanned `adv/pd2_patch_records.json` for patch sites inside each function's address range.

## 1. Magic find in the quality roll

**Roll:** D2Game `0x6FC2FC40` (`__stdcall`; ecx = ilvl, eax = item id; args: game, seed-unit, killer, `u16 tc[4]`). It has one caller, the TC drop routine at `0x6FC3291C`.

**PD2:** there are no patch records in `0x6FC2E130`, `0x6FC2F2F0`, `0x6FC2FC40…0x6FC2FFBD`, `0x6FC41260`, or the ItemRatio lookup D2Common #10560 `0x6FDC4000`. PD's hook at `0x6FC32A0C` (→ `0x102C89D0`) runs *after* the roll. It only receives the rolled quality, and it forces "normal" for ItemTypes 105 (Worldstone Shard). It does not re-roll quality or read MF.

**MF value:** `0x6FC2F2F0`. MF = stat 80 on the killer (unit type 0 or 1 only), plus stat 80 of the killer's owner when the killer is a monster with an owner link (`0x6FC41260`: MonsterData+0x28 → unit lookup; READ; presumably mercenaries/pets).

**Diminishing helper:** `0x6FC2E130(x = 100+mf, f)`:
```
x <= 110 ? x : tdiv((x-100)*f, (x-100)+f) + 100
```
So effMF = `mf <= 10 ? mf : tdiv(mf*f, mf+f)`. There is no diminishing at all for mf ≤ 10, and that includes negative MF.

```js
effectiveMF(mf) = { unique: dim(mf,250), set: dim(mf,500), rare: dim(mf,600), magic: mf }
```

**The roll.** The checks run in order and the first success wins.

For each of unique (7), set (5), rare (6, only if ItemTypes +0x15 `Rare` is set) and magic (4):
```
c = (Ratio − tdiv(ilvl − qlvl, Divisor)) << 7          // qlvl = Items.txt level (+0xFD)
if (mf != 0) c = tdiv(c*100, 100 + effMF)               // magic uses 100+mf (no diminishing)
c = max(c, Min)
c -= (c * tc[q]) / 1024  (signed, toward zero)          // tc = TreasureClassEx quality value, max along the TC chain
success if c <= 0 or rand(c) < 128                      // p = min(1, 128/c)
```

If the raw MF is ≤ −100, all four checks are skipped.

After the four checks:
- Superior (3): `c = (HiQuality − tdiv(ilvl−qlvl, HiQDiv)) << 7`, with no MF, no min and no TC.
- Normal (2) vs low (1): the Normal / NormalDivisor columns in the same way. c ≤ 0 → normal.

RNG: `0x6FC211D0`. It uses the seed at seed-unit+0x20. seed = lo·0x6AC690C5 + hi, and the result is lo % n (or lo & (n−1) when n is a power of two).

**ItemRatio row:** D2Common #10560 picks the row with the highest Version ≤ 100 (the constant passed) that matches:
- Uber = the base is exceptional/elite (`0x6FD74D00`)
- Class Specific = ItemTypes.Class < 7 (`0x6FD74320`)

So only the Version 1 rows are used. PD values (`data.zip`, identical in `itemratio.bin`):

| row | Unique/Div/Min | Rare/Div/Min | Set/Div/Min | Magic/Div/Min | HiQ/Div | Normal/Div |
|---|---|---|---|---|---|---|
| normal | 400/1/6400 | 100/2/3200 | 160/2/5600 | 34/3/192 | 12/8 | 2/2 |
| uber | same | same | same | same | 12/8 | 1/1 |
| class | 240/3/6400 | 80/3/3200 | 120/3/5600 | 17/6/192 | 9/8 | 2/2 |
| class uber | same | same | same | same | 9/8 | 1/1 |

Record layout (+0 Unique, +4 UDiv, +8 UMin, +0xC Rare … +0x18 Set … +0x24 Magic … +0x30 HiQ, +0x38 Normal) was confirmed from the code and from `itemratio.bin`.

**VERIFIED:** 50,000 random rolls, 0 mismatches. The real `0x6FC2FC40`, MF getter, helper and RNG were run, varying ilvl, qlvl, MF (−150…2000 including the edges 10/11/−100), TC values, all 6 rows, and rare allowed/forbidden. The model matched both the returned quality and the final RNG state.

Stubs: data lookups (#10695 item record, #10082 item type, #10560 row) and the stat getter #10910.

**Correction (drops.md, VERIFIED with the full PD2 tables, harness/tcq.c):** the roll also has early returns (ItemTypes `Normal` → 2, Items `unique` → 7, ItemTypes `Magic` + Items `quest` → 7), and for ItemTypes with `Magic`=1 a failed rare check returns 4 directly (0x6FC2FED2) without the magic check. The harness above used a type row with Magic=0, so it did not exercise this.

**Caveats:**
- The TC column → `tc[]` word mapping (magic, rare, set, unique) is by position in the roll. The TC-side field offsets (+0xE…+0x14 in the per-pick record, max over the chain at `0x6FC3285A…`) are READ.
- The later "no unique/set base available → downgrade" step and PD map-mod item changes (inside `0x102C89D0`) are not modelled.

## 2. Gold find: D2Game `0x6FC2F260` (VERIFIED, 50k, 0 mismatches)

Only for items whose type (#11088) is ItemTypes row 4 `gold`.

```js
gf = stat79(killer) + stat79(owner of killer, same owner rule as MF)
gold' = max(0, tdiv(gold * (100 + gf), 100))         // goldFind(gold, gf)
```
- No cap. The MaxGold (#10049) clamp in the code only runs when the target unit's type is 0 (player). A gold pile is type 4.
- Called once, from the TC drop routine at `0x6FC32A9B`.
- PD: no patches in the function, and PD does not reference it.

## 3. Experience (stat 85 `item_addexperience`)

**Function:** D2Game `0x6FCFC030`, the per-player gain (eax = exp, ebx = player, edi = player level, arg2 = monster level). Steps in order:
1. `exp = min(exp, 0x7FFFFF)`. If exp ≤ 0 → nothing. If player level ≥ max level (#10066) → nothing.
2. Level-difference penalty. Stock `0x6FCFAA40`; **PD2 replaces this call** (`0x6FCFC075` → `0x102EF9A0` → `0x102CACF0`).
   - Monster level ≤ player level: `f = T_le[min(p−m,10)]`.
     - Table D2Game `0x6FD1A01C`: `[256×6, 207, 159, 110, 61, 13]`.
   - Monster level > player level, player level ≤ 19 (or monster level ≤ 0): `f = T_gt[min(m−p,10)]`.
     - Table `0x6FD1A048`: `[256×6, 225, 174, 92, 38, 5]`. PD resolves both tables to these same stock addresses through its version table.
   - Player level 20–24 (PD only): `f = clamp(tdiv(3p, p+5d)·256 − 64, 13, 256)`, integer division.
   - Player level ≥ 25: `exp = trunc(exp·p/m)`. Stock does the same through MulDiv.
   - PD computes `trunc(exp·f/256)` in double precision, capped at 0x7FFFFFFF.
   - Stock differs only in the 20–24 band, where it uses T_gt.
3. ExpRatio (`0x6FCFA9E0`): `exp = (exp·ExpRatio[plvl]) >> ExpRatio[0]` (shift 10), with a `(exp>>10)·ratio` overflow path.
4. **Stat 85:** `if (b) exp += MulDiv(b, exp, 100)`. MulDiv is `0x6FC214D0`, which is `tdiv(exp·b, 100)` for normal sizes. The stat is read with #10973.
5. Award: `0x6FCFB6B0`.

PD also replaces the caller's apply step (`0x6FCFE6D0` → `0x102C9AF0`).
- It adds no multiplier.
- It refuses exp when a playerdata flag (+0x264) is set.
- In two PD-defined level-id sets it requires a caller argument ≥ 80, probably the monster level (READ).

**VERIFIED:**
- Stock `0x6FCFC030` (stock penalty) against the model: 50k cases, 0 mismatches. This covers the exp cap, penalty, ExpRatio, stat 85 incl. negative and large values, and max level.
- PD `0x102CACF0` run natively (tables set to the stock ones): 50k cases, 0 mismatches.

```js
expBonus(exp, bonus)                       // step 4 only
expGain({exp, plvl, mlvl, expRatio, bonus})// steps 1-4 with the PD penalty
```

## 4. Walk/run speed (FRW 96 + velocitypercent 67)

**Function:** D2Common `0x6FD83110`, walk/run branch `0x6FD8331B…0x6FD833CF`. It runs on every mode set and every recalculation.

```js
s   = max(25, stat67 + tdiv(150*frw, 150+frw))      // EFRW, table 0x6FDE4608 row {1,150,96}
vel = tdiv((CharStats.WalkVelocity << 8) * s, 100)  // 0x6FD80D50 (players: byte +0x40 <<8), stored by 0x6FD84D40 in path+0x7C/+0x84
animRate = tdiv(animSpeed * s, 100)                 // unit+0x4C, clamp 0..0x7FFF
```

**Where stat 67 comes from:**
- Base 100, set when the character loads (D2Game `0x6FC760D2`).
- Running (mode 3 / 0x13): SetMode `0x6FD8368A` → `0x6FD82E10` adds a statlist with `tdiv(RunVelocity·100, WalkVelocity) − 100`. PD CharStats has Walk 6 and Run 9 for every class, so this is **+50** (READ).
- Armor/shield `speed` column: `−Items.speed` (`0x6FD7ADD9`).
- Skills/auras that write velocitypercent.

**Movement:**
- Server step D2Common `0x6FD5CEB0`, called with 0x400 (D2Game `0x6FD01960`): `step = vel·0x400 >> 6 = vel·16` in 1/65536-subtile units.
- It is multiplied by a direction vector of length 4096 (table `0x6FDDD470`, 128 entries, norm 4094.8–4096) and shifted right by 12.
- **Speed = vel·16/65536 subtiles per frame × 25 fps = vel·25/4096 subtiles/s.**
  - Walk at 100% = 9.375 subtiles/s.
  - Run (s = 150) = 14.0625.
  - Run with 40 FRW (EFRW 31, s = 181) = 16.97.
- The yard/subtile ratio is not in this code. I report subtiles and percentages.

**PD2:**
- PD redirects the EIAS helper calls (`0x6FD83353` etc. → `0x10267DA0`). The formula is identical (`tdiv(k·v, k+v)`, returning 0 when k+v = 0), and it reads the same stock table: its pointer resolves to D2Common rva 0x94608, and no patch records touch the table.
- PD widens the ItemStatCost record size (0x144 → 0x150, `0x6FD8241A`). That does not change the formula.
- No patches in `0x6FD80D50`, `0x6FD84D40`, `0x6FD5CEB0`, `0x6FD82E10`.

**VERIFIED:** 50k cases, 0 mismatches. The real `0x6FD83110` walk/run branch, stat getter #10973, EIAS helper and CharStats lookup were run, with random class, WalkVelocity, stat 67 (−100…300) and FRW (−100…1000). Only the path velocity is compared.
- **READ, not run:** the run statlist (+50), the path step and direction norm.
- **Not run natively:** PD's EIAS replacement. It is read to be the same formula on the same table.

```js
runSpeed({frw, running=true, velocityPercent /* extra stat 67 */, walkVelocity=6, runVelocity=9})
  → {efrw, speedPercent, pathVelocity, subtilesPerSecond, percentOfBaseWalk, percentOfBaseRun}
```

## 5. Other

- **FHR:** EFHR = `tdiv(120·x, 120+x)` from the same table (row {1,120,99}). The hit-recovery animation rate is 50 + EFHR. Both were VERIFIED earlier through the EIAS helper (see FINDINGS "Attack speed").
- **Light radius (89):** not done. The stat is read in D2Client `0x6FAD15E0` / `0x6FB60C68` and D2Common `0x6FD5970A`, but the radius formula was not traced.
