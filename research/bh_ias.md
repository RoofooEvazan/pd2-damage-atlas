# BH "IAS (Frames)" breakpoints vs the game (Dragon Tail example)

Status: BH side READ in full for the single-hit path and reproduced exactly (`bh_ias.js`). Game side READ.

## Verdict
- **BH is right and our 9-frame answer is wrong.** The difference is PD2's Dragon Tail speed penalty.
  - PD2 Skills.txt gives Dragon Tail (id 270) **Param4 = −20**. Both `excel/` and `excel_live/` have this value.
  - Dragon Tail's start function adds Param4 to `attackrate` (stat 68) for the duration of the kick, then recomputes the animation rate.
  - Our game formula left this out.
- Correct numbers for the example:
  - s = 110 (stat 68: 100 − WSM(−10)) − 20 + EIAS(40) = 30 → **120**
  - rate = ⌊256·120/100⌋ = 307
  - frames = ⌈3328/307⌉ − 1 = **10**
- BH's list 0/4/11/23/39(10)/63/102/187 matches this exactly. It uses the same AnimData (AIKK1HS 13/256), the same start frame (0), the same WSM handling (stat 68) and the same frame formula.

## Game side (READ)
**Where the −20 is applied**
- **Server.** srvstfunc 27 (table 0x6FD27338[27]) = D2Game **0x6FCB2940**.
  - It first calls **0x6FCBE7B0**(unit, SkillsTxt+0x154 = Param4).
  - 0x6FCBE7B0 does the following:
    - allocates a stat list: D2Common #11013 (flags 4, owner = the unit's type and id);
    - attaches it: #10807(unit, list, 1);
    - sets stat 0x44 `attackrate` = Param4: #10188;
    - calls **#10819 = 0x6FD83110**, the animation-rate function, to recompute the unit's rate.
- **Client.** cltstfunc 9 (0x6FB8E928[9]) = D2Client **0x6FAF0400**.
  - It calls 0x6FB50880(unit, Param4), which does the same thing with the client's imports (#11013/#10807/#10188, then **#10819**).
  - So the client animation pace also includes the −20.

**Skills.txt field offsets** (D2Common field table at 0x6FDB5299…): Param1..8 = +0x148/+0x14C/+0x150/**+0x154**/+0x158/+0x15C/+0x160/+0x164; calc1 = +0x138; calc2 = +0x13C.

**No other speed rule for kicks.** 0x6FD83110 treats KK as an ordinary attack mode: s = stat 68 + EIAS, clamp 15..175, rate = ⌊speed·s/100⌋. This is the same as the FINDINGS rule.
- PD2's 4 patches inside 0x6FD83110 (0x6FD8319A/216/2AC/353 → PD 0x102EDD80 → 0x10267DA0) only replace the EIAS helper. The formula is unchanged: k·v/(k+v), with a guard against dividing by zero and the ItemStatCost record size changed to 0x150.
- The start frame is #10031, which applies to A1/A2 only, so KK has start frame 0.

**PD2 changes to Dragon Tail**
- srvstfunc 27 is not overridden. PD2 srvstfunc overrides are 28, 29, 37 and 66–78 only.
- PD2 has 2 call redirects inside 0x6FCB2940:
  - 0x6FCB29FC → PD 0x102EEDA0, which replaces the hit test 0x6FCFE5A0;
  - 0x6FCB2A23 → PD 0x102EDA70, which replaces the damage routine 0x6FCB2360.
  - Neither touches the Param4 call or #10819.
- cltstfunc 9 is not overridden. PD's pointer to the client start table (0x104E626C, from resolver 0x102888D0, offset 0xDE928) is only read, for entries 26 and 28.
- srvdofunc 50 (the kick hit) is overridden by PD 0x102FA2F0. It runs at the hit frame and handles damage and explosion. It was skimmed, and no attackrate change was seen.
- So PD2 changes Dragon Tail's speed only through the data value Param4 = −20.
- Dragon Talon (255) has Param4 = 50, but its srvstfunc 24 / cltstfunc 6 do not use it this way. BH also does not special-case it.

**Open point (minor).** The order of srvstfunc vs SetMode was not traced. It should not matter:
- the start function calls #10819 itself after attaching the list;
- BH deliberately skips adding Param4 while the unit is already in KK, because live stat 68 then already contains it.

## BH side
**Call site.** Panel 0x1007C040 → at 0x1007D379 calls **0x1007E310**(unit, "IAS (Frames):ÿc0 ", out, &y).
- FCR and FHR use a different routine. 0x1007E000(unit, stat, vector) looks up the static maps 0x1014D528 (FCR, key = class, with class 0x8F used for Wereform-type skills 0x35/0x40) and 0x1014D4F4 (FHR).
- The IAS list is computed live, not from a table.

BH resolves the D2 imports through version tables. Index 0 = 1.13c; all of these are in D2Common:
| BH thunk | D2Common | Use |
|---|---|---|
| 0x10042DD0 | #10507 0x6FD80420 | right skill (unit→pSkills+0xC) |
| 0x10043040 | #10350 0x6FDA01E0 | shapeshift class/mode remap |
| 0x10043010 | +0x41F70 0x6FD91F70 | AnimData record(unit, class, mode, type, inv) |
| 0x10042DA0 | #10031 0x6FD82220 | start frame |
| 0x10079C10 → [0x1014D474] | +0x323E0 0x6FD823E0 | EIAS(type 0 = IAS) |
| 0x100423E0 | (BH) | GetUnitStat(unit, stat, 0) |
| 0x10043160 | #10747 0x6FD80530 | can dual wield (Barb, Assassin, 417/418) |
| 0x10042E30 | #10786 0x6FDA1BF0 | skill calc evaluator |
| 0x100FC0D0 | CRT | `ceil` (roundsd mode 0xA) |

**Algorithm of 0x1007E310** (player, one weapon, single-hit skill):

1. **Pick the skill.**
   - skill = right skill. A monster with no right skill uses a per-class map at 0x1014D518.
   - skillId = SkillsTxt+0; mode = Skill+0x8, the skill's anim (Dragon Tail = KK, 12).
2. **Extra attackrate `X`** (starts at 0):
   - id 270 Dragon Tail and the unit's current mode (+0x10) ≠ KK(12) → X = **Param4** (+0x154).
   - id 133 Double Swing and current mode ≠ SQ(18) → X = Param5 (+0x158).
   - Ids 151, 380 and 425 (Whirlwind types) go to a separate path, 0x100808B5.
3. **Animation.**
   - Mode SC(10), or a mode of 3 → "N/A" path.
   - Otherwise #10350 remaps class/mode (wereforms), then 0x6FD91F70 returns the AnimData record. That gives `frames` = rec+8 and `speed` = rec+0xC. A speed of 0 is treated as 256.
   - Class 430/431 (wolf/bear) → special path, 0x10080840.
   - SQ(18) skills use the sequence table instead (0x1014D50C; frames from `seqnum`, speed 256, −30).
   - Charge (107) uses frames 8, speed 100, cap 256.
4. **Speed values.**
   - `base` = stat68 + X. With SQ, also −30.
   - `s` = base + EIAS(stat 93).
5. **Dual wield.** If the unit can dual wield and has two weapons:
   - it keeps the faster weapon, compared by stat68 + stat93 (the PD2 rule);
   - it recomputes base = stat68 and s = stat68 + EIAS (with SQ, both −30);
   - **X is dropped here.** This is a BH bug for dual-claw Dragon Tail and dual-wield Double Swing.
6. **Start frame.** It sets unit mode = skill mode temporarily, calls #10031 to get `start`, then restores the mode.
7. **Clamps.**
   - max = 175 (256 for Charge).
   - If base > max, base = max − 1.
   - s is clamped to 15..max.
8. **Rate values.**
   - len = (frames − start)·256.0
   - rate = trunc(s·speed/100)
   - **current = ceil(len/rate) − 1**, the "(10)"
   - minFrames = ceil(len/trunc(max·speed/100)) − 1
9. **Loop** (0x1007F170…0x1007FAD7). For t = base … max inclusive:
   - e = t − base
   - ias = ceil(120·e/(120 − e)), the exact inverse of EIAS
   - f = ceil(len/trunc(t·speed/100)) − 1
   - If f differs from the previous f, it appends `ias`, followed by " / ". When f == current, the entry is `ÿc8<ias> (<f>)ÿc0`.
   - After the loop there is a closing check at t = max (0x1007FAE2) that normally adds nothing.
10. **Multi-hit skills.** Zeal/Fend/Strafe/Fury/Dragon Talon are skills in the set at 0x1014D510, which is {26, 30, 106, 248, 255}. They use calc1 hit counts and the follow-up restart factor p (Param6/Param2/100). They build `a|b|c` lists via 0x10080B30. This path is not needed here and was not decoded in detail.

**Example check** (`node adv/re/bh_ias.js`):
- base = 110 − 20 = 90; s = 120; rate 307; current 10.
- The loop gives 0(14) 4(13) 11(12) 23(11) 39(10) 63(9) 102(8) 187(7). This is exactly the screenshot's "0/4/11/23/39(10)/63/102/187".

## Consequences for our calculator
- Add skill-specific attackrate from start functions that call 0x6FCBE7B0 or 0x6FB50880 with a Skills.txt param. Dragon Tail: +Param4 (−20 in PD2), applied only during the kick.
  - Other callers of these helpers:
    - Server 0x6FC47449: attackrate = calc3 (SkillsTxt+0x140).
    - Client 0x6FB762D0: attackrate = a calc at SkillsTxt+0x114.
    - Which skill uses them was not identified. Candidate: the Double Swing-type cltstfunc 27, which BH models with Param5.
- The breakpoints for Dragon Tail with this weapon are the ones BH shows. 9 frames needs 63 IAS.
