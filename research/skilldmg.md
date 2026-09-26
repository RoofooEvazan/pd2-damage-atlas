# Skill damage: character-screen box, skill-tree tooltip, server

Sources: the stock 1.13c D2Client, D2Common and D2Game, Fog.dll, ProjectDiablo.dll (PD) and its patch records (`adv/pd2_patch_records.json`). The JS is `adv/re/skilldmg.js`. Its data is `adv/re/skilldmg-data.json`, built by `adv/re/extract_skilldmg.py`.

**Status key**
- **VERIFIED**: the game code was run natively and matched the JS on every case.
- **READ**: read from the disassembly but not executed.

## Native checks
Build steps:

```
python3 harness/skdmg_build.py            # Fog.img + pd2_*.bin (the compiled tables PD2 ships in data.zip)
gcc -m32 -static -nostdlib -O1 -fno-pic -fno-stack-protector -o harness/skdmg  harness/skdmg.c
gcc -m32 -static -nostdlib -O1 -fno-pic -fno-stack-protector -o harness/skcalc harness/skcalc.c
node harness/skdmg_driver.js ; node harness/skcalc_driver.js
```

### `harness/skdmg.c`: 58,455 cases, 0 mismatches
The test builds real Skills, SkillDesc and Missiles records from `skilldmg-data.json`. It then runs this code:
- **D2Common elemental and physical:**
  - #10121 and #11091 elemental min and max, with and without mastery.
  - #10510 elemental length.
  - #10567 and #10297 skill physical.
- **D2Common missile functions:** #10205, #10532, #10242, #10040 and #10256.
- **PD2 mastery replacement:** the code-init patches at D2Common 0x6FDA0443 and 0x6FDA054A redirect to PD 0x102EEAA0, then 0x10268A50.
- **D2Client descdam handlers:** 1, 5, 6, 7, 8, 9, 12, 17, 19 and 23.
- **PD2 handlers:** 25, 26 and 28. PD 0x102F92B0 writes these into the descdam table at runtime.

Test setup:
- The box printer 0x6FADF0A0 is replaced by a recorder.
- The player has no inventory, so the weapon block adds only its item-elemental part (0x6FAE0E30). The JS feeds that part through `charscreen.addElemental`.
- The calc evaluator, Fog #10253, is stubbed and returns the value `skilldmg.js` computed for that calc. The calc language is tested by the second harness.
- Every player skill is tested at 19 levels for each function, and every referenced missile the same way.
- Random masteries (329–332, 357), T25 and item elemental stats are used.

Mutation checks all produce mismatches, which shows the test can fail:
- dropping the synergy gate;
- removing PD's magic mastery;
- changing the SrcDam scaling;
- rounding muldiv.

### `harness/skcalc.c`: 24,315 calc evaluations
The test takes the game's own compiled bytecode from `skills.bin`, `skilldesc.bin`, `missiles.bin` and `*code.bin` in data.zip. These are the files the client loads. It runs them on the real Fog evaluator (0x6FF69E90) through:
- D2Common #10786 (skill calcs);
- #10435 (SkillDesc calcs);
- #10231 (missile calcs);
- D2Common's own keyword and function callbacks (0x6FDA1B60 → 0x6FDA1070, table 0x6FDE9E24).

The player has a real skill list, with hard points at skill +0x28, and a stat list.

Results:
- Every damage-related calc matches `skilldmg.js` evaluating the `.txt` text. That covers every synergy, every `ddam calc`, `p1..3dmmin/max`, every `calc1-4`, every desc line with a damage keyword, `miss()`, `sklvl()` and `stat()`.
- The 495 mismatches are all keywords that `skilldmg.js` deliberately returns 0 for, because they are not damage: `madm`, `math`, `macr` (weapon-mastery text), `len`, `mps`, `usmc` (mana) and `par34`.
- Every calc text in the tables compiled. No field has text without code.

### The tables the game actually loads
`data.zip` ships `skills.bin`, `missiles.bin` and `skilldesc.bin`, dated 2026-09-06. These are newer than the `.txt` files. The .bin records match the .txt values field for field (the extractor's defaults were fixed to match the .bin):
- An empty `HitShift` compiles to **0**, not 8. This affects Poison Javelin and 80 missiles.
- `SrcDamage -1` becomes the byte **255**.

The local `data/global/excel` Skills.txt and SkillDesc.txt only add rows after the stock ones. Missiles.txt, SkillCalc.txt and MissCalc.txt are not in the local folder, so the extractor reads them from `data.zip`.

## Building blocks (D2Common)
All values are in 256ths unless marked. `tier(l, t)` is 0x6FD9DDB0, which applies level brackets 2–8, 9–16, 17–22, 23–28 and 29+ (the same as engine.js `lvtier`). `muldiv(a,b,100)` is 0x6FD511E0, `a*b/100`, with a 64-bit path for large values.

| function | formula | status |
|---|---|---|
| #10121 elem min 0x6FDA0460 (unit, skill, lvl, mastery) | lvl≤0 → 0. `v = (EMin + tier(lvl,EMinLev)) << HitShift`. **Synergy only if `v > 256` or `EMinLev1 ≠ 0`**: `v += muldiv(EDmgSymPerCalc, v, 100)`. Then if mastery: `v += muldiv(M, v, 100)` | VERIFIED |
| #11091 elem max 0x6FDA0360 | Same with EMax/EMaxLev, **no gate** on the synergy | VERIFIED |
| mastery (PD 0x10268A50 replacing the stock 0x6FD9F870) | Fire → 329, light → 330, cold and freeze → 331, poison → 332. **Magic → 357** (PD2 only). Other elements get 0 | VERIFIED |
| #10510 elem length 0x6FD9E900 | lvl≤0 → 0. `ELen + (l≤8 ? (l-1)L1 : l≤16 ? 7L1+(l-8)L2 : 7L1+8L2+(l-16)L3)`, `+ muldiv(ELenSymPerCalc, v, 100)`. The mastery flag is ignored | VERIFIED |
| #10567 / #10297 skill phys 0x6FDA2100 / 0x6FDA1FF0 | `v = MinDam + tier(lvl, MinLevDam)` (plus weapon×SrcDam/128 when flag 1). `v += muldiv(DmgSymPerCalc, v, 100)`. `<< HitShift`. **Kick** flag → 0x6FD9F930 instead | VERIFIED (flag 0) |
| missile #10205 / #10532 / #10040 / #10256 | `(base + tier) + muldiv(syn, ·, 100)` **before** `<< HitShift`. No mastery. lvl≤0 → the missile's own level | VERIFIED |
| missile length #10242 0x6FDB9DC0 | lvl≤0 → ELen, else as #10510 without synergy | VERIFIED |

**Answers to the specific questions:**
- **Elemental mastery multiplies the synergised value.** The order is base → +synergy% → +mastery% of that sum, all in 256ths, each step truncating.
  - It applies only where the caller passes mastery=1: the `enma`/`exma`/`enms`/`exms` keywords, the box handlers, tree line types 10/14/24/26/27, and the server's skill-linked missiles.
  - `edmn`/`edmx`/`edns`/`edxs` pass 0 and get no mastery.
- **+skills** enter through the level argument. The box dispatcher 0x6FADFD5D and the tree tooltip use #10306(unit, skill, 1), which is hard points plus item/oskill bonuses. That level drives the tiers, `lvl`, `ln`/`dm` and the `ddam` calcs.
- **Synergies use hard points only.**
  - `skill('X'.blvl)` is the unit's skill object +0x28, clamped to [0, experience.txt MaxLvl] (keyword 0x6FDA14C6).
  - `skill('X'.lvl)` (calc function 0x6FDA1AF0) is hard + bonuses (0x6FD9FCB0). Dragon Claw's `skill('Claw Mastery'.lvl)` uses it.
  - A skill the unit does not own has lvl = blvl = 0.
- **The low-damage gate** in #10121: when the whole min is ≤ 1 point and EMinLev1 is 0, synergies do not raise the min. Examples are Power Strike and Charged Bolt Sentry at low level ("1-965").

## Calc language (Fog #10254 compiler, #10253 evaluator): VERIFIED
- **Integers only.** `/` truncates. **x/0 = 0** (0x6FF6A008). `^` is a power (b≤0 → 1).
- **Precedence** is a shunting-yard: an incoming operator with code c pops stacked operators whose precedence byte (Fog 0x6FF76E1C) is ≥ c.

| operators | incoming code | stacked precedence |
|---|---|---|
| , | 3 | 3 |
| comparisons | 10–15 | 15 |
| + | 16 | 17 |
| − | 17 | 17 |
| * | 18 | 19 |
| / | 19 | 19 |
| ^ | 20 | 20 |
| unary − | 21 | 21 |
| ? | 22 | 22 |

- **`?` binds tighter than everything** on both sides (`:` is only a separator):
  - `a + b ? c : d` = `a + (b ? c : d)`;
  - `c ? x : y + z` = `(c ? x : y) + z`;
  - `a < b ? c : d` = `a < (b ? c : d)`.
  - An operator inside an unparenthesised branch fails to compile, and the loader then stores no calc.
  - This decides Static Field's and Inferno's desc lines, and Corpse Explosion's radius.
  - engine.js uses C precedence and differs here.
- **Functions** (0x6FDE9E24):
  - `min`, `max`, `rand`;
  - `skill('X'.kw)`: kw evaluated for X at X's total level;
  - `miss('m'.kw)`: missile keyword at this skill's level;
  - `stat('s'.accr)`: full total; `.base` and `.mod` are the other lists;
  - `sklvl('X'.kwLevel.kw)`: level = kwLevel evaluated here, then kw for X. Fire Golem uses it for its Holy Fire.
- **Skill keywords** (0x6FDA1070, table 0x6FDA174C, SkillCalc.txt order):
  - `edmn`/`edmx` = #10121/#11091(…, 0) >> 8;
  - `edns`/`edxs` are the same in 256ths;
  - `enma`/`exma`/`enms`/`exms` are the mastery versions;
  - `edln` = `edma` = #10510;
  - `m1en`/`m1ex`/`m1el` = descmissile1 #10205/#10532 >> 8 and #10242;
  - `m1eo`/`m1ey` are 256ths, `m1rn` = Range + lvl·LevRange (and so on for m2/m3);
  - `clcN` = Skills calcN, `ulvl` = stat 12.
- **Missile keywords** (0x6FDBA790, MissCalc.txt): `par1-5`, `lvl`, `edmn`/`edns`/…, `damn`/`dmns`/…, `rang`, `sl12`/`sd12`/…. Unknown names (e.g. `par7` in `firearrow firewall`) evaluate to 0.

## Character-screen damage box (SkillDesc.descdam)
- **Dispatcher** 0x6FADFD30. Table 0x6FBA5210 is indexed by descdam (< 0x90).
- **PD 0x102F92B0** writes entries 11, 13, 15, 16, 25, 26, 27 and 28 at runtime.
- **Handler call:** `ecx=unit, edx=unit skill, stack (SkillsTxt, lvl, x, y, w)`.
- **Printer** 0x6FADF0A0(ecx=min, eax=max) prints `max(max, min+1)`, so a range always shows min < max. Values ≥ 10000 are shown as "Nk".
- **Notation:**
  - S = skill physical `#10567>>8 .. #10297>>8`.
  - E = elemental with mastery `#10121(…,1) .. #11091(…,1)` (256ths).
  - E↓ = `E>>8`, or for poison `(E·len)>>8` (0x6FADE190).
  - W(ed, flat, src) = the weapon block D2Client 0x6FAE3240: physical 0x6FAE1220 then 0x6FAE0E30 (VERIFIED in charscreen.md).
  - r7(x, s) = `(x·s)/128` toward zero.

| descdam | handler | shown | skills | status |
|---|---|---|---|---|
| 5 | 0x6FAE5270 | `r7(W(0,0,128), SrcDam) + S + E↓` (weapon part only when SrcDam ≠ 0) | 56 spells/auras (Fire Ball, Blizzard, Holy Fire, Poison Nova, Twister, Meteor…) | VERIFIED |
| 6 | 0x6FAE5540 | two ranges: weapon `r7(W,SrcDam)` and skill `S + E>>8 + same-element item damage` (T48/49…, no mastery; poison ×len) | Exploding/Immolation/Freezing/Shattering Arrow | VERIFIED |
| 7 | 0x6FAE4C00 | `W(ddam calc1, ddam calc2, 128)`, then ×SrcDam/128 if SrcDam≠128 (muldiv), `+ S + E>>8` (no poison length) | Jab, Multi Shot, Guided Arrow, Strafe, Fend, Sacrifice, Zeal, Charge, Bash, Stun, Leap Attack, Concentrate, Feral Rage, Fury, Joust | VERIFIED |
| 1, 19 | 0x6FAE5470 / 0x6FAE4AB0 | as 7; dual wield runs 0x6FAE0B40 per hand | Whirlwind, Maul, Blade Dance / Double Swing, Frenzy, Berserk, Dragon Claw | VERIFIED (one hand) |
| 8 | 0x6FAE07B0 | per second: `(a·E·25/b)>>8` with a = ddam calc1 or 1, b = ddam calc2 or 1 | Inferno, Arctic Blast, Shock Field, Inferno Sentry | VERIFIED |
| 9 | 0x6FAE5070 | per 3 s: `(((MinDam<<HitShift) + E)·75)>>8` (raw MinDam, not level-scaled), `+ r7(W,SrcDam)` | Blaze, Fire Wall, Firestorm | VERIFIED |
| 12 | 0x6FAE04A0 | `E↓ + E↓·pct/100`. Expansion (flag 0x6FBC9854): pct = only the Concentration state's stat 25 (D2Common #10037, state 42); classic: T25 ≥ −90 | Blessed Hammer | VERIFIED |
| 17 | 0x6FAE40F0 | weapon `W(ddam1, ddam2, 128)` and `E↓` as two ranges | Charged Strike, Lightning Strike | VERIFIED |
| 23 | 0x6FAE4320 | weapon `W(0,0,128)` and `E↓` | Rabies | VERIFIED |
| 25 | PD 0x102F8B60 | `W(ddam1 × (n+1), 0, 128) + p{n+1}dmmin..p{n+1}dmmax` (calcs, element p{n+1}dmelem). n = aurastat1 in the aurastate's stat list (charges), capped at 2. Nothing printed unless both ends ≠ 0 | Tiger Strike, Fists of Fire, Cobra Strike, Claws of Thunder, Blades of Ice, Royal Strike | VERIFIED |
| 26 | PD 0x102F8CE0 | `W(0,0,SrcDam) + r7(E>>8, SrcDam)`. SrcDam 0 → no elemental at all | Magic/Fire/Cold Arrow (SrcDam 96/64/64) | VERIFIED |
| 28 | PD 0x102F9100 | S is placed in min/max **before** the weapon block, so ED multiplies it: `W(0,0,SrcDam, pre=S)` | Blade Sentinel, Blade Fury | VERIFIED (no weapon) |
| 27 | PD 0x102F8E20 | weapon `W(0,0,128)` and fire `(E·75)>>8` per 3 s, formatted into one string by C++ string code | Fire Claws | READ |
| 10 | 0x6FAE05D0 | shield: see formula in `parts.shield` | Smite | READ |
| 11 | PD 0x102F8040 | weapon + per element (fire/cold/light) calc1/calc2/calc3 % + mastery of it, with the matching item/aura elemental; SSE float math | Vengeance | READ, not reproduced |
| 13 | PD 0x102F8410 | both thrown weapons via PD 0x102F1060 with ddam calc1/calc2 | Double Throw, Split Throw | READ (edPct/flat) |
| 15, 16 | PD 0x102F8490 / 0x102F86B0 | boots kick damage × (100 + ddam calc1 + Str/Dex terms) + weapon min/max, float math | Dragon Talon/Flight, Dragon Tail | READ (edPct only) |
| 21 | 0x6FAE08C0 | throw damage `T159/T160 × (100 + T18/T17 + Str/Dex bonus + T25)/100 + throw mastery (state 78)` + same-element item damage, ×SrcDam/128, then `E↓` as a second range | Lightning Bolt | READ (E part) |
| 22 | 0x6FAE4640 | PD throw damage (0x6FAE2F30 → PD 0x102F91B0) ×SrcDam/128, then `E↓` | Poison/Plague Javelin, Lightning Fury | READ (E part) |
| 2 | 0x6FAE0AE0 | `(MinDam << (HitShift−8)) + T137` | Kick | READ |

Smite (READ) in detail:
- `pct = (Param3 + (lvl−1)·Param4) + Str·StrBonus/100 + Dex·DexBonus/100 + T25`, then `pct ≥ −90`.
- `min = sMin + trunc((pct·sMin + T18)/100)` and `max = sMax + trunc((pct·sMax + T17)/100)`.
- sMin/sMax is the shield's Armor.txt damage (+0xFE/+0xFF), plus Holy Shield's #10567/#10297 while in state 101.
- The stock code adds T17/T18 instead of multiplying them.

The 109 player skills with no descdam (passives, auras such as Might, curses, summons, Venom, Raven, Enchant, Plague Poppy…) show no damage box. For them, `skilldmg.js` takes the primary numbers from the first damage line of the tree tooltip (below).

## Skill-tree tooltip lines (D2Client 0x6FAE16C0, switch 0x6FAE2ABC on descline/dsc2line type)
Structure:
- The builders are 0x6FAE3290, 0x6FAE3550 and 0x6FAE3B50.
- `desc*` lines use the current level. The `dsc2` block shows the current and the next level. `dsc3` is the synergy list.

| type | handler | value | status |
|---|---|---|---|
| 9 | 0x6FADD000 | `S + S·calcA/100 + calcB` (S >> 8) | READ (VERIFIED parts) |
| 10, 24 | 0x6FADEF80 | E with mastery >> 8 ("Fire Damage: a-b") | READ (VERIFIED parts) |
| 11 | 0x6FADF2E0 | #10510 length (frames → seconds) | READ |
| 14 | 0x6FADEC60 | `(E·len)>>8` over len (min == max prints one number, no +1) | READ |
| 26 / 27 | 0x6FADEB80 / 0x6FADED80 | `(E·25)>>8` per second / `(E·75)>>8` per 3 s | READ |
| 22 | 0x6FADEDD0 | descmissile1 `mE·75`, plus fire (T329) or light (T330) mastery **by the skill's** EType, >> 8 | READ |
| 50 | 0x6FADEF20 | descmissile1 `mE >> 8`, **no mastery** | READ |
| 38, 47, 59–61 | various | `calcA-calcB` | calcs VERIFIED |
| 43, 44 | 0x6FADDE00 | calcA/256 and calcB/256 as decimals | calcs VERIFIED |

Most PD2 damage lines are calc-driven (types 38/43/47/59), for example:
- Fire Arrow `enma / 2`;
- Cold Enchant `edmn·(100+T331)/100`;
- Skeletal Mage `miss('necromage3'.edns)·…`;
- Fist of the Heavens `m1en·(100+T357)/100`.

## Server (what is actually dealt)
- **Missiles** (D2Common #10413 0x6FDBBAA0):
  - A missile whose `Skill` column names a skill, or that has the MissileSkill flag, rolls that skill's #10567/#10297, #10121/#11091 **with mastery**, and #10510. Its SrcDam is Skills.SrcDam, or 0 if the missile's SrcDamage is 255.
  - Other missiles use their own Missiles.txt damage at the skill level. They add mastery only if `ApplyMastery` is set (0x6FDBAB20 → PD 0x10268A10, same element map).
  - When SrcDam ≠ 0, item elemental damage is added (0x6FDBABE0, PD-patched calls) and then **every damage slot, including the skill's elemental, is ×SrcDam/128** (0x6FDB9C60).
  - So for Fire/Cold/Magic Arrow the box's `r7(E, SrcDam)` matches the server.
  - A descdam-5/7 skill with 0 < SrcDam < 128 (Blade Shield 32, Guided Arrow 64) shows its own E unscaled. If that skill hits through a skill-linked missile, the missile deals E×SrcDam/128. Check `res.server`.
  - `res.server` lists each srvmissile (and one level of sub/explosion missiles) with what it rolls.
- **Where the tooltip and the server disagree:**
  - Line type 50 omits mastery that the server adds for ApplyMastery missiles.
  - Line type 22 picks fire/light mastery from the skill's EType, not the missile's.
  - `edmn`-based tooltip calcs omit mastery (PD2's texts usually multiply it back in by hand).
  - Descdam 8/9/27 show per-second or per-3-second figures of a per-frame roll.
- **Weapon skills:** the server's damage % comes from the do-function evaluating a Skills.txt calc with #10786 (READ). For example, srvdofunc 13 (Zeal/Fend/Fury) uses calc2 (+0x13C), which is the same text as the tooltip's `ddam calc1 = clc2`.
  - Stock do-functions 7/8/11/12/13/14/42/50/64/76/120/150 read calc1–calc4 (scan of 0x6FD274A8).
  - The do-functions ≥ 153 used by PD2 skills are installed at runtime and were not audited.

## skilldmg.js
`skillDamage(skillId, ctx)` returns an object:
- `type`: the element of the primary numbers, one of phys/fire/ltng/cold/pois/mag.
- `min`, `max`: the skill's own numbers as the handler adds them. The weapon share is not included.
- `lenFrames`, `edPct`, `flat`, `srcDam`.
- `source`: 'skill' or 'missile'.
- `descdam` and `display`.
- `parts`: `skillPhys`, `elem`, `weapon {edPct, flat, src, pre, …}`, `itemElem`, `charge`, `shield`, `conversion`.
- `total`: the exact printed pair. Present only when `ctx.weaponFn({edPct, flat, src, pre}) → {min,max}` is supplied, for example built from `charscreen.js`. `totalWeapon` is the second range for 6/17/23.
- `lines`: the tree tooltip damage lines.
- `server`.
- `notes`.

`ctx` fields:
- `level` (incl. +skills), `blvl`, `levelsOf(id) → {lvl, blvl}`, `T(stat)`;
- `D` (engine data; used for stat names);
- `SD` (defaults to `require('./skilldmg-data.json')`);
- `charges` (descdam 25);
- `concPct` and `expansion` (descdam 12).

Other exports: `elemMin`/`elemMax`/`elemLen`/`physMin`/`physMax`/`missile`/`evalCalc`/`evalMissCalc`/`muldiv`/`lvtier`.

## Coverage
Player skills: 231 class skills plus Attack, Kick, Throw, Unsummon and the two left-hand skills.
- **Box numbers VERIFIED:** descdam 1, 5, 6, 7, 8, 9, 12, 17, 19, 23, 25, 26, 28. That is 111 skills.
- **Box READ with the skill part computed:** 21, 22, 27 (5 skills), and 13 (edPct/flat).
- **Not reproduced beyond edPct or notes:**
  - Smite (shield data needed);
  - Vengeance (PD float conversion);
  - Dragon Talon/Tail/Flight (PD kick code);
  - the PD throw formula (descdam 3/4/22 weapon part);
  - the weapon share of every weapon skill, which needs gear: pass `weaponFn`.
- **No descdam (109 skills):** tree damage lines are computed where they exist:
  - types 9/10/14/22/26/27/50;
  - calc lines 38/43/44/47/59–61. Examples are summons' line 9, Raven, Venom, Enchants, Skeletal Mage and Fire Golem.
  - Passives, auras and curses without damage return 0.

## Caveats
- **Calcs are evaluated from the `.txt` text** with Fog's rules. The .bin compiled code was proven equal for every damage calc at the tested levels. The mana and weapon-mastery keywords return 0.
- **engine.js differences** (not edited):
  - its `edmn`/`edmx` synergy ignores the #10121 min gate;
  - its `enma`/`exma` add no mastery;
  - it lacks `m1en`, `miss()` and `sklvl()`;
  - it uses C ternary precedence;
  - its extractor defaults an empty HitShift to 8 (the game uses 0).
- **The 0x6FAE0E30 weapon elemental** in the box ignores magic mastery. Descdam 6's same-element item damage and 21's add no mastery at all.
- **Descdam 25** reads the charge count from the unit's aurastate stat list. `ctx.charges` must be supplied.
- **Descdam 12's Concentration percent** is the state's own stat-25 list only.
- **`rand()` in a calc** returns the low bound.
