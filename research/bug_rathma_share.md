# Rathma + Mendeln shared damage — why poison is not shared (and every other hole)

Scope: client-shipped 1.13c `D2Game.dll` / `D2Common.dll` plus PD2's runtime code in `ProjectDiablo.dll` (PD, base 0x10000000).
Same caveat as the other reports: realm servers may run different server code; this is what single player runs.

Status key: **VERIFIED** = PD2's own code executed natively in the harness and matched the model on every case.
**READ** = read from disassembly. **INFERRED** = follows from READ facts but depends on runtime behaviour not traced.

## TL;DR

1. **Mechanism (VERIFIED, 40,000 native cases, 0 mismatches).** There is no shared life pool.
   - PD2 put the sharing inside its per-damage-type resist function **PD 0x1026F410**, which replaces stock 0x6FCFBD00 in the resist loop (patch at D2Game 0x6FCFC4E9 → PD 0x102ED620 → 0x1026F410).
   - After resist/DR/absorb, when the defender's MonStats id is **0x3A7 `rathmaBoneClone` or 0x3A8 `rathmaPoisonClone`** and both game slots `game+0x2600` and `game+0x2604` are non-null, it does three things:
     - it halves the value: `v = min(trunc(v·50.0/100), INT_MAX)` (constant 0x1037F808 = 50.0);
     - it writes the halved value back into the damage struct, so the defender takes half;
     - it sets the partner's life directly: `SetStat(partner, 6, max(life − v, 0))` (#10887).
   - The partner is skipped when it has state 8 (`resistall`, the Salvation aura state; #10494 check).
2. **Why poison is not shared (VERIFIED arithmetic, READ path).**
   - The share block runs once for every entry of the type table 0x6FD22AB0, including entry 7 (poison *length*, +0x2C) and entry 8 (poison *rate per frame*, +0x28).
     - Both get halved, so the defender's poison state gets rate/2 for len/2 frames. **The defender takes 25 % of the poison.**
     - The partner loses rate/2 exactly once: one frame's worth of the halved poison, plus `len/2` 1/256-points from the length entry.
   - The poison itself is applied only to the defender, by PD's replacement poison-apply **PD 0x10271A70** (patched in at D2Game 0x6FCFE2AE). It has no Rathma branch.
   - The poison then ticks as negative `hpregen` (stat 74) through the monster regen timer **D2Game 0x6FC96740**. That timer never enters the damage function or 0x1026F410.
   - PD2 *did* write a proper DoT share for open wounds (PD 0x102AF355–0x102AF5D4), but nothing equivalent exists for poison.
3. **Worst other bugs (READ):**
   - **The slots are never cleared when a clone dies** (death case 0x102C1C86). After the first kill the survivor still takes only 50 % of every hit, and the other 50 % goes into the corpse.
   - **Melee is shared at swing start, not at impact.** A swing that rolled a hit but never lands its damage (attacker interrupted or restarted before the hit frame, or the target out of reach by then) still costs the partner 50 %. Misses and blocked, dodged, avoided or evaded attacks are *not* shared: see §6.
   - **Crushing blow is not shared.**
   - **Shared damage can leave the partner alive at 0 HP.**

---

## 1. The fight and the slots (READ)

| What | Where | Detail |
|---|---|---|
| Spawn, level 161 Necropolis Jungle | PD 0x102C729E | Mendeln `rathmaPoison` 0x3A6 alone (phase byte `game+0x262A` 0→1) |
| Spawn, level 162 Necropolis Swamp | PD 0x102C734E | Rathma `rathmaBone` 0x3A5 alone (2→3) |
| Spawn, level 163 Necropolis Void | PD 0x102C74AA / 0x102C74C8 | **`rathmaPoisonClone` 0x3A8 at (0x4E97,0x1403) and `rathmaBoneClone` 0x3A7 at (0x4E9D,0x1403) together** (4→5). These two are the pair that is "fought together". |
| Slot registration | AI PD 0x102B2010, branch 0x102B2560 (all four ids 0x3A5–0x3A8 reach it via 0x102B21F8–0x102B2236) | `if class==0x3A8: game+0x2604 = self else game+0x2600 = self`. This runs on every AI tick. |
| Clone death | death dispatcher PD 0x102C11F0, table 0x102C519C/0x102C51D0 → case **0x102C1C86** for both 0x3A7 and 0x3A8 | 1st death: phase 5→6, and the surviving clone gets skill 0x1E5 (485 `RathmaDeath`) through 0x102BFE20. 2nd death: phase 6→7, and golems/totems (0x3A9–0x3AB, plus 0x3C8/0x3C9/0x452/0x453) are killed. **Neither branch touches `game+0x2600/0x2604`.** |
| Slot clearing that does exist | 0x102C20CC (class 0x2E7), 0x102C3328 (0x48E), 0x102C3380 (0x451), 0x102C33DC (0x450) | These belong to other bosses that reuse the same `game+0x2600..` scratch slots. |

## 2. The share block — PD 0x1026F410 (VERIFIED)

```
0x1026F419  ent = type-table entry, ctx = {game, diff, att, def, ..., dmg@+0x18}
            v = dmg[ent.off]; if v < 1 → dmg[ent.off]=0, return 0
            v = resist/pierce/DR/absorb (0x1026F680, 0x1026EA70, 0x1026F820)   (VERIFIED earlier, damage.md §2)
0x1026F4E9  dmg[ent.off] = v
0x1026F4EB  player-owner damage credit: playerdata+0x1A8 += min(v>>8, life>>8)  (uses the UNHALVED v)
0x1026F5E9  if def.type==1 && def.class in {0x3A7,0x3A8} && game+0x2600 && game+0x2604:
0x1026F615      v = 0x102CE840(v, 100, 50.0)        // trunc(v*50/100), capped at 0x7FFFFFFF
0x1026F62F      dmg[ent.off] = v
0x1026F635      partner = game+0x2604; if partner.class == def.class: partner = game+0x2600
0x1026F64C      if !GetState(partner, 8):           // #10494, state 8 = resistall (Salvation)
0x1026F66E          SetStat(partner, 6, max(GetStat(partner,6) − v, 0), 0)   // #10973 / #10887
            return v
```

**Native test:** `/tmp/claude-0/rshare/share.c` + `drv.py`.
- It ran the real PD 0x1026F410 with the real 0x1026F680/0x1026EA70/0x1026F820/0x102CB180/0x102CE840 and the D2Game type table.
- Stubbed: stat get/set, #10494, #10871, #10680.
- Coverage: 40,000 random cases, 0 mismatches. The cases span entries 0–8, v up to 2³¹−1, defender classes 0x3A5–0x3A8, each slot present or missing, state 8 on or off, and partner life 0 to 2³¹−1.
- Every case matched: defender gets v/2, the partner's HP is set to max(life − v/2, 0), nothing happens for 0x3A5/0x3A6 or when a slot is null, and the partner is not touched when state 8 is set.

Properties that follow directly:
- **Split, not mirror (VERIFIED).** Each hit is divided 50/50. The pair's combined loss equals one un-shared hit.
- **The partner is charged with the *defender's* resistances, pierce, DR and absorb (VERIFIED).** Its own resistances are never read.
  - The clones differ (Hell): fire 75 vs 65, cold 50 vs 40, lightning 50 vs 40.
  - Hitting Mendeln with elemental damage therefore damages Rathma at Mendeln's lower resist.
- **There is no attacker filter (READ).** The block reads only `ctx.game` and `ctx.def`. Players, summons, mercs, other monsters and self-damage are all shared.
- **There is no distance, room or alive check (READ).** The partner can be anywhere, in an inactive room, or dead.
- **Overflow (VERIFIED).** It is safe. 0x102CE840 computes in double and saturates at 0x7FFFFFFF (compare 0x1037F890 = 2147483647.0). The partner's value is clamped at 0 with `cmovns`.

## 3. Poison, exactly (VERIFIED halving + READ path)

The resist loop at D2Game 0x6FCFC4CC–0x6FCFC4F5 calls the per-type function for table entries 0…8. It goes on to 9–11 only when `ctx+0x10` (attacker is a mercenary, #11104) is set.

| entry | field | value | what the share block does to a clone |
|---|---|---|---|
| 5 | +0x30 cold length | frames | halved; partner −len/2 (in 1/256 HP) |
| 6 | +0x34 freeze length | frames | halved; partner −len/2 (in 1/256 HP) |
| 7 | +0x2C **poison length** | frames | **halved**; partner −len/2 (in 1/256 HP) |
| 8 | +0x28 **poison rate** | 1/256 HP per frame | **halved**; partner −rate/2 **once** |

Then, in the execute function 0x6FCFE0C0:
- Life is subtracted at 0x6FCFE242–0x6FCFE264. This covers only phys + fire + light + magic + cold (the total is built at 0x6FCFC4F7). Poison is not included.
- Poison is applied at 0x6FCFE2AE. The patch record `{3, 0xDE2AF, 0x10271A70, rel}` points it to **PD 0x10271A70**:
  - it finds or creates state 2 on the **defender only** (per-player state via 0x10269230);
  - it sets `hpregen = −rate` and the state length to `len`;
  - a new application replaces the old one only when its drain is at least as strong.
- Ticks: state 2's negative stat 74 is applied by the monster regen timer **0x6FC96740** (`life += hpregen`, death when life < 1). No damage struct, no resist loop, no 0x1026F410.

Result for a poison hit of total T = rate·len on a clone:

|                   | intended "shared" | actual |
|---|---|---|
| defender          | T/2 | **T/4** (rate/2 for len/2 frames) |
| partner           | T/2 | **rate/2 = T/(2·len)**, instantly (e.g. 0.5 % of T for a 4-second poison) |

So poison is not shared, and in addition the clone that was poisoned takes only a quarter. This is the reported bug.

**The exact check responsible** is 0x1026F5F5–0x1026F613. It is a class and slot test with no test of the damage type (`ent`), so a DoT rate and a duration are treated like instant damage. The other half of the cause is that 0x10271A70 has no partner branch.

**Contrast with open wounds (READ).** PD 0x102AF060 at 0x102AF347–0x102AF3B8 has its own clone test with the same slots and the same 50.0. It halves the OW drain and then **creates or stacks OW state 62 on both** the defender (0x102AF4FA) and the partner (0x102AF520–0x102AF5D4), skipping a partner that has state 8. This is how poison should have been done.

## 4. Other paths

| Path | Shared? | Evidence | Status |
|---|---|---|---|
| Missiles (incl. novas, AoE, corpse explosion–type area damage) | Yes, at impact | missile hit 0x6FC5AFCA calls execute with resist flag 1 | READ (CE not individually traced: INFERRED) |
| Melee (normal attack, all srvst melee skills) | Yes, but **at swing start** | srvst 1 → 0x6FC64AC0: hit roll 0x6FC64BE5, then store 0x6FC64C06 → 0x6FCFDDE0, which runs the resist loop at 0x6FCFDE2D. The record is applied later by 0x6FCFED30 → execute with flag 0 (0x6FCFEDF8) | READ |
| Crit / deadly strike | Yes | applied to phys in the fill step (PD 0x10270E00), before the resist loop | READ |
| Thorns (stat 78/128) | Yes | item event 6 = PD 0x102AEE30 → 0x102ED2E0 → execute 0x6FCFE0C0 with flag 1 | READ |
| Summons / merc / other monsters | Yes | no attacker test in the block | READ |
| Merc leech % (+0x38/+0x3C/+0x40, entries 9–11, merc attackers only) | Halved; partner loses the % value as 1/256 life (negligible) | table entries 9–11 have `+0x1C=1`, gated on #11104 | READ |
| **Open wounds** DoT | Yes (DoT on both, drain halved) | 0x102AF347–0x102AF5D4 | READ |
| **Poison** DoT | **No** (see §3) | 0x10271A70, 0x6FC96740 | VERIFIED halving / READ path |
| **Burning** (+0x14/+0x18 → 0x6FCFC940, state 115, hpregen) | **No.** Not in the type table, so it is neither resisted nor shared. | 0x6FCFE2BD | READ. In this data only the monster skills UberTalicBlaze and ZharNovaOrb have fire ELen, so there is practically no player source. |
| **Crushing blow** | **No** | PD 0x102AF610 lowers the defender's life directly during on-hit events (after the resist loop). There is no clone or slot code in 0x102AF610–0x102AFA40. | READ |
| Life leech from the hit | Only on the defender's half | leech (0x6FCFE218 → PD 0x102700F0) uses dmg+0x08 after the halving; the partner's half gives no leech | READ |
| Damage credit (playerdata+0x1A8) | Credits the **unhalved** value | credit at 0x1026F4EB runs before the halving at 0x1026F615 | READ |
| Static Field | n/a | PD 0x102FC760 passes target filter 0x1026F150, which rejects 0x315, 0x3A5–0x3A8 and 0x458 before the stock %-life filter 0x6FC62EC0, so it never hits them | READ |
| Self-damage with resist flag 0 (0x6FC6F4A9 Paladin, 0x6FCB9371) | No resist loop, so not shared | callers of 0x6FCFE0C0 with flag 0 and no prior 0x6FCFC0B0 | READ (irrelevant to players) |

## 5. Edge cases

1. **One boss dead (READ, high impact).**
   - Death case 0x102C1C86 leaves the dead clone's pointer in its slot. A dead unit runs no AI, so nothing overwrites the pointer.
   - Both slots stay non-null, so the survivor keeps taking **50 % of every hit for the rest of the fight**.
   - The other 50 % goes to `SetStat(corpse, 6, max(0 − v, 0))`, which does nothing.
   - Open wounds also keeps putting a state on the corpse.
   - The only escapes are state 8 on the corpse (not normally possible) or the pointer changing.
2. **Freed corpse → dangling pointer (INFERRED).** If the corpse unit is freed while the pointer is still in `game+0x2600/0x2604`, 0x1026F63E, 0x1026F64C and 0x1026F66E read and write freed memory. The unit pool can reuse that memory for another unit, which would then receive the shared damage, or the game could crash. No code that clears the slot on unit free was found.
3. **Partner out of range, other room or other level (READ).** There is no distance check.
   - 0x3A5 and 0x3A6 also write `game+0x2600` every AI tick (0x102B2571).
   - If the level-161 Mendeln (0x3A6) is still alive and its room is active (for example, another player is there), `game+0x2600` alternates between it and the BoneClone.
   - Damage to the PoisonClone (partner = `+0x2600`) can then go to the original Mendeln in another level.
   - Before the BoneClone's first AI tick, `+0x2600` still holds the dead level-162 Rathma, because death case 0x102C1C3F does not clear it.
4. **Shared damage larger than the partner's remaining life (VERIFIED clamp, READ consequence).**
   - The partner is set to 0 HP through SetStat, with no death processing: no mode change and no kill events.
   - It stays alive at 0 HP until its own damage path runs. Any later direct hit on it kills it: execute's `life ≤ 0` check at 0x6FCFE2CC sets kill flag 2 even for 0 damage.
   - It can also climb back from 0 through heals (item 5).
   - Its poison or OW tick also kills it (0x6FC967EC).
5. **Healing and regen (READ).**
   - The blood golem (AI PD 0x102B1BA0, class 0x3AA) heals only the Mendeln slot (0x3A6/0x3A8) with `hp = min(max, hp + max·k/100)`. This is not mirrored, so the two life totals drift apart.
     - It has no alive check, so it can heal a dead Mendeln clone's HP stat.
   - When a player moves from levels 161–163 to level 109 (Harrogath), PD 0x102F4C98 heals both slots by 20 % of max. This is mirrored, but also has no alive check.
   - Regen and poison ticks never pass through the share.
6. **Melee timing and phantom damage (READ).**
   - The partner loses its half when the swing *starts* (store 0x6FCFDDE0).
   - The defender takes its half at the damage event (0x6FCFED30), and only if the record survives.
   - The record is lost if the attacker restarts or changes mode (0x6FCFA750, see bug_dragon_tail.md W1), or if the target is out of range at the event (0x6FCFEDAF → delete).
   - In those cases the partner has lost 50 % and the defender 0 %.
   - **Misses and blocks are not shared (READ, checked 2026-09-25).** The swing-start path (0x6FC64BE5) calls the hit/block resolver first (stock 0x6FCFE5A0, redirected by PD2 to 0x10270EB0: range, hit roll, then block/dodge/avoid/evade 0x1026FB60) and stores its flags in the damage record (+4). PD2's resolver clears the hit bit (`and ebx,0xFFFE` at 0x10270FB9) whenever block, weapon block, dodge, avoid or evade succeeds, and returns 0 on a miss or out of range. The store 0x6FCFDDE0 runs the resist loop (where the share lives) only when the hit bit is set and none of 0x8380 are (`test eax,0x8380` / `test al,1` at 0x6FCFDE04–0x6FCFDE0D). So only swings that rolled a clean hit charge the partner.
   - Also, the defender's physical damage is clamped to its current life at 0x6FCFE1FB, *after* the share. Overkill on the defender still fully charges the partner.
7. **Cold/freeze duration (VERIFIED halving).** Chill and freeze lengths on either clone are halved by the same block. This is a side effect.
8. **AoE hitting both.** Each boss takes v_self/2 + v_other/2, which is neutral when the two hits are equal. Nothing is lost (READ).
9. **Very large values (VERIFIED).** No overflow; see §2.

## 6. Suggested fixes (PD code)

1. **Gate the share on instant damage types.**
   - In 0x1026F410, run the block only for entries whose `+0x28` shift field is 8 (entries 0–4, and 9–11 for mercs), or explicitly skip entries 5–8.
   - This stops the halving of poison length and poison rate and of cold/freeze lengths.
2. **Share poison like open wounds.**
   - In PD 0x10271A70, when the defender is 0x3A7/0x3A8 and both slots are valid, halve the rate and apply the same poison (full length) to the partner as well, using the same per-attacker state logic.
   - The code at 0x102AF355–0x102AF5D4 is the template.
   - An alternative is to intercept the regen tick (0x6FC96740 path, state 2) and mirror the life loss, but that would also mirror the partner's own regen.
3. **Clear the slots on death.**
   - In death case 0x102C1C86, add `if (unit == game+0x2600) game+0x2600 = 0; if (unit == game+0x2604) game+0x2604 = 0;`.
   - Also add an alive test (mode ≠ 0/12) for the partner in 0x1026F635 and 0x102AF389.
   - This fixes the survivor's permanent 50 % reduction and the dangling pointer.
4. **Validate the partner.** Accept the partner only if its class is in {0x3A7, 0x3A8}, differs from the defender's class, and both are in the same level. Alternatively, register slots only for 0x3A7 and 0x3A8 in 0x102B2560.
5. **Kill properly or keep 1 HP.** When `life − v < 256`, either run the normal monster kill path or clamp to 256 so a 0-HP zombie cannot exist.
6. **Share at impact, not at store.** Move the partner subtraction from 0x1026F410 to the execute path, just before or after the life SetStat at 0x6FCFE264. Use the final `dmg+0x4C` (and the CB delta). This fixes phantom melee sharing, the overkill transfer, and the unshared crushing blow in one place.
7. Optional design points:
   - apply the partner's own resistances instead of the defender's;
   - credit the damage meter with the halved value;
   - mirror or deliberately exempt golem healing.

## 7. Reproduction files

- `/tmp/claude-0/rshare/share.c`: native harness for 0x1026F410. Build with `gcc -m32 -nostdlib -static -O1 -fno-pie -no-pie -ffreestanding -fno-stack-protector`. Run from `harness/`.
- `/tmp/claude-0/rshare/drv.py`: case generator and model. Note that the harness's fake monster caps phys resist at 50 and elemental at 75 in the getter, so the generator keeps resist inside those caps. The share block itself does not depend on resist.
- `/tmp/claude-0/ai.asm` (AI 0x102B2010) and `/tmp/claude-0/death.asm` (death dispatcher 0x102C11F0): extracted listings.
