# Strafe and "chance to cast" procs (PD2 = 1.13c + ProjectDiablo.dll)

Question: "What frame in Strafe triggers 'on hit' procs?"

**Answer.** No animation frame triggers them directly.
- The animation's event frame fires one arrow per swing.
- Each arrow rolls "chance to cast on striking" (stat 198) when it collides with a target and the hit connects. That happens on the server frame the arrow reaches the target, per arrow and per target struck.
- "Chance to cast on attack" (stat 195) never triggers from Strafe. The event it listens to for missiles is never raised anywhere in D2Game or ProjectDiablo.dll.
- "On crit" (stat 203) is not wired to any fire site either, and no item in the data grants it.

Status key: **VERIFIED** means native execution (the harness). **READ** means from the disassembly only.

---

## 1. Arrow release: srvdofunc 12 = D2Game 0x6FC6DCD0 (stock, not replaced by PD2) — READ

PD2's srvdofunc overrides (0x102BE412, table 0x6FD274A8 via lazy slot 0x104E3114) write offsets 0x10, 0x18, 0x20, 0x64, … but not +0x30. So index 12 stays stock 0x6FC6DCD0.

The do-func runs once per animation event (the AnimData event frame; `AMA1BOW` = 14 frames, event frame 6). Each call does the following:
1. Reads the remaining count (skill param, 0x6FC2AD12). It stops if the count is 0.
2. Resolves the current target (0x6FD003A0). If the target is gone, it picks a new one with 0x6FCC0F40 (mode 3).
3. Builds missile params on the stack and calls the missile create function **0x6FC8F930 once**:
   - flags 0x21, owner, missile = Skills `srvmissilea` (+0x48, `strafearrow`) or `srvmissileb` (+0x4A, `strafebolt`)
   - target coordinates, skill 26 and level
   - callback 0x6FC6C190, which adds Strafe's damage % (calc +0x13C) to the arrow's stat 25
4. `count − 1` → 0x6FC2AC94. If the count is still > 0, it finds the next target (0x6FCC0F40, excluding the current one) and restarts the animation: 0x6FCC35D0 → 0x6FD02600, which PD replaces with 0x102CD0B0. That call rolls the animation back to 50% (Param6).
5. On the last arrow there is no restart, and the animation plays to its end.

So **arrow i is created on the server frame of the i-th animation event**. The frame numbers come from the server schedule, which FINDINGS "Follow-up swings" verified natively (harness/fup.c, 40k setups; schev.py 20k).

## 2. Proc registration: item stats → unit event list — READ

- ItemStatCost (compiled .bin, record 0x150; stat id at +0, itemevent at +0x48/+0x4A, itemeventfunc at +0x4C/+0x4E):

  | stat | name | events | func |
  |---|---|---|---|
  | 195 | `item_skillonattack` | 7 `domeleeattack`, 8 `domissileattack` | 20 |
  | 198 | `item_skillonhit` | 5 `domeleedamage`, 6 `domissiledamage` | 20 |
  | 203 | `item_skilloncrit` | 16 `docrit` | 30 |

  Event numbers are Events.txt row order. They were cross-checked against every fire site (5/1 melee, 6/2 missile, 7/3 melee attack, 9/10 kill/killed, 13 corpsefinish).
- The stat-change callback D2Game 0x6FCF9470 registers one node per (stat, layer) on the unit's list at unit+0x90, through 0x6FCBE290 → 0x6FC57190. The node's +0xC = stat<<16 | layer. The node function comes from table **0x6FD277A8**.
  - The existing-node check 0x6FC570E0 means the same skill+level from two items is one node. Its chance is the summed stat value.
  - Different skill/level pairs are separate nodes and roll independently.
- Func 20 = **D2Game 0x6FCCDDC0**. PD2's table patch 0x102BEA10 replaces funcs 6, 7, 15, 16, 19, 22–25, 27, 32 and 33, but **not** 20 or 30.

## 3. Where an arrow's on-striking proc fires — READ

Chain, one pass per arrow per target struck:

1. **Missile collision: PD 0x102723E0**. PD replaced stock 0x6FC5EDE0 at all 12 callers through wrapper 0x102F1010.
   - `strafearrow` has Missiles.txt `ToHit` = 1, so it rolls the hit: 0x10271740(owner, target, 0x10268220(missile, owner), isMissile 1). See hit_pd2.md.
   - **Miss** → only event 0 `hitbymissile` goes to the target (0x1026F070 → 0x6FC57030). **No proc.**
   - **Hit** → SrvHitFunc (none for strafearrow) → memset damage → 0x1026E9F0 (missile damage from its stats, stock 0x6FC5A4E0) → SrvDmgFunc (none) → 0x1026E990 → D2Game 0x6FC5AFF0 → patched call 0x6FC5AFF7 → **PD 0x1026E7D0**.
2. **PD 0x1026E7D0**, the missile damage and events function (replaces stock 0x6FC5AE10):
   - Sets the hit bit (dmg+4 |= 1).
   - Block/avoid 0x1026FB60. Any nonzero result (block, dodge, avoid, evade, weapon block) clears the hit bit.
   - dmg+0 |= 0x20 if the missile's flags (#10623) & 1, and |= 0x80 if & 2.
   - Then, **only if the hit bit is still set**: 0x102ED2E0 → stock **D2Game 0x6FCFE0C0**(attacker = missile owner, defender, bMissile = 1, dmg). After that the damage is applied (0x1026EDD0).
   - So the proc is rolled on the same server frame as the arrow's damage, just before the damage is applied.
3. **D2Game 0x6FCFE0C0** (stock; no PD patch in 0x6FCFE1A5–0x6FCFE1F7):
   - Requires the hit bit, and no result bit 0x20.
   - For missiles it also requires **dmg+0 & 0x20**.
   - Then event **6 `domissiledamage`** (attacker), unless dmg+0 & 0x80.
   - Then event 2 `damagedbymissile` (defender).
   - Dispatcher 0x6FC57030 walks unit+0x90.
4. **Why Strafe arrows carry flag 0x20.**
   - The missile create function calls 0x6FC8F620 → D2Common #10413. That call site (0x6FC8F63D) is patched to **PD 0x1026B090**, and then goes to #10484, which copies damage-data flag 1 into missile flag 1 (D2Common 0x6FDBB4DD).
   - PD 0x1026B090 sets damage-data flag 1 when the missile's `SrcDamage` (+0x12D) ≠ 0. For missiles with a `Skill` column, the skill's SrcDam (+0x1A5) is used instead.
   - `strafearrow`/`strafebolt` have SrcDamage 128, so the flag is set.
   - Missile flag 2 (params flag 0x10000 at 0x6FC8FE74, or damage-data bit 4) is never set for Strafe (params flags 0x21; PD never sets bit 4). So `domissiledamage` is not suppressed.
5. **Func 20 = 0x6FCCDDC0**(attacker, target, dmg, stat<<16|layer):
   - chance = #10910(attacker, 198, layer). This is read **at hit time**, not at release.
   - skill = layer >> 6, level = layer & 63 (shift and mask from the data tables at +0xC6C/+0xC70).
   - Refuses if dmg ≠ NULL and !(dmg+0 & 0x20).
   - Roll: stock LCG inline on the **attacker's** seed (new = lo·0x6AC690C5 + hi, 64-bit). It is *not* PD's generator 0x102C5D10. Proc if (new lo mod 100) < chance.
   - **Cast:**
     - With a target: 0x6FD117E0(caster = attacker, target = struck unit, skill, level, 1). **The player casts at the struck unit.**
     - Skills whose flag byte +7 bit 0x20 is set (only Teleport, Blink and 4 monster novas/blinks) are cast by the struck unit on itself.
     - No target: 0x6FD11730 casts at the attacker's position.
6. **PD2 limits.**
   - PD patches 0x6FD1181D (`test` → `xor`), which removes the stock "caster has state 54 `uninterruptable` → no proc" check.
   - The executor call 0x6FD116B4 → PD 0x102C63C0 checks per-skill cooldown (playerdata+0x1D0) only when that last argument is 0. On-striking casts pass 1, so there is **no cooldown or cap** on this path.
   - No per-sequence or per-frame proc limit was found.
   - Each arrow and each target is an independent roll. A piercing arrow rolls again on each further target it hits.

**Consequences for Strafe**
- Missed arrows (failed AR roll), blocked arrows and dodged/avoided/evaded arrows never proc.
- Every connecting arrow rolls every on-striking node once.
- The proc happens on the arrow's collision frame, which is its release frame plus travel time.
  - `strafearrow`: Vel 30 (+0x9A), Range 40 (+0x96). Range + LevRange·lvl is set as the missile's frame count through #10558/#10970 (0x6FC8FC85), so an arrow that hits nothing within 40 frames expires.
  - Travel time itself depends on distance and was not converted to frames here.

Crushing blow (func 16 → PD 0x102AF610) and open wounds (func 15 → PD 0x102AF060) are nodes on the same events 5/6. They fire at the same point, on every connecting arrow, and CB also requires dmg+0 & 0x20. Their formulas are in damage.md.

## 4. On attack (195), on crit (203) — READ

- **Every** fire site was enumerated:
  - D2Game `call 0x6FC57030`: 16 sites, with events 0, 0, 13, 10, 9, 11, 3, 12, 6, 5, 1, 10, 9, 12, 7, 3.
  - PD: 0x102ED3E0 is the only user of slot 0x104E4E38 (→ 0x6FC57030). Its 5 callers use events 0, 9, 10, or the timer callback 0x102BECF0, which is registered only with 14 (dospellcast) and 15 (doblock).
  - PD's filtered dispatcher 0x102B0160 is called only with 5 and 1, from the area/splash callback. It skips funcs 20/21/30 for splash.
  - These are the only two walkers of unit+0x90 in either module.
- **Event 8 `domissileattack` is never raised.** So 195 cannot proc from any missile attack, Strafe included, neither per arrow nor per skill use. Event 7 `domeleeattack` is raised only by the melee damage routine 0x6FCFED30 (0x6FCFEE4B), which Strafe does not use.
- **Event 16 `docrit` is never raised.** PD's missile crit (0x6FC5A730 → 0x10270C50 → 0x10270E00) only sets result flag 0x2000 and doubles damage. No property in UniqueItems, Runes, SetItems or Magic affixes uses `crit-skill` (stat 203) or `pierce-skill` (205).

## 5. Worked example: Amazon, 5 arrows (Strafe level ≥ 4, ≥ 5 targets)

Setup:
- `AMA1BOW`: 14 frames, event frame 6, bow start frame 0, Param6 = 50.
- speed = 100 − WSM + EIAS; rate = 256·speed/100.
- Frames are game frames counted from the frame Strafe starts (25 per second).
- Schedules come from the calculator's clientCycle/serverHits, which are natively verified.
- Note: `amb` is the Matriarchal Bow (WSM −10); the Grand Matron Bow is `amc` (WSM +10).

| Bow, total IAS | rate | client arrow frames | **server arrow frames (arrow created)** | cycle |
|---|---|---|---|---|
| Matriarchal `amb`, 0 IAS | 281 | 6, 9, 12, 15, 18 | **6, 9, 12, 15, 18** | 25 |
| Matriarchal `amb`, 20 IAS | 325 | 5, 8, 11, 14, 17 | **5, 9, 12, 15, 18** | 22 |
| Matriarchal `amb`, 40 IAS | 358 | 5, 8, 11, 14, 17 | **5, 8, 11, 14, 17** | 21 |
| Matriarchal `amb`, 60 IAS | 384 | 4, 6, 8, 10, 12 | **4, 7, 10, 13, 16** | 17 |
| Grand Matron `amc`, 40 IAS | 307 | 6, 9, 12, 15, 18 | **6, 9, 12, 15, 18** | 24 |

Worked case: Matriarchal Bow at 20 IAS.
- Your client shows the releases on frames 5/8/11/14/17.
- The server creates arrows 1–5 on frames 5/9/12/15/18. The server copy is the one that matters, because procs are server events.
- Arrow i can proc on-striking only on frame (server frame i + travel frames to its target), and only if its AR roll hits and the target does not block, dodge, avoid or evade.
- Each such arrow rolls each on-striking item node once (e.g. a 100% proc goes off five times if all five arrows connect).
- On-attack items never proc.

## Address index
- srvdofunc 12: D2Game 0x6FC6DCD0 (table 0x6FD274A8). Missile create: 0x6FC8F930. Restart: 0x6FCC35D0 → 0x6FD02600 / PD 0x102CD0B0.
- Collision: PD 0x102723E0. Missile damage/events: PD 0x1026E7D0 (patch 0x6FC5AFF7). Events: 0x6FCFE0C0. Dispatcher: 0x6FC57030. Registration: 0x6FCF9470 / 0x6FCBE290 / 0x6FC57190.
- Func 20: 0x6FCCDDC0. Cast: 0x6FD117E0 → 0x6FD114F0 → PD 0x102C63C0 (patch 0x6FD116B5). PD byte patch: 0x6FD1181D.
- Missile damage data: D2Common #10413 → PD 0x1026B090 (patch 0x6FC8F63E). Flag copy: #10484 0x6FDBB4A0. Flag getter: #10623.
- PD event-func table patch: 0x102BEA10. PD filtered dispatcher: 0x102B0160 (sets 0x104E1EA8 / 0x104E1F4C / 0x104E1E48, built at 0x100EF3F0–0x100EF590).
