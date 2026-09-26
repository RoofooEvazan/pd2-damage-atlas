# Whirlwind hit timing (PD2 = 1.13c + ProjectDiablo.dll)

This replaces the "Whirlwind hit schedule" MODEL in `crit_cb.md`. That model was wrong in two ways: it tied the hit rate to IAS, and it spread a pack's hits round-robin.

JS: `adv/re/whirlwind.js` (`whirlwindHits`, `timeline`, `pickTarget`, `targetRadius`).
Native check: `harness/ww.c` + `harness/ww.py` (game code against the Python model), then `harness/ww_check.js` (JS against the same native results).

**Status key**
- **VERIFIED**: the real code ran natively in the harness and matched the model on every case.
- **READ**: read from the disassembly only.
- **DATA**: from the PD2 tables.

## Summary

- **IAS, weapon speed (WSM) and skill attack rate do nothing to Whirlwind in PD2.** (VERIFIED)
  - Whirlwind has no `UseAttackRate` in Skills.txt. Its sequence rate therefore stays at 256, one sequence record per frame, whatever the gear.
  - PD2 also removed the stock gate that took the delay from weapon speed.
- **The do-func runs every server frame.** After the first event, each call rewinds the sequence to record 3, which puts the next event on the next frame. The 8-record sequence never loops on the server. (VERIFIED)
- **Hits come from PD2's gate alone.**
  - One weapon: 1 hit every 5 frames = **5.00 hits/s**.
  - Two weapons: 2 hits every 6 frames = **8.33 hits/s**.
  - Two weapons on a PvP map: 2 hits every 5 frames = **10.00 hits/s**. (VERIFIED)
- **The first hit lands 3 frames (0.12 s) after the cast starts.** Hits then fall on fixed 5-frame (or 6-frame) slots. A slot with no enemy in range is wasted. (VERIFIED)
- **Moving further does not change hits per second, only how long the cast lasts.** The path moves one step per frame. Whirlwind stops on the frame the path ends, and 4 more idle frames pass before the sequence ends. (VERIFIED for the timing; the path speed itself is READ)
- **Targets:** each hit takes the nearest enemy in range that was not the previous hit's target.
  - With two or more enemies in range, the hits alternate between the **two nearest**.
  - A third enemy is hit only when movement changes which two are nearest. (VERIFIED)

## 1. Sequence rate: fixed at 256 (VERIFIED, 20,000 cases)

Server SQ start, D2Game 0x6FC987C0 → 0x6FCFFF10:
- **#10099 (D2Common 0x6FD82820)** sets up the sequence (READ):
  - +0x30 = records 0x6FDEBE20 (seqnum 10, every weapon class)
  - +0x34 = 8·256
  - +0x38 = 0
  - **+0x3C = 0x100**
- PD2 swaps the sequence lookup (0x6FD8282C → PD 0x10268F90), but only for seqnums 1, 6, 18 and 23. Seqnum 10 uses the stock table (READ).
- **#10819 (0x6FD83110)** then sets the rate. It uses the attack branch (`s = attackrate + EIAS − 30`, clamped 15..175, which writes +0x3C and +0x4C) only when 0x6FD80BD0 says the mode is an attack. For player mode SQ, 0x6FD80BD0 uses the mode table 0x6FDDC5F8 (mode 18: +0xC = 0, +8 = 1), so it asks the unit's current skill.
  - The skill counts as an attack only if its record byte +6 has mask[3] set. Across all PD2 skills this bit matches Skills.txt `UseAttackRate` exactly (DATA).
  - Whirlwind (151) has `UseAttackRate` blank. So do Leap Attack and Charge.
  - These skills fall through to the "other" branch 0x6FD8358C: `+0x4C = 256·clamp(stat69 other_animrate,15,175)/100`. **+0x3C is not written and stays 0x100.**

```
WW sequence rate r = 256     (one of the 8 records per frame; IAS, WSM, Fanaticism: no effect)
```

**Native check (harness/ww.c, test A):**
- Setup: the real #10099 and #10819, with the real PD2 Skills table and the real mode and skill-flag tests. Stubs: COF key, anim record, shapeshift, weapon swap, weapon-class chooser.
- Inputs: random attackrate 40..200, IAS 0..250 and other_animrate.
- Whirlwind, Leap Attack and Charge always gave +0x3C = 256. The control skills (Frenzy, Berserk, Double Swing) gave `256·clamp(ar+EIAS−30,15,175)/100`.
- 20,000 cases, 0 mismatches.
- Mutation: giving Whirlwind the attack formula mismatched 12,346 cases.

## 2. How the sequence advances on the server

The server does not step the SQ animation every frame. It schedules **timers** from the rate (READ; the scheduler itself was VERIFIED in FINDINGS round 2).

- **At the cast**, PD 0x102CCC30 (via 0x6FC98819 → 0x102ED240, same logic as stock 0x6FCFF7B0) works out positions `pos_k = k·r` for k = 1, 2, … while `pos_k < 8·256`.
  - Game frame F+k checks the records it crossed. #10153 0x6FD7E6B0 reads record byte +5 (the event).
  - Records 3 and 7 carry event 1. An END timer goes at F + max(n,1) + 1.
  - With r = 256: event at **F+3**, event at F+7, END at F+8.
- **Timer dispatch:**
  - Event timers (type 0) → 0x6FC99A10 → player tick table 0x6FD279D0[mode 18] = **0x6FC99170**.
  - END timers (type 1) → 0x6FC99850 → neutral mode.
- **Tick 0x6FC99170** (READ; stock, not patched):
  - While the skill's flag bit 0 (WW active) is set, it steps the path once (0x6FD01930 → #10342).
  - When that step finishes the path, it sets flag bit 1.
  - It then calls the do-skill path 0x6FCC2060 → PD 0x102C63C0 (a cooldown/disable pre-check) → stock 0x6FCC19A0 → **srvdofunc 76 = 0x6FC48BE0**.
- The client runs its own animation (`cltdofunc 45`). Not traced; the server decides the hits.

## 3. The do-func: why it runs every frame

`srvdofunc 76` D2Game 0x6FC48BE0 (READ; the parts marked VERIFIED ran natively):

1. **Stop check.** If the flags (#10397) have bits 0 and 1 set (active and path finished), it runs the stop routine 0x6FC48170: flags = 0, remove the state, reset the path. No hit.
   - It does not change the mode, so the unit stays in SQ until the END timer.
2. **Other exits:**
   - If flag bit 0 is clear, it exits.
   - If the unit is dead or dying (0x6FCFFA90), it exits.
3. **Player (unit type 0): D2Game 0x6FD02310(game, p = 3)** (VERIFIED)
   - Removes all of the unit's pending event and END timers.
   - Reschedules from sequence position 3·256, checking records **2** up to `(768+r)>>8` on the first step.
   - Record 3 is always in that range, so **the next event is always the next frame**.
   - Later events and END are rescheduled again on that next frame, so END keeps moving away.
   - (Monsters instead set +0x48 = 0x400.)
4. **Hit gate.** 0x6FC48CE5 is patched to call PD 0x102BED80 (via 0x102EDB10). See §4. It returns the hit count.
5. **Per hit:**
   - Pick a target with PD 0x102CB2C0 (0x6FC48D22 → 0x102F04D0). See §6.
   - Store its GUID as "last target" (#10748).
   - Run the hit: 0x6FCFE5A0 roll, 0x6FCFDDE0 damage and events.
   - Toggle skill flag 0x2000. This alternates hands with two weapons: right, then left.
   - If no target is found, "last" is set to −1 and the loop ends. The gate slot is still used up.

**How Whirlwind "loops":** it does not loop. Every frame, step 3 pins the sequence at record 3 and pushes END out. The cast lasts until the path ends. Then:
- One more event runs the stop (no hit).
- The END left over from the last reschedule arrives 4 frames later.

```
frame 0            cast (SQ start); no movement yet
frame 3            first event: first path step, first gate (always opens: next was reset to 0 at cast)
frames 3..M+1      one path step + one do-func per frame; hits on gate frames
frame M+2          the path's M-th step finishes it -> stop, no hit
frame M+6          END -> neutral          (r = 256; M = path steps needed, M >= 2)
```

## 4. PD2 hit gate 0x102BED80 (VERIFIED)

```
delay = Skills.txt Param3 (+0x150) = 5 for Whirlwind          (DATA)
if skill+0x24 != 0 and frame < skill+0x24: no hit             (#10631 get, unsigned compare)
dual  = both hands hold type-0x2E items (D2Game 0x6FC46A30 via PD ptr 0x104E50C4)
count = dual ? 2 : 1
if dual and not PvP map (levels 157/159/166, PD 0x102CEA70): delay += 1
skill+0x24 = frame + delay                                    (#11031)
```

- The WW start function (srvstfunc 38, 0x6FC48290) resets skill+0x24 = 0 at 0x6FC4848C. So the first gate of every cast opens.
- It also sets flags = 1 and last target = −1.

**Native check (test B):**
- Real code: PD scheduler, 0x6FD02310 and 0x102BED80, plus the real #10099/#10819 rate and the real #10153, #10631 and #11031.
- Replaced by C: the timer queue (insert/remove) and the tick glue (step counter; the path finishes at step M).
- Result: 6,000 casts (random frame, M 1..120, dual, PvP, stats, and a quarter of them with a forced rate 38..448), **0 mismatches** on every hit frame, hit count, stop frame and END frame.
- Mutations:
  - The old "events only on records 3 and 7" model: 6,000/6,000 mismatches.
  - A gate on every event: 5,906.
  - Dual period 5 instead of 6: 1,498.
  - First event on frame 1: 6,000.

## 5. Movement

- **Speed.** At the cast (READ), 0x6FCBE730 sets the path speed (#10488) to CharStats **WalkVelocity** (byte +0x40) << 8; Barbarian 6 → 0x600. Whirlwind's aurastat adds `velocitypercent = min(ln56, 65)` (DATA: Param5 0, Param6 2). Where the path code applies velocitypercent was not traced.
- **When the path moves.** Only in the tick, once per event (§2). So the Barbarian does not move on frames 1–2. From frame 3 he moves one step per frame.
- **Hit rate.** Hits come from a frame counter (the gate), not from distance. **Hits per second do not depend on distance.** Distance sets M, and therefore the hits in one cast:

```
single weapon:  hitsPerCast = floor((M-2)/5) + 1                    (M >= 2; M = 1 -> 0)
two weapons:    hitsPerCast = 2*(floor((M-2)/P) + 1),  P = 6 (5 on PvP maps)
cast length    = M + 6 frames (cast -> END)
```

- **PD2 changes at the start** (READ):
  - 0x6FC482F8 → PD 0x102BEE20: when the Whirlwind is aimed at a unit, the destination moves 1 subtile past it on each axis (`x += sign(x−ux)`, `y += sign(y−uy)`).
  - 0x6FC48336 is patched to `jmp`. This skips the stock branch that cancelled the Whirlwind and did a normal attack (0x6FCC2DE0) when the clicked unit was already in melee range (#11138).

## 6. Target choice: PD 0x102CB2C0 (VERIFIED, 20,000 cases)

```
r = D2Common #11133(player) = Weapons.txt rangeadder of the weapon 0x6FD70700 returns (0 unarmed)
R = r + 2 + floor(r/2)            (monster: r + 3);   on PvP maps R = min(R, 6)
candidates: unit search 0x6FCC0C70 with flags 3|0xA783, squared subtile distance <= R*R
pick: smallest squared distance, skipping the previous hit's GUID (#10179); ties keep the first found
if nothing else is in range but the previous target is, hit it again
```

| rangeadder | typical weapons (DATA) | R (subtiles) |
|---|---|---|
| 1 | knives, throwing axes | 3 |
| 2 | most swords, axes, maces, claws, Phase Blade, Berserker Axe | 5 |
| 3 | two-handed swords (Colossus Blade), Thunder Maul, most staves | 6 |
| 4 | polearms, spears, some axes | 8 (6 on PvP maps) |

**Native check (test C):**
- Real code: PD 0x102CB2C0, its callback 0x102CB250, 0x102CB180 and the PvP level test.
- Stubs: the room search (it walks a list and keeps units within the R that PD passes) and #11133 (returns the chosen rangeadder).
- 0 mismatches on both the chosen target and R.
- Mutations: dropping `floor(r/2)` gave 9,909 mismatches; not skipping the previous target gave 945.

**What this means for packs:**
- One weapon, one enemy: every hit on it.
- One weapon, two or more enemies: A, B, A, B, … (the two nearest).
- Two weapons: each gate hits the nearest and then the second nearest. With one enemy, both hits land on it.

## 7. Tables

### Sustained hits per second (while whirlwinding, targets in range; geometry fixed)

| build | map | total | 1 target | 2 targets (each) | 3+ targets (nearest two / others) |
|---|---|---|---|---|---|
| two-hander or weapon + shield | any | 5.00 | 5.00 | 2.50 | 2.50, 2.50 / 0 |
| dual wield | normal | 8.33 | 8.33 | 4.17 | 4.17, 4.17 / 0 |
| dual wield | PvP (157/159/166) | 10.00 | 10.00 | 5.00 | 5.00, 5.00 / 0 |

While moving through a pack, "the nearest two" changes, so more of the pack gets hit. The total rate does not change.

### By sequence rate / IAS / weapon

The game always uses r = 256. The table below shows every combination of the requested weapons and speed; all give the same result.

| weapon (PD2 WSM, DATA) | IAS 0 | IAS 40 | IAS 80 | IAS 120+ | any s from 15 to 175 |
|---|---|---|---|---|---|
| Colossus Blade (WSM **5** in PD2 Weapons.txt, not 10) | 5.00 | 5.00 | 5.00 | 5.00 | 5.00 |
| Phase Blade (−30), dual | 8.33 | 8.33 | 8.33 | 8.33 | 8.33 |
| Berserker Axe (0), with shield / dual | 5.00 / 8.33 | 5.00 / 8.33 | 5.00 / 8.33 | 5.00 / 8.33 | 5.00 / 8.33 |
| Thunder Maul (20) | 5.00 | 5.00 | 5.00 | 5.00 | 5.00 |

**What-if:** if the sequence did use `s` (as it would with `UseAttackRate`), only the first-hit delay and the idle tail would change. The do-func would still run every frame after the first event, and the gate would still set the rate. These values were checked natively with forced rates in test B.

| s | r = floor(256·s/100) | first hit (frames) | cast length for M = 50 | hits in that cast (1 weapon) |
|---|---|---|---|---|
| 15 | 38 | 21 | 103 | 10 |
| 50 | 128 | 6 | 64 | 10 |
| 70 | 179 | 5 | 61 | 10 |
| 100 | 256 | 3 | 56 | 10 |
| 150 | 384 | 2 | 54 | 10 |
| 175 | 448 | 2 | 53 | 10 |

### One cast (r = 256)

The "back-to-back" column assumes the next cast starts at END. The client's re-send timing was not traced.

| path steps M | 1 weapon hits | dual hits | cast → END frames | back-to-back hits/s (1 / dual) |
|---|---|---|---|---|
| 5 | 1 | 2 | 11 | 2.27 / 4.55 |
| 10 | 2 | 4 | 16 | 3.13 / 6.25 |
| 25 | 5 | 8 | 31 | 4.03 / 6.45 |
| 50 | 10 | 18 | 56 | 4.46 / 8.04 |
| 100 | 20 | 34 | 106 | 4.72 / 8.02 |

## 8. Worked example

Barbarian with a Colossus Blade (rangeadder 3 → R = 6 subtiles), Whirlwind through a pack, path of M = 50 steps (2 s at 25 fps):
- Hits land on frames 3, 8, 13, …, 48: **10 hits**. The path finishes at frame 52 and the sequence ends at frame 56.
- IAS 0 or IAS 120 gives the same frames.
- With three monsters within 6 subtiles and their distance order unchanged, the nearest two take 5 hits each.
- CB example from `crit_cb.md` (7.81% of current life per proc at 50% PR, 29% chance), per hit on one target: 0.29 × 7.81% = 2.27%.
  - Against one monster at 5 hits/s: 1 − (1 − 0.0227)^5 = **10.8%** of current life per second at any IAS. This is the top number in `crit_cb.md`; its "9.5% at s = 70 / 6.8% at s = 90–100" lines do not apply.
  - In a pack, each of the two nearest takes 2.5 hits/s: about 5.6% of current life per second each.

Two Phase Blades (dual wield, rangeadder 2 → R = 5), same path:
- Gates on frames 3, 9, …, 51: 9 gates, **18 hits**.
- One monster takes all 18. Two or more monsters: the nearest two take 9 each.

## 9. Corrections to earlier notes

- `crit_cb.md` "Rate" bullets:
  - The 4.4 / 5.0 / 3.1 hits/s figures by sequence rate came from a MODEL in which the do-func ran only when records 3 and 7 were crossed. In the real code the do-func runs every frame after the first event (§3), and the sequence rate for Whirlwind is fixed at 256 (§1).
  - "Spread round-robin across the targets" is wrong. The hits alternate between the two nearest (§6).
- Stock 0x6FC46E40 (weapon-speed delay 4–16 frames) no longer runs in PD2 (patched call at 0x6FC48CE5). In PD2, WSM affects Whirlwind neither through the gate nor through the sequence rate.

## 10. Bugs and quirks

1. **IAS and WSM have no effect on Whirlwind** (VERIFIED). **Verdict: intended but surprising.** PD2 set a fixed 5-frame delay, and the sequence ignores IAS because Whirlwind lacks `UseAttackRate`. Tooltips and calculators that show a Whirlwind IAS breakpoint are wrong.
2. **Only the two nearest enemies are hit while the geometry holds** (VERIFIED). The "not the last target" rule excludes only one GUID, so a third enemy in range is skipped until movement changes the distance order. **Verdict: unclear.** It is probably meant to spread hits, but in a tight static pack it concentrates them on two targets.
3. **Gate slots are used even with no target in range** (READ, logic VERIFIED). The gate runs before the target search. Entering range can therefore wait up to 4 frames (5 with dual) for the next slot. **Verdict: intended but surprising (minor).**
4. **Dual wield on PvP maps is faster** (VERIFIED): 5 frames instead of 6, i.e. 10 hits/s instead of 8.33. **Verdict: intended** (explicit level check 157/159/166).
5. **No movement for the first 2 frames, and 4 idle frames after the path ends** (VERIFIED timing). This is stock behaviour (events drive the path step, and END comes from the last reschedule). It lowers the hit rate of short, repeated Whirlwinds (table in §7). **Verdict: intended but surprising.**
6. **A new Whirlwind command mid-cast is accepted and resets the gate** (READ). Command acceptance for SQ (0x6FC98100) allows any command while `cur ≤ END+5`, which always holds during a Whirlwind. The start function sets skill+0x24 = 0 and the first event comes 3 frames later. A client that re-sent Whirlwind every 3–4 frames could get more than one hit per 5 frames. Whether the real client can send that fast was not traced. **Verdict: unclear.**
7. **With two weapons, a no-target gate flips which hand strikes first** (READ). The 0x2000 hand flag toggles once on the no-target exit. **Verdict: intended but surprising (harmless).**

## Unverified
- The client-side Whirlwind (cltstfunc 31 / cltdofunc 45) and when the client re-sends the command.
- How the path code applies `velocitypercent`, i.e. how many steps M a given distance takes.
- The real room/unit search 0x6FCC0C70 (stubbed; the radius test inside it, `dist² ≤ R²`, is READ).
- The timer queue and the event → tick → do-func glue (modelled in C from READ code). The dual-wield item test 0x6FC46A30 (stubbed).
