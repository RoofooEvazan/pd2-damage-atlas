# Dragon Tail kicks that animate but stop hitting — root-cause analysis

Scope: the client-shipped 1.13c `D2Game.dll` / `D2Common.dll` / `D2Client.dll` plus PD2's runtime patches in
`ProjectDiablo.dll` (PD). **Caveat:** PD2 realm servers may run different server code. Everything below describes the
server code in this install, which is what single player runs.

Status key:
- **VERIFIED**: executed natively in the harness.
- **READ**: read from the disassembly.
- **SIM**: arithmetic built from VERIFIED formulas, where the chain connecting them is READ.

## TL;DR

1. **Most likely cause (READ chain, SIM numbers): Dragon Tail's attack-speed penalty stacks on the server during a held attack.**
   - Every time Dragon Tail starts on the server, srvstfunc 27 attaches a new −20 `attackrate` stat list. That is Param4 (0x6FCB298C → 0x6FCBE7B0).
   - Normally these lists are freed only when the mode changes. `SetUnitMode` #11090 returns early for the same mode (0x6FD8396A), so the free #10196 at 0x6FD83955 is skipped.
   - After one held kick is accepted before the server's END, the server restarts KK in place. Examples of such early kicks: network jitter, or an IAS where the client and server frame counts match.
   - Each restart adds another −20, and the restart path recomputes the swing with *all* lists attached (0x6FCFFF10 → #10819).
   - The server swing gets slower on every kick. Its END moves further past the client's next re-send, so every later kick is also a restart. The state locks itself in.
   - After about 3–7 stacked kicks, the server's damage event (anim frame 4) falls *after* the client's next re-send. Every restart wipes the pending damage record (0x6FC987CA → 0x6FCFA750) and the pending event timer (0x6FC98812 → PD 0x102CDF80).
   - From then on no kick deals damage or makes its fire explosion. The client still animates at normal speed, and mana is still charged.
   - Only a server mode change clears it: moving, hit recovery, or letting the slowed (~3.5 s) server swing end. **This is why a small reposition fixes it.**
   - This mechanism is specific to Dragon Tail. It is the only kick that uses 0x6FCBE7B0.
2. **Second (READ):** with Shift/stand-still, or with no monster under the cursor, the client's `SearchEnemyXY` probe can miss the monster. It checks only 5 fixed subtiles at radius 2–3 around the player's facing (0x6FB01500).
   - When it misses, the kick is sent **on a location**.
   - The server clears the path target (#10286 / #11129(unit,0)). srvstfunc 27 finds no target and returns 0, and the server drops to neutral (0x6FCC098C).
   - The client animates anyway. This depends on geometry, so repositioning fixes it. It affects Dragon Talon the same way.
3. **Third (READ):** real reach and line-of-sight misses under Shift.
   - PD2 replaced every server-side melee-range call with its own wrapper. The start-time wrapper counts its bonus twice, so the start check is more lenient than the event-time check.
   - As a result a kick can pass the start check (hit roll, mana) and then be discarded at its event frame (0x6FCFEDAF).
4. **Not supported by the code:** knockback.
   - No knockback flag (hit flag 0x8) is set anywhere in D2Game's Dragon Tail start or do path (0x6FCB2940 / 0x6FCB3110).
   - The only kick-style knockback roll found (0x6FCB28FE) is in srvdofunc 33 (0x6FCB2770), which no PD2 Skills row uses.
   - `dragontail missile` has no knockback column.
   - PD2's damage hook for Dragon Tail (0x6FCB2A22 → PD 0x102EDA70 → 0x1026EAD0) was not fully read, so a knockback added by PD2 cannot be ruled out.
   - Even with knockback, the non-Shift path would re-approach the target, and the Shift path reduces to cause 3.

---

## 1. Server code path (Dragon Tail, Id 270)

Data: srvstfunc 27, srvdofunc 50, cltstfunc 9, cltdofunc 7, anim KK, range h2h, TargetableOnly, SearchEnemyXY, Param4 = −20 ("Attack rate penalty"), srvmissilea `dragontail missile`, ToHit 210.
AnimData `AIKK*` = 13 frames, speed 256, event frame 4 (site/ias-data.base.json).

### 1a. Command → mode start
- **Client→server packet table** at D2Game 0x6FD1A078 (8-byte entries):

  | Opcode | Handler | Meaning |
  |---|---|---|
  | 0x05 | 0x6FCEE5A0 | L-skill on location |
  | 0x06 | 0x6FCEE9D0 | L-skill on unit |
  | 0x07 | 0x6FCEE950 | L on unit, Shift |
  | 0x08 | 0x6FCEE8F0 | L on location, held |
  | 0x09 | → 0x06 path | L on unit, held |
  | 0x0A | → 0x07 path | L on unit, held + Shift |
  | 0x0C | 0x6FCEE520 | R on location |
  | 0x0D | 0x6FCEE870 | R on unit |
  | 0x0E | 0x6FCEE7F0 | R on unit, Shift |
  | 0x0F | 0x6FCEE790 | R on location, held |
  | 0x10 | → 0x0D path | R on unit, held |
  | 0x11 | → 0x0E path | R on unit, held + Shift |

- **Use skill on unit: 0x6FCEE620.** Target lookup by type/GUID (0x6FD003A0).
  - If arg4 = 1 (packets 0x06/0x09/0x0D/0x10) and the skill's range type is h2h (#10002 = 1), the server runs the melee test `#11138(player, target, #10793(target))` at 0x6FCEE74E.
  - Out of range → 0x6FCEE120: run to the target (mode RN), with the skill queued in playerdata+0x150.
  - The Shift packets (0x07/0x0A/0x0E/0x11) pass arg4 = 0, so **the server does no range check on the command**.
- **Use skill on location: 0x6FCEDF30** → 0x6FC98550.
- **Mode dispatch:**
  - Unit: 0x6FC98430 → accept 0x6FC98340 (PD 0x102CDFE0; KK: `cur ≤ END+5`, 0x6FC97E00) → mode table 0x6FD1A3F8[mode] = {XY fn, unit fn}. KK (12) = {0x6FC98840, 0x6FC987C0}.
  - Location: 0x6FC98550.
- **Unit start 0x6FC987C0**, in this order:

  | Address | Call | Effect |
  |---|---|---|
  | 0x6FC987CA | 0x6FCFA750 | delete **all** pending melee-damage records of the attacker |
  | 0x6FC987D5 | #11090 SetUnitMode(KK) | same mode → returns at 0x6FD8396A without the flag-4 stat-list free #10196 |
  | 0x6FC98804 | #11129 | path target = unit (path+0x58/0x5C/0x60) |
  | 0x6FC98809 | 0x6FCFFF10 | unit+0x44 = 0 (animation restarts); #10819 rate with the stat lists attached now |
  | 0x6FC98812 | 0x6FD00750, replaced by PD 0x102CDF80 | remove pending timers of type 0 (events) and type 1 (END) |
  | 0x6FC98818 | 0x6FCFF7B0, replaced by PD 0x102ED240 | schedule events and END (VERIFIED identical, FINDINGS) |
  | 0x6FC9882A | 0x6FCC0830 | skill start |

- **XY start 0x6FC98840** calls #10286 (path+0x10/0x12 = x,y; **path+0x58 target = 0**) and #11129(unit, 0).
- **Skill start 0x6FCC0830:**
  - 0x6FCBFC00 target validity check. TargetableOnly → hostility checks 0x6FD00A60/0x6FD00210/0x6FD00660. A dead target (unit+0xC4 bit 1, or a monster in mode 12) → target cleared.
  - Then the srvstfunc dispatcher 0x6FCBFD00 (reached through PD 0x102ED560 → the original).
  - If srvstfunc returns 0 for a player, the unit is set to neutral (0x6FCC0981–0x6FCC098C). The client is not told.

### 1b. srvstfunc 27 — 0x6FCB2940 (READ)

| Address | Action |
|---|---|
| 0x6FCB298C–0x6FCB2996 | `push [skill+0x154]` (Param4 = −20); `call 0x6FCBE7B0`. It allocates a **new** stat list: #11013 flags 4, #10807 attach (no de-duplication, 0x6FD8A800 loop only), #10188 stat 0x44 = −20, #10819 rate. |
| 0x6FCB299F | 0x6FD007A0 re-validates the path target by type/GUID and clears it if stale. |
| 0x6FCB29A8 | `#10392(path)` = target. Null or self → `return 0` (0x6FCB29B7). |
| 0x6FCB29FB | Melee hit roll 0x6FCFE5A0, redirected by PD to 0x102EEDA0 → **0x10270EB0**: hostility, a range check, AR roll 0x10271740, then block/avoid 0x1026FB60. Miss → `return 0`. |
| 0x6FCB2A22 | Damage calculation 0x6FCB2360 (PD hook 0x102EDA70). |
| 0x6FCB2A3F | 0x6FCFDDE0 → 0x6FCFAB90 stores a record `{game, attType, attGUID, tgtType, tgtGUID, damage[0x1C dwords]}` on **attacker+0xAC**. |

### 1c. srvdofunc 50 — 0x6FCB3110 (at the event frame) (READ)
- 0x6FD007A0 / #10392 target again, then 0x6FCFAA90 looks up the record by (attacker, target) type and GUID. **No record → return 0: no damage and no explosion.**
- 0x6FCB1B70 runs a secondary effect: a state/stat loop over stats 0x15E/0x15F, gated by its own #11138. It was not traced further. Then 0x6FCFED30 applies the stored damage.
- If `0x6FCFFA90(attacker) == 0`, it creates the fire area 0x6FCC0C70 (callback 0x6FCC1210) with calc1 % damage (0x6FCB31CA–0x6FCB3255).
- **0x6FCFED30** finds the record and runs `#11138(attacker, target, attackerIsMonster ? 3 : 1)` at 0x6FCFEDAF, which PD redirects to 0x10270FF0.
  - Out of range → delete the record (0x6FCFEEC6) and return: **no damage**.
  - Otherwise it applies hit effects (0x6FC57030, 0x6FCB8A70, hit reaction 0x6FCFCF20) and deletes the record (0x6FCFEEAC).

## 2. Every condition that makes a kick whiff while the client animates

| # | Condition | Where | Visible? |
|---|---|---|---|
| W1 | Server KK restarted before the event frame → record wiped and event timer removed | 0x6FC987CA, 0x6FC98812 | no |
| W2 | Kick sent on a location (no unit) → path target cleared → srvstfunc 27 returns 0 → server set to neutral | 0x6FC98840 (#10286), 0x6FCB29B7, 0x6FCC098C | no |
| W3 | Target stale or dead, or hostility check fails (TargetableOnly), at start | 0x6FD007A0, 0x6FCBFC00 | rare |
| W4 | Start-time range check (PD 0x10270EB0) fails, or the AR roll misses / is blocked, or PvP rules | 0x6FCB29FB | miss |
| W5 | Event-time range check fails, including the line test with collision mask **0x804** (0x6FD814A0 → 0x6FD80730) | 0x6FCFEDAF | no |
| W6 | Record not stored: target is not a player or monster (0x6FCFAB90) | 0x6FCFAB9A | n/a |

Normal client behaviour, which explains why the client animates anyway:
- Without Shift, the client runs the stock melee test `#11138(player, target, targetWalking)` at 0x6FACB758.
  - In range → it starts the local kick and sends 0x06/0x0D (0x6FACA060 arg 1).
  - Out of range → it walks to the target (0x6FADA7C0), which is itself a reposition.
- With Shift, the client kicks regardless of range (0x6FACB7A3 → 0x6FACA060 arg 0).

## 3. Top cause — W1 made persistent by attack-rate stacking (Dragon Tail only)

### Evidence chain (READ)
1. **Held attacks.**
   - The client re-sends when its local swing ends (FINDINGS "Held attacks"). In the loop order, server packets are processed before server timers in the same frame (FINDINGS; D2Client 0x6FAF4B50).
   - A re-send arriving at or before the server END is therefore accepted in KK (0x6FC97E00: `cur ≤ END+5`). It restarts through 0x6FC987C0.
2. **The first server kick is scheduled without the penalty.**
   - 0x6FCFFF10 and 0x6FCFF7B0 run before 0x6FCC0830, and 0x6FCC0830 is what attaches the −20.
   - The client applies the penalty from its first frame (cltstfunc 9 → 0x6FB50880).
   - So in steady state the server finishes first (server n ≤ client n). Unless an IAS breakpoint makes them equal, the server returns to neutral, which frees the list.
3. **One early kick starts the chain.**
   - One re-send that arrives early (MP jitter), or equal frame counts, gives a same-mode restart. #11090 keeps the first −20 list and srvstfunc 27 adds a second (0x6FCBE7B0 has no replace logic).
   - 0x6FCFFF10 now schedules with −20. The server swing equals the client's, and the next re-send arrives at END (packets before timers) → another restart → −40.
   - From then on the server is slower than the client, so every re-send lands before END. It stacks further, down to the 15 floor of the 0x6FD83110 clamp.
4. **Kicks stop landing.**
   - With k lists, the server event (frame 4) comes at ceil(1024 / rate(s0 − 20k)) frames.
   - Once that is later than the client's re-send period n_c, each restart deletes the pending record and event before it fires (W1).
   - srvstfunc 27 still succeeds on every restart, so mana is charged and the hit is rolled, but nothing is applied.
5. **How it clears.** Only a server mode change clears it:
   - A walk/run command. KK accepts any command while `cur ≤ END+5`, which always holds during a slowed swing. It leads to #11090 with a different mode → #10196 frees the flag-4 lists.
   - Getting hit into GH.
   - Or not kicking until the slowed END fires (~87 frames ≈ 3.5 s at s = 15).
   - Short button releases do not clear it, so to the player it looks like "only repositioning fixes it".

### SIM (VERIFIED formulas)
Inputs: rate = ⌊256·clamp(s,15,175)/100⌋ (VERIFIED 0x6FD83110); n = #{k≥1: k·rate < 13·256} (VERIFIED scheduler); event frame 4.

| s0 (100 + EIAS − WSM, before DT) | client re-send period n_c | server first kick n | server n equal after the first restart? | stacked lists before kicks stop landing |
|---|---|---|---|---|
| 100 | 16 | 12 | yes | 4 |
| 135 | 11 | 9 | yes | 5 |
| 150 | 10 | 8 | yes | 6 |
| 175 | 8 | 7 | yes | 7 |
| ≥185 | 7 | 7 | **locks with no jitter at all** | 7 |

- Below s0 ≈ 185, locking in needs one re-send that arrives n_s − n_c = 1–4 frames early.
- After 2–3 stacked lists the tolerance for late packets grows (e.g. s0 = 150: tolerance 1, 4, 8 frames at k = 2, 3, 4). The state then holds.
- This matches "occasionally, after standing and kicking for a while".
- Scratch simulator: /tmp/claude-0/sim.py.

Confidence: **READ** for the chain. The individual formulas are VERIFIED (FINDINGS: rate, scheduler, acceptance). The full restart and stacking loop was **not** executed natively; that would need the stat-list allocator and the timer queue.

Side notes:
- Berserk (srvstfunc 32, the same helper) stacks in the positive direction on the server. That makes it faster and harmless for hits.
- skill_speed.md had already flagged the stacking caveat ("Dragon Tail −40"). This analysis closes it.

## 4. Second cause — W2, the SearchEnemyXY probe (Dragon Tail and Dragon Talon)

- Client target resolve **0x6FB01AE0**.
  - It keeps the hovered unit from 0x6FB01A80, which reads the selection globals 0x6FBC964C/0x6FBC9638.
  - With no unit, and `SearchEnemyXY` set (Skills +0x6 bit 0x20; `test [gdwBitMasks+0x14]` at 0x6FB01CC9), it calls **0x6FB01500**. This is gated on an argument from 0x6FAC9350: Shift, or a right-skill with no target.
- **0x6FB01500** probes only 5 subtiles: directions facing + {0, −1, +1, −2, +2} octants (0x6FB85EC8), at offsets dx/dy from tables 0x6FB85F04/0x6FB85EE4. Those offsets are (0,3), (−2,2), (−3,0), (−2,−2), (0,−3), (2,−2), (3,0), (2,2).
  - At each probe it checks unit collision `#10573(room, x, y, 1, 0x180)` (player|monster), then `#10389` finds the unit there.
  - A monster standing on any other subtile (for example (1,3), or 4 or more subtiles away) is not found.
- With Shift, a failed probe sends the skill **on a location** (0x6FACB909 → 0x6FACA060 arg 0) and plays the kick locally. Without Shift, the h2h skill walks to the location instead (0x6FACB8C7 → 0x6FACA010).
- Server: location handler → 0x6FC98840 clears path+0x58 → 0x6FCB29A8 target null → srvstfunc 27 returns 0 → neutral. **No hit, no explosion.**
- Repositioning moves the monster onto a probe subtile. Confidence **READ**.

## 5. Third cause — reach and line of sight under Shift (W4/W5)

- **#11138 (0x6FD826C0)** — **VERIFIED**, 16,473 cases, 0 mismatches:
  - In range iff d ≤ attackerRange + bonus + 1, where d = size-adjusted distance #10407 (0x6FDCFCD0) on integer subtile coordinates (path+2/+6).
  - For 0 < d ≤ limit, a line test also runs (mask 0x804: barrier 0x4 | door 0x800). The harness bypassed it with a null room.
  - A player's range comes from #11133 = the weapon's `rangeadder` (item record +0x104). It is 2 for every claw in Weapons.txt.
  - Harness: /tmp/claude-0/mr/mr.c.
- **PD2 wraps every server call** of #11138: 41 of 42 sites, all except 0x6FCC7FCB. The wrapper is 0x10270FF0:
  - Player-owned attacker: bonus += ⌊range/2⌋ + #10143(target is walking).
  - Plain monster: bonus += 1.
- **No client site is redirected.** The client keeps stock #11138 at 0x6FACB758 and 9 other sites.
- **The PD2 hit-roll replacement 0x10270EB0 counts the bonus twice.** It adds `arg+2 + range/2 + walking` itself, then calls 0x10270FF0, which adds `range/2 + walking` again.

Resulting limits for claws, stationary target (READ arithmetic on the VERIFIED formula):

| Check | Where | Limit |
|---|---|---|
| Client | 0x6FACB758 | d ≤ 3 |
| Server command (0x06/0x0D) | 0x6FCEE74E | d ≤ 4 |
| Server start and hit roll | 0x6FCB29FB | d ≤ **7** |
| Server event | 0x6FCFEDAF | d ≤ **5** |

- Without Shift, the client never kicks beyond 3, so this cannot whiff.
- With Shift, targets at d = 6–7 pass the start (mana is charged and the hit rolled) but are discarded at the event.
- A kick can also be discarded when a 0x804 cell sits on the line.
- Monsters approach until their own AI check passes (e.g. `MeleeRng + 2` after the PD2 +1). Long-reach monsters (MeleeRng ≥ 4) can stand where they hit you but a Shift-kick cannot reach them.
- Confidence: **READ** for the PD wrapper; **VERIFIED** for #11138 and #10407.

## 6. Hypotheses checked and ranked

| Rank | Hypothesis | Verdict |
|---|---|---|
| 1 | Server attack-rate stacking → restart wipes the event (§3) | **Most likely.** Fits "same spot, extended time", "animations continue", "reposition fixes". Dragon Tail only. READ + SIM. |
| 2 | SearchEnemyXY probe miss → kick on location (§4) | Likely contributor with Shift or the cursor on the ground. Also explains Dragon Talon. READ. |
| 3 | Real reach or line block under Shift, with PD2's start/event mismatch (§5) | Possible for long-reach monsters or door/barrier cells. READ/VERIFIED. |
| 4 | Knockback pushes the target out | **Not supported.** No knockback flag is set in Dragon Tail's D2Game path. The only roll found (0x6FCB28FE) belongs to the unused srvdofunc 33. PD2's damage hook 0x1026EAD0 was not fully read. |
| 5 | Client/server position desync | Not needed for the symptom. The client is stricter than the server, so a desync would make the server *run* the player (0x6FCEE120), which is visible. Not established. |
| 6 | Dragon Flight | range `rng`, SearchEnemyNear, srvdofunc 178 (PD-extended table). Not a melee-reach path. Not analysed further. |

## 7. Suggested fixes for PD2

1. **Stop the stacking** (fixes §3):
   - In 0x6FCBE7B0 or srvstfunc 27, remove or replace the previous Dragon Tail attack-rate list before attaching a new one. For example, tag it with a state and use the state-replace path.
   - Or, in the same-mode restart branch of 0x6FC987C0, call #10196 (0x6FD8AE60) to free flag-4 lists before 0x6FCFFF10.
   - Also apply the penalty **before** the schedule is built (0x6FCFF7B0), so the server's first swing matches the client's.
2. **Do not drop an unfired kick on restart:** fire or keep the pending event and record. At minimum, skip 0x6FCFA750 and the event-timer removal when the pending record belongs to the same skill and target.
3. **SearchEnemyXY:**
   - Server side: for h2h SearchEnemyXY skills on a location, search for the nearest hostile within melee reach of the XY (as SearchEnemyNear does) before failing.
   - Client side: probe the whole ring d ≤ reach, not 5 points.
   - Or do not play the local kick when no target is found.
4. **Range:**
   - Remove the double-counted bonus in 0x10270EB0: call stock #11138 there, or drop its own `range/2 + walking`.
   - Make the start and event bonuses equal, so a kick that is rolled and paid for is also applied.
   - Consider hooking the client's #11138 the same way, so Shift/no-Shift behaviour matches the server.
