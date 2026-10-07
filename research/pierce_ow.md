# Projectile pierce and open-wounds stacking (PD2 = 1.13c + ProjectDiablo.dll)

Native checks: `harness/pierce_ow.c` + `harness/pierce_ow.py` (run `python3 pierce_ow.py`; `python3 pierce_ow.py mutate` must fail every group).
JS: `damage.js` gains `pierceCount`, `stockPierceCount`, `pdSeededRand`, `multiShotPiercePct`, `openWoundsTarget`. Every existing export is kept.
Table of pierce counts by chance: `adv/re/pierce_table.json`, written by the driver from the native runs.

## Summary

- **Pierce is not rolled when the missile hits.** It is rolled once, when the missile is created. The roll counts how many consecutive draws land under the chance, up to 10 (6 for Lightning Fury). Each later hit spends one of those pierces.
- **The random seed is the owner's base stat 328 `pierce_idx`.** No code ever sets that stat on a player, so for players it is always 0. A player's missiles therefore get a **fixed** number of pierces for a given chance:

  | total pierce chance | pierces per missile (PD2) | Lightning Fury | stock 1.13c |
  |---|---|---|---|
  | 0 – 66 | **0** | 0 | 0 |
  | 67 – 86 | **1** | 1 | 4 |
  | 87 – 94 | **5** | 5 | 4 |
  | 95+ | **10** | 6 | 4 |

  Monsters, mercenaries and summons have a seed taken from their position, so for them each missile's count follows the intended chance^k distribution.
- **Open wounds:** each target carries **one** open-wounds state, owned by whoever created it.
  - The owner's next two procs add their damage (3 stacks).
  - Later procs from the owner only reset the 5-second duration.

## 1. Projectile pierce

### 1.1 Where it happens

| step | PD2 | stock D2Game | status |
|---|---|---|---|
| roll at missile creation | PD `0x102CB8D0` (called from D2Game `0x6FC8FE59` → PD `0x102F0D70`) | `0x6FC8F660` | VERIFIED (both, native) |
| client-side prediction of the roll | PD `0x102CBA10` (D2Client `0x6FB6363D` → PD `0x102F0D80`) | – | READ |
| collision gate | PD `0x102723E0` at `0x102724CB–0x10272526` (all 10 D2Game call sites of `0x6FC5EDE0` → PD `0x102F1010`) | `0x6FC5EEF1–0x6FC5EF33` | READ |
| per-hit step | PD `0x1026EFE0` (client: D2Client `0x6FB61529` → PD `0x102ED3B0`) | `0x6FC58F40` | VERIFIED (PD, native) |
| PD's seeded rand | PD `0x102C5D70` | inline 64-bit LCG | VERIFIED (native) |
| seed init | D2Common #10614 `0x6FD86740`: `{lo = value, hi = 666}` | same | VERIFIED (run as real code) |

### 1.2 The rule

```c
// At missile creation (PD 0x102CB8D0). owner = the unit that fires (player, merc, summon, monster).
if (owner is not a player/monster || missile has no MissileData)            return;
if (!(Missiles.txt flags & 4))                                               return;   // "Pierce" column
chance = T(owner,156 item_pierce, layer 0)                                   // #10910
       + T(owner,166 skill_pierce, layer 0)                                  // #10973
       + layered(owner,166)                                                  // 0x102D30B0: weapon-matched layers,
                                                                             //   e.g. Throwing Mastery on 'thro'
if (chance == 0)                                                             return;   // stat 328 stays 0
seed = { lo = BASE stat 328 of the owner (#10587, base list only), hi = 666 };
cap  = (missile skill == 35 Lightning Fury) ? 6 : 10;                        // MissileData+0xA
n = 0;
do { r = pdSeededRand(&seed); } while ((unsigned)(r % 100) < (unsigned)chance && ++n < cap);
missile.stat[328 pierce_idx] = n;
// The seed lives on the stack. The owner's stat 328 is never written back, so the next missile starts over.
```

```c
// When a missile touches a unit (PD 0x102723E0 → 0x1026EFE0). Order within one hit:
//   last-hit check (#10227) → record hit (#10024) → target/owner checks (0x102F0FF0, 0x102ECE30) → PIERCE STEP
//   → damage 0x1026F070 (incl. Multiple Shot dmg func 17) → hit func
owner = direct owner of the missile (0x102CB180(missile,0)), only if the missile has unit flag 0x400
if (owner && (Missiles.txt flags & 4) && (T(owner,156,0) != 0 || T(owner,166,0) + layered(owner,166) != 0)) {
    if (missile.stat[328] > 0) { missile.stat[328]--; missile.stat[469 pierce_count]++; result = 2; }  // keeps flying
    else                         result = 3;                                                            // stops here
} else result = 3;
```

- **Whose stats are read** (READ, VERIFIED for the roll): the **owner's**, not the missile's. The count is fixed at creation from the owner's stats at that moment.
  - At each hit, the owner's *current* 156/166 must still be nonzero, or the missile stops.
  - If the owner can't be found (it died or left), the missile also stops.
- **How 156 and 166 combine** (VERIFIED): they are summed before the roll. There is no separate roll per stat and no cap on the sum. A sum of 100 or more passes every draw.
  - The compare is **unsigned**. A negative total (for example, a negative `item_pierce` somewhere) passes every draw and gives the full 10. Stock uses a signed compare, which never passes.
- **Which missiles** (DATA): Missiles.txt `Pierce`=1 is flag bit 2 (0x4). The bit order was checked against the compiled missiles table: all 1057 rows and all 16 flag columns match.
  - 78 missiles have it. These are all arrows and bolts, the javelins (including Lightning Fury, Plague/Poison Javelin, pilum), throwing axes, knives and glaives, Split Throw, Strafe, Blessed Hammer, Death Sentry's bolt, and a few monster missiles.
  - Lightning Fury's own `lightningfury` missile is capped at 6 through the skill-35 test.
- **Pierce sources** (DATA): `item_pierce` 156 comes from items. Examples: Kuko Shakaku 50, Warshrike 50, Ichorsting 50, Gutsiphon 45, Razortail 33, Buriza 20, Pullspite 25–50.
  - `skill_pierce` 166 comes from Pierce (skill 33, `edmn`: 20 at level 1, +2 per level to 62 at 22, then +1 per level; 67 at level 27, 87 at 47, 95 at 55), from Throwing Mastery (skill 135, `edln` on layer `thro`: 15 at level 1, 41 at level 20), from RoguePierce (merc, 66) and from Skeleton Archer Bow (100).

### 1.3 Why a player's count is fixed (VERIFIED for the roll; READ for "stat 328 is 0 on players")

- **Where stat 328 is written:** every write of stat 328 (`0x148`) in D2Game, D2Common and PD was checked.
  - The missile itself: roll and step.
  - Monsters: D2Game `0x6FC9B263` (the monster case of the unit-send function `0x6FC9B0A0`) and PD `0x102E6B56` both write `(x + y) & 0xFFFF` of the unit's position.
  - Nothing writes it for players, so a player's base stat 328 is 0. The seed is `{0, 666}`.
- **Draw sequence:** from `{0, 666}`, PD's rand gives `r % 100` = **66, 86, 72, 39, 45, 94, 66, 56, 33, 83, …**. So:
  - chance ≤ 66 fails the first draw: 0 pierces.
  - 67–86 fails the second draw: 1.
  - 87–94 passes draws 2–5 and fails the sixth (94): 5.
  - 95+ passes everything up to the cap: 10.
- **Stock:** 1.13c uses the plain LCG, whose sequence is 66, 18, 35, 30, 81, … with a cap of 4. So stock players got 0 below 67% and 4 from 67% up.
- **Why it's done this way:** the client predicts the roll with the same seed (PD `0x102CBA10`), and the seed has to be something both sides know.
- **Monsters, mercs and summons:** each unit's own seed gives it a fixed count per chance, but across many units the counts follow the intended geometric distribution. Measured over all 65,536 seeds: chance 66 gives a mean of 1.91 pierces (the ideal is 1.91); chance 50 gives 1.00.

**Worked examples (players, VERIFIED counts):**

| build | chance | pierces |
|---|---|---|
| Pierce level 20 (58) | 58 | 0 |
| Pierce level 20 + Razortail (33) | 91 | 5 |
| Pierce level 20 + Kuko Shakaku (50) | 108 | 10 |
| Warshrike (50) alone | 50 | 0 |
| Warshrike (50) + Throwing Mastery level 20 (41, holding a throwing weapon) | 91 | 5 on the server; the client predicts 0 |
| Lightning Fury with 95+ | 95+ | 6 |
| Skeleton Archer (merc or summon, skill_pierce 100) | 100 | 10, whatever the seed |

### 1.4 After a pierce

- **pierce_count 469** (VERIFIED): +1 on every pierce, on every piercing missile. Only Multiple Shot's damage function 17 (PD `0x102F2030`) reads it (VERIFIED earlier, `dmg_C_spells.md`):
  ```c
  k = stat469 - 1;  if (k > 0) every damage slot ×= max(20, 100 - 20k) %
  ```
  - **Ordering** (READ): the pierce step runs **before** the hit's damage. The i-th unit hit by one arrow therefore sees `469 = min(i, count)`.
  - With count ≥ 5, the damage on the 1st, 2nd, 3rd, … unit is **100, 80, 60, 40, 20, 20, …%**.
  - The unit where the arrow stops takes the same % as the one before it. With count 1, the arrow deals 100% then 100%.
  - `damage.js multiShotPiercePct(i, count)` gives these values.
  - Only `multipleshotarrow` and `multipleshotbolt` use damage function 17 (DATA).
- **Same target twice** (READ): D2Common #10227 `0x6FDBA130` skips a unit if Missiles.txt `LastCollide` (bit 0) is set and the unit is the missile's *last* hit unit (MissileData+0x18/+0x1C). #10024 records each hit.
  - 76 of the 78 piercing missiles have LastCollide, so they never hit the same unit twice in a row.
  - `guidedarrow` and `bladefragment1` don't have it, so they can re-hit a unit they stay inside of. Each re-hit uses up one pierce.
- **Limit:** the number of pierces is the rolled count (≤ 10, or 6 for Lightning Fury). There is no separate limit on range or targets.

### 1.5 PD2 versus stock

| | stock 1.13c | PD2 |
|---|---|---|
| cap | 4 | 10 (Lightning Fury 6) |
| rand | 64-bit LCG | PD `0x102C5D70` (different high word) |
| compare | signed | unsigned |
| weapon-matched layers of 166 | no | yes, on the server (Throwing Mastery). The client prediction `0x102CBA10` still omits them. |
| pierce_count 469 / Multiple Shot falloff | – | yes |
| collision gate | 166 total or 156 | 156 or 166 + layers |

## 2. Open wounds stacking (PD event 15 = `0x102AF060`)

### 2.1 Per-proc value (VERIFIED earlier, damage.md §5; re-run here)

```c
dmg = (levelScale(clvl) + 25 + 5*deep_wounds) * (100 - r_eff)/100   // r_eff: phys res − 425, negatives halved
// monster owned by a player: dmg /= 4 (0x102AF30F)
// PvP target: (lvl+25)·{8,6,4}%·… per difficulty, deep_wounds from the 'deepwounds' state (225) counted at 1/3, and phys res capped at 50 (READ)
```

### 2.2 How procs combine (VERIFIED, 11,062 random procs across 1,500 sequences with 1–3 attackers, 0 mismatches)

> **Correction (audit):** the 11,062 cases are proc **attempts**; 5,517 of them roll no proc. The actual procs are 5,545 (3,745 creations, 1,152 own stacking, 105 at 3 stacks, 543 other-attacker refreshes). 0 mismatches re-confirmed.


```c
// on a successful proc (rand(100) < item_openwounds) by attacker A with value dmg, at frame f:
mine = PD 0x10269230(target, state 62, A)      // a state-62 list whose owner type+GUID is A
if (mine) {
    s = mine.stat[189];
    if (s < 3) { mine.stat[74] -= dmg; stacks = s + 1; }        // 0x102AF400-0x102AF421
    else        { stacks = s; }                                 // 0x102AF428: drain NOT changed
} else stacks = 1;
L = PD 0x102BFE20({owner A, target, skill 0, level 1, 125 frames, stat 74 = -dmg, state 62});
//   0x102BFE20, non-curse path 0x102C02FF:
//     existing = FIRST state-62 list on the target, whatever its owner   (#10871)
//     same skill (0) and level (1) → refresh only: expire = f + 125 (#10161), new timer; value NOT written
//     none → create a list owned by A with stat 74 = -dmg, expire f + 125
L.stat[189 openwounds_stack] = stacks;
```

What this means (every point VERIFIED natively):

1. **At 3 stacks:** a new proc from the owner **does not change the drain**. It only resets the duration to 125 frames (5 s), and the stack count stays at 3.
   - The drain stays at d₁ + d₂ + d₃: the values of the first three procs, each computed at its own proc time.
   - The old damage.md note ("the new state carries just the new dmg") was wrong. The `-dmg` passed to `0x102BFE20` is used only when the state is *created*.
2. **Per attacker or shared:** each target has only one OW state.
   - `0x102BFE20` finds any existing state 62 regardless of owner and refreshes it. It never creates a second one.
   - The owner-matched search `0x10269230` only decides whether the proc counts as "mine".
3. **Expiry:** 125 frames after the last proc. The drain then drops to 0, and the next proc starts over at 1 stack. (The timer model is READ: removal happens when the last scheduled expiry frame is reached.)
4. **Damage meter** (VERIFIED value; its meaning is unclear): if the attacker is a player, PD adds `min(credit, target life)` to `pPlayerData+0x1A8` and stores the frame at `+0x265`.
   - The credit is `trunc(dmg·5/256)` per proc. At 3 stacks it is `trunc(dmg·(125 − remaining)/125/100)`.
   - Either way, it is about 1/25 of what the proc actually drains.

**Worked example (native run):** level 90, 200 deep wounds, 0 resist, so each proc is 3716 (14.5 life/frame, 363 life/s).

| frame | proc by | drain (per frame ×256) | stacks | life/s |
|---|---|---|---|---|
| 0 | A | 3716 | 1 | 363 |
| 10 | A | 7432 | 2 | 726 |
| 20 | A | 11148 | 3 | 1089 |
| 30 | A | 11148 | 3 | 1089 (refresh only) |

`damage.js openWoundsTarget()` reproduces this table.

### 2.3 Regeneration and prevent-heal (READ)

- **Monsters, mercs and summons** are handled by the monster-mode life tick D2Game `0x6FC96740` (MonMode table `0x6FD1BC20`). PD does not patch it.
  ```c
  r = T(unit, 74 hpregen)                          // all lists: base regen + OW state + others
  if (#10372(unit)) r -= base stat 74              // unit has a state from the 'life' group:
                                                   //   States.txt life=1 → poison (2), preventheal (52), openwounds (62)
  if (unit has state 52 preventheal && r >= 0) stop (no tick, not re-armed)
  life = min(life + r, maxlife);  if (life < 1) life = 0  → dies; the kill goes to the owner of the
                                                   //   poison (state 2) list, else of the open-wounds (62) list
  ```
  - The monster's own regen is base stat 74 = `(maxlife<<8)·MonStats.DamageRegen >> 12` per frame (`0x6FCD00B5–0x6FCD0117`; most PD2 monsters have DamageRegen 2, which is about 1.2% of max life per second).
  - **While open wounds is on a monster, that regen is switched off completely**, and the full OW drain applies. Prevent-heal also switches it off (it is a 'life' state too). Adding prevent-heal on top of OW changes nothing.
  - OW can kill a monster. The kill is credited to the OW state's owner, which is always the first attacker.
  - The mask behind #10372 (D2Common `0x6FD83D80`, table +0x14C) was not decoded. Treating it as the 'life' column is an inference: those three states are the only ones with life=1.
- **Players** use `0x6FC97CB0` (block_regen.md §3). It sums all of stat 74 with no state tests, so replenish life directly offsets OW, and life is clamped at 1 point (OW cannot kill a player).

### 2.4 PD2 versus stock (stock func 15 = D2Game `0x6FCCD410`, READ)

| | stock | PD2 |
|---|---|---|
| value | levelScale + 40 | levelScale + 25 + 5·deep_wounds, phys res and 425 pierce, halving |
| vs players / others | player target ÷4, and ÷2 more from a missile hit (event 6); monster target ÷2 when D2Game `0x6FC42A80(target, 12)` | ÷4 for a player-owned monster; separate PvP formula |
| duration | 200 frames | 125 frames |
| stacking | none (a re-proc refreshes; value kept) | 3 stacks per owner as above |
| Rathma/clone share | – | halved drain also applied to the partner |

## 3. Verification

| check | real code run | cases | mismatches |
|---|---|---|---|
| PD pierce roll | `0x102CB8D0` + `0x102CE600` + `0x102C5D70` + #10614 | 9,899 (random chance −20…250, seeds, skills, rows) | 0 |
| stock pierce roll | `0x6FC8F660` + `0x6FC21250` + #10614 | 10,101 | 0 |
| PD per-hit step | `0x1026EFE0` | 5,000 | 0 |
| PD seeded rand | `0x102C5D70` | 3,000 | 0 |
| open-wounds proc sequences | `0x102AF060` + `0x10269230` + `0x102BFE20` + real D2Common list accessors | 11,062 procs / 1,500 sequences (1,152 own stacking, 105 at 3 stacks, 543 other-attacker refreshes, 3,745 creations) | 0 |
| mutation run | cap, rand, stock cap, 469, the 3-stack rule each broken | – | every group fails |

- **Stubbed in the harness:**
  - Stat getters.
  - Statlist alloc/free/merge and per-list stat get/set.
  - The timer queue: a list is removed at its last scheduled expiry.
  - The unit rand.
  - `0x102D30B0`.
  - The owner lookup `0x102CB180`.
  - PD's `unordered_set` lookup at `0x102BFEFE`, which only matters for state 0x3A.
- **Not run:**
  - The collision function `0x102723E0` itself.
  - The monster regen tick `0x6FC96740`.
  - The PvP OW branch.
  - The client-side copies.
