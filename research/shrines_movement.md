# Shrines, wells, stamina and movement speed (PD2 = 1.13c + ProjectDiablo.dll)

Status labels:
- **VERIFIED** = the real code was run natively in the harness and matched a model.
- **READ** = from the disassembly only.
- **DATA** = from the PD2 tables (`bin/pd2data.mpq`, which matches `data.zip`).

Units and conventions:
- The server runs 25 frames per second.
- Life, mana and stamina (stats 6–11) are stored in 8.8 fixed point (×256).
- `tdiv` is C integer division, truncating toward zero.
- Existing notes that this file extends rather than repeats:
  - `block_regen.md` §5 covers the stamina regen and drain formulas.
  - `mf_misc.md` §4 covers the walk/run velocity formula.
  - `dmg_B_pipeline.md` covers the length prestep, where the shrine states 131 and 133 zero the burn and poison lengths.

---

## 1. Shrines

### 1.1 The data (DATA)

- PD2 ships `Shrines.txt` and `shrines.bin` in `pd2data.mpq`.
  - The `.bin` file (23 records of 0xB8 bytes) matches the `.txt` field for field.
  - Both are identical to vanilla LoD.
- Record layout, as read by the code:

  | Offset | Field |
  |---|---|
  | +0 | code |
  | +4 | arg0 |
  | +8 | arg1 |
  | +0xC | duration in frames |
  | +0x10 | reset (byte) |
  | +0x11 | rarity |
  | +0xB2 | effectclass |
  | +0xB4 | LevelMin |

- The record getter is D2Common `0x6FD8E910`. It reads the table pointer at `0x6FDF0B8C` and the count at `0x6FDF0B90`.

| id | name | arg0 | arg1 | duration | reset | effectclass | LevelMin |
|---|---|---|---|---|---|---|---|
| 1 | Refill | 100 | 100 | – | 2 | 4 | 1 |
| 2 | Health | 200 | 400 | – | 5 | 2 | 1 |
| 3 | Mana | 200 | 400 | – | 5 | 3 | 1 |
| 4 | Health Exchange | 50 | 500 | – | 0 | 2 | 2 |
| 5 | Mana Exchange | 50 | 500 | – | 0 | 3 | 2 |
| 6 | Armor | 100 | 1 | 2400 | 5 | 4 | 5 |
| 7 | Combat | 200 | 200 | 2400 | 5 | 4 | 8 |
| 8 | Resist Fire | 75 | 0 | 3600 | 5 | 4 | 5 |
| 9 | Resist Cold | 75 | 0 | 3600 | 5 | 4 | 26 |
| 10 | Resist Lightning | 75 | 0 | 3600 | 5 | 4 | 32 |
| 11 | Resist Poison | 75 | 0 | 3600 | 5 | 4 | 21 |
| 12 | Skill | 2 | 0 | 2400 | 5 | 4 | 1 |
| 13 | Mana Recharge | 400 | 0 | 2400 | 5 | 4 | 1 |
| 14 | Stamina | 200 | 0 | 2400 | 5 | 4 | 1 |
| 15 | Experience | 50 | 50 | 3600 | 0 | 4 | 1 |
| 16 | Enirhs | 0 | 0 | – | 0 | 1 | 1 |
| 17 | Portal | 0 | 0 | – | 0 | 1 | 3 |
| 18 | Gem | 0 | 0 | – | 0 | 1 | 4 |
| 19 | Storm | 50 | 2000 | – | 0 | 1 | 1 |
| 20 | Warping | 0 | 0 | – | 0 | 1 | 3 |
| 21 | Exploding | 5 | 10 | – | 0 | 1 | 1 |
| 22 | Poison | 5 | 10 | – | 0 | 1 | 1 |

Duration conversions: 2400 frames = 96 s, and 3600 frames = 144 s.

### 1.2 Choosing the shrine type (VERIFIED)

At game start, D2Game `0x6FC2BAE4..0x6FC2BBC9` builds one list per effectclass, in id order:
- class 1: 16–22
- class 2: 2, 4
- class 3: 3, 5
- class 4: 1, 6–15

Every shrine object runs InitFn 1, D2Game `0x6FC4EF90`. It branches on `Objects.txt Parm0` (+0x178):

```
Parm0 = 1 (health shrine objects) -> class 2
Parm0 = 2 (mana shrine objects)   -> class 3
Parm0 = 3 (magic shrine objects)  -> class 1 if rand(A) % 10 == 0 else class 4   (10% / 90%)
Parm0 = 0 (unused by PD2 objects) -> id = rand(A) % 22 + 1, same LevelMin retry as below

pick(class, levelId)  [0x6FC4ED70, game shrine seed G]:
    repeat up to 8 times:
        id = list[class][rand(G) % n]   (0 -> 1)
        if levelId >= LevelMin[id]: return id
    return the 8th id                   (kept even if LevelMin is not met)

final remap: 5 -> 3, 4 -> 2, 16 -> 18   (Mana/Health Exchange and Enirhs never appear)
```

- `A` is the object's own seed (arg +0xC). `G` is the game shrine seed at `[game+0x10F0]`.
- The RNG is `seed = lo·0x6AC690C5 + hi`, and `rand(n)` uses `& (n−1)` when n is a power of 2.
- **`levelId` is the area's Levels.txt Id**, not the area level. It comes from room→room2+0x58→level+0x1D0, via D2Common `0x6FD8C000`.
  - So LevelMin 26/32 only excludes Resist Cold / Resist Lightning in Act 1 areas with id < 26 / < 32.
- The **rarity column is not read** by any of this code.
- Every shrine id in a class is equally likely.

**VERIFIED:** `harness/shrine.c` + `shrine.py`.
- 120,000 cases: Parm0 0–3, area ids 0–201, random seeds on both RNGs.
- It compared the final id, the record set on the object, and both final seeds.
- Result: 0 mismatches. A mutated model gives 5,995 mismatches.
- A 100k-draw Monte Carlo per area matched the closed form below to within ±0.2 points during the original work. **The shipped `shrine.c`/`shrine.py` do not include that run, so treat this as READ + model, not VERIFIED (audit).**

Chance of each type for a **magic shrine object** (Parm0 = 3), from `shrineTypeChances(levelId)` in `mf_misc.js`:

| area id | Refill, Skill, Mana Recharge, Stamina, Experience (each) | Armor | Combat | Res Fire | Res Poison | Res Cold | Res Light | Gem | Portal, Storm, Warping, Exploding, Poison (each) |
|---|---|---|---|---|---|---|---|---|---|
| 1–2 | 17.86% | 0.12% | 0.12% | 0.12% | 0.12% | 0.12% | 0.12% | 2.50% | Storm / Exploding / Poison 2.50%, Portal / Warping ≈0% |
| 5–7 | 12.85% | 12.85% | ≈0 | 12.85% | ≈0 | ≈0 | ≈0 | 2.86% | 1.43% |
| 8–20 | 11.25% | 11.25% | 11.25% | 11.25% | 0 | 0 | 0 | 2.86% | 1.43% |
| 21–25 | 10.00% | 10.00% | 10.00% | 10.00% | 10.00% | 0 | 0 | 2.86% | 1.43% |
| 26–31 | 9.00% | 9.00% | 9.00% | 9.00% | 9.00% | 9.00% | 0 | 2.86% | 1.43% |
| ≥ 32 (Act 1 from the Inner Cloister, all of Acts 2–5, every PD2 map) | 8.18% | 8.18% | 8.18% | 8.18% | 8.18% | 8.18% | 8.18% | 2.86% | 1.43% |

Health shrine objects always give Health (id 2), and mana shrine objects always give Mana (id 3).

### 1.3 Where shrines spawn (READ + DATA)

- **ObjGroup populate** (D2Game `0x6FC4F0D0..0x6FC4F2B9`), for each room:
  - For each of the level's 8 `ObjGrp`/`ObjPrb` slots:
    - Roll `rand(100)` against `ObjPrb`. It passes when the roll ≤ ObjPrb, so the chance is ObjPrb+1 %.
    - Then pick a member by cumulative `PROBn`.
    - Then call the object's PopulateFn with the group's `DENSITYn`.
- **Shrine PopulateFn 2** (`0x6FC527D0`) keeps per-level counters in `game+0x10F0 → +0x48 + level·4`:
  - It stops once 10 shrines are placed, or once `placed > rooms/8`. So the maximum per level is `min(10, rooms/8 + 1)`.
  - It tries 3 placements (0x6FC517F0).
  - It records up to 10 shrine positions and counts Health shrines at +0x10.
  - **Forced health shrine:**
    - Trigger: once more than 75% of the level's rooms are populated (`+4·128/+8 > 96`) and no Health shrine exists yet.
    - The next shrine skips the roll and gets 30 placement tries.
    - It is converted into the object's `Parm1` "health" counterpart (0x6FC4E8B0; default object 84, healing well) and set to shrine 2.
  - The meaning of the +4/+8 fields as "rooms populated / rooms total" is inferred.
- **Outdoor act shrines** come from preset (DS1) maps rather than ObjGroups. They run the same InitFn.
- **PD2 maps have shrines (DATA, Levels.txt):**
  - Mesa, Siege, Palace, Sewers, Arcane, Kurast, River of Blood, Lava, Ice, Desert, Jungle, Fortress, Tomb, Fall of Caldeum, Pandemonium Citadel, Lost Temple, Canyon, Torment, Hole, Library, Westmarch, Crypts, Sanctuary of Sin, Ruined Cistern, Kanemith Outside/Dungeon, Demon Road, Imperial Palace, Halls of Torture, Na-Krul's Abyss and Kyovoshad all have shrine groups (e.g. `inner hell shrines` 60, `travinsal shrine` 25).
  - Rathma, Zhar, Throne, Monastery, Graveyard, Ashen Plains, Black Abyss, Ureh, Nemyr, Djinn, Skovos, Diamond Gate, Fallen Gardens, Hellcaves, Outer Void, Lucion and the PvP maps have none.
  - Imperial Boss Map lists `outer hell shrines` with ObjPrb 0, which the roll above still passes 1% of the time.
  - All map ids are ≥ 138, so every shrine type is possible on maps.

### 1.4 Using a shrine (READ)

OperateFn 2 = D2Game `0x6FC8DA00` (dispatch table `0x6FD27BB8`, OperateFn at Objects +0x1B3).

1. It refuses if the object mode ≠ 0, or if `objdata+0xC ≠ 0`.
   - It stores the user's unit id + 1 there. **Only one player can take it**, until it resets.
2. It sets the object mode to 1. It shows the overhead text (string 0xE63 + id), and clears the text after 300 frames (event 6).
3. It dispatches on the shrine code through the table `0x6FD1A5F8` (fn, stat, state). The unknown code falls back to Refill `0x6FC89C00`.
4. If `reset ≠ 0`, it schedules event 5 at `now + reset·1200 + 1` frames.
   - Event 5 (`0x6FC89D60`) puts the object back to mode 0 and clears `objdata+0xC`. The same shrine type is reusable; it is **not re-rolled**.
   - The "minutes" column is really 1200 frames = **48 s** per unit.
   - Refill: 96 s. Health, Mana and every booster except Experience: 240 s.
   - Experience and all magic shrines: never.
5. The effect is applied **only to the operating player** (`arg+8`). Nothing iterates party members or pets.
   - **Shrines are not shared with the party.**
   - This matches the PD2 code path: none of these functions are patched, except the stamina one below.
6. If an unloaded room is reloaded (level cache `0x6FC40130`), a pending reset is rescheduled from the saved frame.

### 1.5 Effects

**Booster application** (`0x6FC8ABB0` → `0x6FC8A960` → `0x6FCC00B0`, the generic timed state):
- It creates a statlist on the player with the state id, one stat, and an expiry of now + duration.
- Value helper `0x6FC89EE0(stat, arg0)` (**VERIFIED**, 40k cases, 0 mismatches; a mutation gives 2,905):

  ```
  stat 11, 27, 39, 41, 43, 45, 85, 171 -> value = arg0
  stat 19 (tohit)                      -> value = tdiv(AR(current skill) · arg0, 100)     0x6FC89B00, READ
  other stats                          -> value = tdiv(GetUnitStat(player, stat) · arg0, 100)
  ```

| Shrine | What it does | Stats in the state list | State | Duration | Status |
|---|---|---|---|---|---|
| Refill (1) | life = max life, mana = max mana (args ignored) | – | – | – | READ `0x6FC89C00` |
| Health (2) | life = max life | – | – | – | READ `0x6FC898A0` |
| Mana (3) | mana = max mana | – | – | – | READ `0x6FC89860` |
| Armor (6) | +100% defense | 171 `skill_armor_percent` = 100 (additive with Shout and item %ED) | 128 | 96 s | READ |
| Combat (7) | flat AR = 2× your current attack rating, +200% damage | 19 `tohit` = tdiv(AR·200,100); 25 `damagepercent` = 200 | 129 | 96 s | READ `0x6FC8ACE0` |
| Resist Fire (8) | +75 fire resist; **burning length → 0** | 39 = 75 | 131 | 144 s | READ; length rule VERIFIED in dmg_B |
| Resist Cold (9) | +75 cold resist (no effect on chill or freeze) | 43 = 75 | 132 | 144 s | READ |
| Resist Lightning (10) | +75 lightning resist | 41 = 75 | 130 | 144 s | READ |
| Resist Poison (11) | +75 poison resist; **poison length → 0** | 45 = 75 | 133 | 144 s | READ; length rule VERIFIED in dmg_B |
| Skill (12) | **+2 to all skills, hard-coded** (arg0 ignored) | stat 11 = 0 (placeholder) | 134 | 96 s | READ D2Common `0x6FD9EBA0` |
| Mana Recharge (13) | +400% mana regeneration (×5) | 27 `manarecoverybonus` = 400 | 135 | 96 s | READ |
| Stamina (14) | stamina refilled; effectively unlimited stamina; **PD2: +35% movement speed** | 28 = 1000, 67 = 35 (PD2), 162 = 2× own stat 162, 10 = 2× that | 136 | 96 s | READ `0x6FC8AC30`, PD `0x102C7B60` |
| Experience (15) | +50% experience | 85 `item_addexperience` = 50 | 137 | 144 s | READ |

Details:
- **Skill shrine.** D2Common `0x6FD9EBA0` returns 2 when a player, monster or missile unit has state 134. The skill-level bonus function `0x6FD9FCB0` adds this to every skill level, together with stat 127 (auras.md). The Shrines.txt arg0 of 2 is never read. Remove callback `0x6FC8A930` refreshes skills.
- **Resist Fire / Poison.** The length prestep (stock `0x6FCFA7B0` at `0x6FCFA81A`/`0x6FCFA83A`, PD `0x102713C0`) zeroes these lengths:
  - The burn length when state 131 is set. Burning is the fire-over-time that ignores fire resistance.
  - The poison length when state 133 is set. Only the single frame of poison that is part of the hit total gets through, after +75 resist.
  - The cold shrine has no equivalent.
- **Stamina shrine:**
  - It first sets stamina = max stamina.
  - Its list has stat 28 `staminarecoverybonus` = 1000. From `block_regen.md`, stat 28 ≥ 1000 turns regeneration on in *every* mode, including running, at `(max>>8)·11` per frame. That is 11/256 of the bar per frame, far above the running drain of 40/256 of a point.
  - **PD2** replaces the call that writes stat 28 (`0x6FC8ACD4`, patch rel32 at `0x6FC8ACD5`) with `0x102C7B60`, which does:

    ```
    SetStat(list, 67 velocitypercent, 35, 0)   // D2Common #10261 via lazy thunk 0x10273B70
    SetStat(list, 28, 1000, 0)
    ```

    So a PD2 stamina shrine also gives **+35 velocitypercent** (not FRW, so there are no diminishing returns) for 96 s.
  - Stat 162 is `skill_staminapercent` (op 1 on maxstamina). Its value is 2× whatever the player already has, so 0 unless a Vigor or Battle Orders list is active.
- **Combat shrine AR.** `0x6FC89B00` computes the full attack rating for the player's current skill (base tohit + stat 119 % + skill ToHit + CharStats ToHitFactor). The shrine adds 2× that as flat tohit, so the displayed AR roughly triples. Re-taking the shrine re-writes the value (see 1.6).
- **Experience shrine.** Stat 85 is added to the item stat, so it is additive with gear %exp. It is applied after the level penalty and ExpRatio (`mf_misc.md` §3). Example: 1000 base exp for a level 80 player on a level 85 monster gives 941 without the shrine, and `expGain({…, bonus: 50})` = 941 + 470 = 1411.

> **Correction (audit):** the example leaves out ExpRatio (it calls `mf_misc.js expGain` with its default `expRatio = 1024`). A level-80 player has ExpRatio 496, so the real gain is **455** without the shrine and **682** with it (`experience.js gainXp(1000, 80, 85, 0 / 50)`). The +50 % relation is unchanged.


### 1.6 Stacking and replacement (READ)

- **Different boosters stack.**
  - `0x6FCC00B0` looks up only the list for *its own* state (#10871-style lookup `0x6FC2A5D4(player, state)`).
  - No code in D2Game, D2Common or PD removes the other shrine states 128–137.
    - Searched: every push/compare of 0x80–0x89, the dispatch table users, and the state apply path.
  - States.txt gives the shrine states no `group`.
  - So Armor + Skill + Experience can all be active at once.
- **The same shrine again refreshes rather than stacks.** If the list exists with the same skill 0 / level 0, only the expiry is reset to now + duration (event 0xC rescheduled). The stat value is not re-added.
  - For Combat and Stamina, the caller re-writes its extra stats on the returned list. So Combat's flat AR is re-computed from your current AR, which already includes the old +200%.
- All shrine states have `plrstaydeath` = 0 (DATA), so they are expected to clear on death. The death path was not traced.

### 1.7 Magic shrines (READ)

- **Portal (17)** `0x6FC8B9B0`: opens a town portal at the shrine.
- **Gem (18)** `0x6FC8B430`: upgrades a random gem in the inventory (Gems.txt lookup) or creates one. Not traced in detail.
- **Storm (19)** `0x6FC8C900`:
  - Every living player and hostile monster in the area query around the shrine loses `tdiv(life_points · 50, 100)` points of life (half their *current* life, so it cannot kill).
  - The filter `0x6FC89C60` skips dead units, and monsters flagged by `0x6FC2A688`. arg1 (2000) is not read.
  - Then it fires 16 Fireballs (missile 62) in a 4×4 pattern. Skill level = `clamp(clvl/5, 1, 8)`.
- **Warping (20)** `0x6FC8B8A0`: runs callback `0x6FC8A440` over nearby monsters to upgrade one. Not traced in detail.
- **Exploding (21)** `0x6FC8C580` and **Poison (22)** `0x6FC8C200`:
  - They drop `arg0 + rand(arg1 − arg0)` = **5–9** potions (the table says 5–10). Item level = the player's clvl, minimum 1.
  - They then throw 6 potion missiles around the shrine. Skill level = `clamp(clvl/5, 1, 8)`.
    - Exploding throws missile 45 `explosivepotion` (8–12 plus fire 8).
    - Poison throws missile 48 `chokinggaspoition` (poison 144 over 50 frames).
  - **PD2 patches the dropped item codes:**
    - Exploding: `opm ` → `tpfs` (TPot Fire Small, missile 912), patch at `0x6FC8C601`.
    - Poison: `gpm ` → `tpgs` (TPot Gas Small, missile 921), patch at `0x6FC8C281`.
  - The thrown missiles are unchanged.
- Enirhs (16), Health Exchange (4) and Mana Exchange (5) have functions (`0x6FC89770`, `0x6FC89800`, `0x6FC897A0`) but can never be rolled (1.2).

### 1.8 Wells (READ + DATA)

Wells are not shrines. They use OperateFn 22 `0x6FC8CEE0` and InitFn 16 `0x6FC4E760`. PD2 Objects.txt (DATA) gives all wells Parm0 750, Parm1 128, Parm2 1, Parm3 3.

- **Charges:** `2·Parm2` = 2. Recharge event 2 (`0x6FC89CA0`) adds 1 charge `Parm0 + 1` = **751 frames (30 s)** after each use, up to 2.
- **Per use** (`0x6FC8A560`), a charge is consumed only if something changed:
  - Life `+= tdiv(Parm1 · maxLife, 256)` (**+50% of max**), if Parm3 bit 2 is set.
  - Mana +50% of max, if Parm3 bit 1 is set.
  - Stamina +50% of max, always.
  - Removes poison (state 2) and freeze (state 1), plus `0x6FCDC920`.
  - The player's pets and mercenary (`0x6FCB6AF0` with `0x6FC89DE0`): **life set to max**, poison and freeze removed.
- Wells, like shrines, act only on the user and the user's own pets.

### 1.9 PD2 changes to shrines (summary)

| Change | Where | Status |
|---|---|---|
| Stamina shrine also gives +35 velocitypercent | `0x6FC8ACD5` → PD `0x102C7B60` | READ |
| Exploding / poison shrine drop `tpfs` / `tpgs` instead of `opm` / `gpm` | byte patches `0x6FC8C601`, `0x6FC8C281` | READ |
| Fire and poison shrine length zeroing kept in PD's length prestep | PD `0x102713C0` | VERIFIED (dmg_B) |
| Shrines.txt / shrines.bin, the selection code, durations, resets | unchanged | DATA / VERIFIED |
| Many PD2 maps carry shrine ObjGroups | Levels.txt | DATA |

---

## 2. Stamina

### 2.1 Stats (DATA ItemStatCost)

| Stat | Name | Effect |
|---|---|---|
| 10 | `stamina` | current (8.8 FP), max = stat 11 |
| 11 | `maxstamina` | max (8.8 FP) |
| 28 | `staminarecoverybonus` | regen % (items "Heal Stamina Plus"; Vigor aura `ln34`) |
| 154 | `item_staminadrainpct` | "% slower stamina drain"; positive values reduce drain |
| 162 / 163 | `skill_staminapercent` / `skill_passive_staminapercent` | op 1 → % of maxstamina (Vigor, BO CTA) |
| 241 / 242 | per-level regen / stamina | op 2 |
| 294 / 295 | by-time regen / stamina | op 6 |
| 67 | `velocitypercent` | movement speed (section 3) |

### 2.2 Drain while running (VERIFIED, extended)

D2Game `0x6FC97BB0`, called by the run-mode handler `0x6FC99B90` only while `mode == 3`:

```
if room is town (#D2Common 0x6FD8C390):   return 1            // no drain in town
d = CharStats.RunDrain * 2                                   // +0x42
if body armor (bodyloc 3):  d *= tdiv(Armor.speed, 10) + 1   // Items +0xD8
d -= tdiv(d * stat154, 100)                                  // 0x6FC97C53: divide by -100 and add
d  = max(d, 1)
stamina -= d                                                 // 1/256 points per frame
if stamina <= 0: stamina = 0; return 0  -> caller sets mode 2 (walk)
```

- **VERIFIED:** `harness/shrine.c` kind 2.
  - 40,000 cases: RunDrain 0–60, stamina 0–200000 FP, stat 154 in −100..100, body armor speed 0–60 or none, town on/off.
  - It compared the return value, the add value and the new stamina. 0 mismatches. A mutation gives 17,055.
- **PD2:** every Armor.txt row has `speed` = 0 (DATA, 205 rows). So **the armor multiplier is always ×1 in PD2**, and heavy armor has no drain or velocity penalty.

### 2.3 Regeneration (VERIFIED in block_regen.md)

```
a = maxstaminaFP >> sh ;  a += tdiv(a * stat28, 100) ;  stamina = min(stamina + a, max)
sh: NU / TN = 8 (full bar 10.24 s), WL / TW = 9 (20.48 s; WL only if stamina >= 1 point),
    any other mode (RN, attacks, casts, GH ...) = no regen unless stat28 >= 1000, then 8
```

### 2.4 At 0 stamina (READ)

- The drain function sets stamina to 0 and returns 0. The run handler then calls the mode setter `0x6FC98610` with mode 2.
- In the mode setter, a request for run (mode 3) is turned into walk (2), or town-walk (6) in town, **when stamina is exactly 0**. Any stamina > 0, even 1/256, allows running.
- **Walking with less than 1 point of stamina regenerates nothing** (WL branch at `0x6FC97A98`). After running dry, you must stand still (or be in town, TW) to recover.

### 2.5 PD2 numbers per class (DATA CharStats; drain VERIFIED)

| Class | starting stamina | RunDrain | drain / s (running) | seconds to empty from start | standing refill | walking refill |
|---|---|---|---|---|---|---|
| Amazon | 184 | 20 | 3.906 | 47.1 | 10.24 s | 20.48 s |
| Sorceress | 149 | 20 | 3.906 | 38.1 | 10.24 s | 20.48 s |
| Necromancer | 154 | 20 | 3.906 | 39.4 | 10.24 s | 20.48 s |
| Paladin | 189 | 20 | 3.906 | 48.4 | 10.24 s | 20.48 s |
| Barbarian | 217 | 20 | 3.906 | 55.6 | 10.24 s | 20.48 s |
| Druid | 159 | 20 | 3.906 | 40.7 | 10.24 s | 20.48 s |
| Assassin | 220 | 15 | 2.930 | 75.1 | 10.24 s | 20.48 s |

- StaminaPerLevel / StaminaPerVitality are 4 / 4 (Assassin 5 / 5). By the usual CharStats quarter-point convention that is 1 per level and 1 per vitality (1.25 for the Assassin). The level-up code was not traced.
- Drain with "slower stamina drain" (RunDrain 20 → d = 40):

| stat 154 | 0 | 15 (Eld in helm) | 50 | 80 | 90 (of Traveling max) | 100 |
|---|---|---|---|---|---|---|
| d per frame | 40 | 34 | 20 | 8 | 4 | 1 (floor) |
| stamina / s | 3.91 | 3.32 | 1.95 | 0.78 | 0.39 | 0.10 |

### 2.6 Is running free anywhere in PD2? (READ)

- **Town:** yes, as in vanilla. The drain returns early for town rooms. No PD2 patch touches `0x6FC97A50`, `0x6FC97BB0` or `0x6FC99B90`.
- **Maps:** no. Map rooms are not town, so the normal drain applies.
- **Effectively free anywhere:** the stamina shrine for 96 s (stat 28 = 1000).
  - Vigor does **not** reach 1000: its `ln34` is 50 + 25·(lvl−1), which is 525% at level 20 and needs level 39+ to hit 1000.
  - Otherwise, anything that pushes stat 28 to ≥ 1000 turns on regen while running.

---

## 3. Movement speed

### 3.1 Formula (VERIFIED in mf_misc.md §4)

```
EFRW = tdiv(150 * FRW, 150 + FRW)                      // item_fastermovevelocity (stat 96) only
s    = max(25, velocitypercent + EFRW)                 // stat 67 total, floor 25 %
vel  = tdiv((WalkVelocity << 8) * s, 100)              // WalkVelocity 6 for every PD2 class
speed = vel * 25 / 4096 subtiles per second             // 0.09375 * s
velocitypercent = 100 (base) + 50 (running: 100*9/6 - 100) + skills/auras/curses/chill + shrine
```

- Only FRW has diminishing returns. `velocitypercent` from skills, shrines and slows is added **raw**.
- There is no upper cap. The only floor is **25%**.
- **Shapeshifting** does not change the base: D2Common `0x6FD80D50` always reads CharStats WalkVelocity for players. PD2 Werewolf/Werebear carry no velocity stats (Werewolf gets `attackrate` `dm34` 10→80 instead). READ + DATA.

### 3.2 Sources (DATA PD2 Skills.txt; slow floors READ)

Calc codes:
- `ln` = a + (lvl−1)·b.
- `dm` = lo + tdiv((hi−lo)·tdiv(110·lvl, lvl+6), 100).
- `edln` = ELen + ELevLen1·(levels 2–8) + ELevLen2·(9–16) + ELevLen3·(17+).

| Source | stat | formula | lvl 1 | 5 | 10 | 20 | 30 |
|---|---|---|---|---|---|---|---|
| Burst of Speed (Quickness) | 67 | edln 20 / 2 / 1 / 1 | 20 | 28 | 36 | 46 | 56 |
| Increased Speed (passive) | 67 | dm12 7→50 | 13 | 28 | 36 | 43 | 46 |
| Vigor aura (owner + party) | 67 | dm56 7→50 | 13 | 28 | 36 | 43 | 46 |
| Vigor passive (owner only, PD2 passivestats) | 96 FRW | blvl, then diminished | 1 | 5 | 10 | 20 | – |
| Stamina shrine (PD2) | 67 | +35 flat, 96 s | | | | | |
| Holy Freeze (enemy aura) | 67 | −dm34 25→60, **floored** | −30 | −42 | −48 | −54 → −50 | −56 → −50 |
| Decrepify (curse) | 67 | −(9 + lvl), **not floored** | −10 | −14 | −19 | −29 | −39 |
| Terror (curse) | 67 | −(15 + 2·blvl + CurMas blvl) | | | | | |
| Chill (cold hit) | 67 | player −50; monster MonStats `coldeffect[diff]` | | | | | |

- **Chill and aura slow floor:** D2Game `0x6FCFAE20` returns −50 for players and MonStats `coldeffect` (normal / nightmare / hell) for monsters.
  - Chill (`0x6FCFC780`) uses it as the chill value.
  - The aura target callback `0x6FCBA1A0` uses it as a **floor** for stats 67/68 (`0x6FCBA2EB`): `value = max(value, coldEffect)`.
  - So Holy Freeze and other aura slows cannot slow a player below −50. On a monster they cannot slow it more than its cold effect (e.g. −33 or −25 in Hell, **0 for monsters whose coldeffect is 0**).
- The curse path (`0x6FC70250`) does not call it, so Decrepify/Terror slows are unfloored and stack on top.
- Feral Rage has `velocitypercent dm34` 20→120 in the aura list. How charges scale it (state setfunc 16) was not traced.
- Frenzy (`edln` 32 / 2 / 1 / 1), Evade (`edmx`) and Joust (`min(3+2·lvl, 65)`) are listed in Skills.txt but were not traced.

### 3.3 Worked examples (runSpeed in mf_misc.js; formula VERIFIED)

| Situation | velocitypercent | FRW | s | subtiles / s |
|---|---|---|---|---|
| Walk | 100 | 0 | 100 | 9.38 |
| Run | 150 | 0 | 150 | 14.06 |
| Run, 40 FRW (EFRW 31) | 150 | 40 | 181 | 16.97 |
| Run, 40 FRW, BoS lvl 20 (+46) | 196 | 40 | 227 | 21.28 |
| Run, PD2 stamina shrine (+35) | 185 | 0 | 185 | 17.34 |
| Run, 40 FRW, stamina shrine | 185 | 40 | 216 | 20.25 |
| Run, chilled (−50) | 100 | 0 | 100 | 9.38 |
| Run, 40 FRW, chilled | 100 | 40 | 131 | 12.28 |
| Run in a lvl-20 Holy Freeze (−54 → −50) | 100 | 0 | 100 | 9.38 |
| Run, chilled + Decrepify lvl 20 (−29) | 71 | 0 | 71 | 6.66 |
| Walk, chill + Decrepify 20 + Terror (−57) | −36 → floor | 0 | 25 | 2.34 |

---

## 4. Verification summary

| Item | How | Result |
|---|---|---|
| Shrine type selection `0x6FC4EF90` + `0x6FC4ED70` | native, 120k cases incl. both RNG seeds | 0 mismatches (mutation: 5,995) |
| Booster value helper `0x6FC89EE0` (all stats except 19 and 21) | native, 40k | 0 (mutation: 2,905) |
| Stamina drain `0x6FC97BB0` with body armor speed, stat 154, town | native, 40k | 0 (mutation: 17,055) |
| Type probabilities | Monte Carlo 3×100k vs closed form (not in the shipped harness) | within ±0.2 points (READ) |
| Stamina regen, walk/run velocity | earlier harnesses (block_regen.md, mf_misc.md) | VERIFIED |
| Effects, stacking, resets, wells, spawn rules, PD2 stamina hook, slow floors | disassembly | READ |

- Harness files: `harness/shrine.c`, `harness/shrine.py`, `harness/pd2_shrines.bin` (a copy of `shrines.bin` from `pd2data.mpq`).
- Not verified natively:
  - the whole operate path (`0x6FC8DA00` and `0x6FCC00B0`);
  - the combat AR helper `0x6FC89B00`;
  - the PD stamina hook;
  - the populate/spawn counters;
  - storm, gem and warping;
  - well heals;
  - Feral Rage charge scaling;
  - the death clearing of shrine states.
