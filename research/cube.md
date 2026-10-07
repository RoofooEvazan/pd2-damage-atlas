# PD2 Horadric Cube: recipe matching, corruption, currency and crafting

Scope: the PD2 client install (1.13c D2Game/D2Common + ProjectDiablo.dll "PD", base 0x10000000). Realm servers may run other code or data.

Status labels (AGENT_CONTEXT.md):
- **VERIFIED**: the real code was run natively in the harness and matched a model, with a mutation check.
- **READ**: from the disassembly only.
- **DATA**: from the tables.

Files:
- `adv/re/cube.js`: UMD model (`window.PD2Cube`); corruption odds, the brick re-roll engine, cube output level, crafted-affix odds, recipe lookup.
- `adv/re/cube-data.json`: data for cube.js, built by `adv/re/extract_cube.py` from the live MPQ tables.
- `adv/re/test_cube.js`: checks cube.js against the native vectors (20,000/20,000) and checks that every distribution sums to 1.
- `harness/cube/cube.c`: native run of PD's corruption engine 0x102BB5F0 against the real MPQ `cubemain.bin`.
- `harness/cube/craft.c`: native run of the stock crafted-affix roller D2Game 0x6FC34BF0, with PD's patch applied.

Related notes, reused and not redone:
- `maps.md` §4: map currency. Its corruption table was built from the stale `data.zip` copy; §2.5 below replaces it.
- `FINDINGS.md` "Correction: corruption sockets": still true, but the MPQ row numbers moved (item corruption is now rows 419–1551).
- `drops.md`: `map_glob_dropcorrupted`.
- `items.md` (being written in parallel): affix picking, alvl, and the rare/magic affix rules. This note only says which item-generation call the cube makes.

> **Which data copy.** Every row number and value below is from the **MPQ** copy (`bin/pd2data.mpq` → `excel_mpq/CubeMain.txt`, 2,341 rows). The compiled `cubemain.bin` inside the MPQ has the same 2,341 records (0x148 bytes each), and spot checks of the record fields match the text (for example row 341's corruptor range 1..3000, and row 453's threshold 325).
> - The `data.zip` copy (2,300 rows) differs. There, the map corruptor is 1..1000 and the thresholds are 90/180/…/1000, with no Djinn/Na-Krul rows.
> - The *proportions* for T1–T3 are the same in both copies. Only the unique-map slices differ.
> - The MPQ property ids in the `.bin` are Properties.txt row indexes with the `Expansion` separator row skipped. For example, 0x10E = `corrupt` and 0x1A9 = `desecrate`.

> **Correction (audit):** the zip `CubeMain.txt` has **2,302** rows (and `cubemain.bin` 2,302 records), not 2,300.


---------------------------------------------------------------------------------------------------------------------

## 1. The cube pipeline

### 1.1 Call chain (READ)
PD2 replaces the stock transmute but keeps the stock helpers for the input test and the output.

| step | code | how it is reached |
|---|---|---|
| transmute entry | **PD 0x102BDB70** | stock D2Game call at 0x6FC934F2 patched to PD (patch record) |
| candidate list + refusal rules | PD 0x102BCC00 | called by 0x102BDB70 |
| player-stat op test | PD 0x102BC7F0 | per candidate |
| full input match | PD 0x102BBA50 → stock **D2Game 0x6FC90240** per input (through PD 0x102ECC90) | per candidate |
| output creation | PD 0x102BBFF0 → stock **D2Game 0x6FC92220** (PD thunk 0x102826C0) | once, for the first matching row |
| corruption engine | PD 0x102BB5F0 | after "ladder 3" rows; also at item creation (§2.6) |

The stock D2Game call sites 0x6FC93463 and 0x6FC93493 are also redirected, to PD 0x102ECC70 and PD 0x102BBFF0.

### 1.2 Recipe record (`cubemain.bin`, 0x148 bytes; DATA + READ)

| offset | field | notes |
|---|---|---|
| +0x00 | enabled | disabled rows are skipped by the candidate builder and by the engine |
| +0x01 | **ladder** | repurposed by PD as a *phase* tag: 1 = corruption outcome, 2 = corruption phase 1, 3 = "destroyed → rare", 4 = map currency. PD never checks ladder status. |
| +0x02 | min diff | must be ≤ the game difficulty (0x102BD2A5) |
| +0x03 | class | 0xFF = any, else the player's class |
| +0x04 | op | §1.4 |
| +0x08 / +0x0C | param / value | |
| +0x10 | numinputs | |
| +0x12 | version | PD only tests the value **200**: those rows refuse circlets (§4.1). Rows with version 2 are active too. |
| +0x14 + 8·i | input i (7) | word flags, word item/type id, byte +6 quality, byte +7 quantity |
| +0x4C + 0x54·k | output A/B/C | word flags; word item id; byte +6 quality; byte +7 qty (or socket count for `sock=N`); byte +8 kind; byte +9 lvl; byte +0xA plvl; byte +0xB ilvl; words +0xC… prefix/suffix; +0x18: 5 mods of 12 bytes `{dword prop, word param, word min, word max, byte chance}` |

**Input flag bits** (DATA from the bin, meaning READ in 0x6FC90240):

| bit | meaning |
|---|---|
| 0x1 | specific item code (id 0xFFFF = `any`) |
| 0x2 | ItemTypes code (with Equiv) |
| 0x4 | `nos` (no sockets) |
| 0x8 | `sock` (socketed) |
| 0x10 | `eth` |
| 0x20 | `noe` |
| 0x40 | named unique or set, e.g. `Annihilus`, `The Stone of Jordan` (the index is at +4) |
| 0x100 / 0x200 / 0x400 | `bas` / `exc` / `eli` |

Quality byte: 1 low, 2 nor, 3 hiq, 4 mag, 5 set, 6 rar, 7 uni, 8 crf, 9 tmp.

**Output kind byte:**

| value | meaning |
|---|---|
| 0xFE | `useitem`: the same unit is kept and modified |
| 0xFF | `usetype`: a new item of the **same base (class id) as the matching input** |
| 0xFC | a specific item code |
| 0xFD | a random base of an ItemTypes code, e.g. `t2me` |

**Output flag bits:**

| bit | meaning |
|---|---|
| 0x1 | `mod`: keep mods on an upgrade |
| 0x2 | `sock=N` |
| 0x10 | `uns`: destroy socketed items |
| 0x40 | regenerate the same unique (stock `reg`; no live row uses it) |
| 0x80 | `exc` |
| 0x100 | `eli` |
| 0x200 | `rep` |
| 0x400 | `rch` |

### 1.3 What one transmute does (PD 0x102BDB70; READ)
1. **Throttle.** If `game+0x25FC + 10 > game frame (+0xA8)`, the click is ignored. Otherwise the frame is stored (0x102BC770).
2. **Collect.** The items in the cube (inventory page 3) are collected (0x102BC790).
3. **Candidates.** PD 0x102BCC00 builds the candidate list, or refuses everything (§1.5).
4. **Orb flag.** 0x102BB580 is true when a `wss` or `cwss` is in the cube and no shard carries stat 360.
5. **Find the first match.** Walk the candidates **in file order**:
   - Skip rows with ladder 1 or 3 when the orb flag is false.
   - Run the op test (0x102BC7F0), then the full input match (0x102BBA50).
   - **The first row that passes wins.** Nothing else is tried.
6. **Special ops:**
   - op 30 (BLOCK): nothing happens.
   - op 31 (Demonic Cube reroll): PD 0x102BD5E0.
   - op 28 with min diff 2 and param 1 (row 317, PD2 Pandemonium key): PD 0x102BD370 opens a portal to one of the three not-yet-opened levels 133/134/135. The level is picked by `rand() % 3`, with up to 40 retries. Flags are kept in `game+0x2628`.
7. **Outputs.** Outputs are created by 0x102BBFF0 (§1.6).
8. **Ladder 2 (corruption phase 1).** If a shard is still in the cube, `game+0x25FC` is cleared and **the transmute runs again at once** (a recursive call to 0x102BDB70).
9. **Ladder 3 (destroyed → rare).** For every item left in the cube except the shard (type `corr`):
   - If an input item was ethereal, the new item is made ethereal with D2Common #10890 (SetEthereal).
   - Then **the corruption engine runs on it** with flag 0 (§2.3).
10. **Stacks.** Stackable items (types book, scro, tpot, key, misc, corr, imma…lbox, gsm…r1pg, ubr, ubor, jewf) whose quantity is outside the ItemsTxt min/max stack are fixed up (0x102C05C0).

### 1.4 Op codes (READ)
PD 0x102BC7F0 tests **player** stats (param = stat, value compared after the ItemStatCost shift):

| op | test | notes |
|---|---|---|
| 1, 2 | always false | |
| 3 / 7 / 11 | player stat ≤ value | three different stat getters: #10973, #10587, #10379 |
| 4 / 8 / 12 | ≥ | |
| 5 / 9 / 13 | ≠ | |
| 6 / 10 / 14 | = | |
| 15–27, 29, 30, ≥31 | true here | ops 15–27 are tested on the item instead (below) |
| 28 | param 1: player in level 109 (Harrogath); param 2: cow-level quest / level checks | |

Stock D2Game 0x6FC905B8 tests **the item in input 1** (only input index 0):

| op | test | getter |
|---|---|---|
| 15 / 16 / 17 / 18 | stat ≥ / ≤ / ≠ / = value | GetStat |
| 19–22 | same four tests | base stat getter |
| 23–26 | same four tests | a third getter |

**PD2 ops:**
- **op 30 = BLOCK.** The candidate builder always adds enabled op-30 rows, whatever their input counts. If the row matches, the cube does nothing. Used for white maps with std/wss/fort/iwss/ears, unique maps with ears, "map + 3 perfect gems", "map + 6 skulls", and PvP charms.
- **op 31 = Demonic Cube** (§3).
- **op 16/18 on stat 361 (`corruptor`)** drives corruption (§2).
- op 18 on stat 185 (`uber_difficulty`) cycles uber maps.
- op 18 on stat 458 (`heroic`) and stat 276 (Invader ears) refuses a second application.

### 1.5 Candidate list and refusals (PD 0x102BCC00; READ)
The whole cube is refused (nothing happens) when any of these holds:

| condition | refused when |
|---|---|
| a `wss`/`cwss` is present | more than 2 items are in the cube, or **any item has stat 360 (corrupted)** |
| any item has stat 486 (`mirrored`) | always |
| a Lilith's Mirror (`llmr`) is present | more than 2 items |
| a `fort` is present | the map is already fortified (stat 493 set), or more than 2 items |
| an `iwss` is present | the map already has stat 500 (`map_force_event`), or more than 2 items |
| a Lightsong vial (`lsvl`) is present | more than 2 items, or the item is a Phase Blade (`7cr`), indestructible (stat 152), or its own type is `miss`/`bow`/`xbow`/`abow`/`tpot` |
| a Puzzlebox (`lbox`) or Larzuk's Malus (`lmal`) is present | an item's own type is `thro`, `tpot` or `misl` (throwing knives, axes and javelins are allowed) |
| a Puzzle piece (`lpp`) is present | the target is a **set or unique**, or has one of the types above |
| a Demonic Cube (`imrn`) is present | no set or unique item is present, or a corrupted item is present |
| `rkey` and `key` are both present | always |
| an item of PD's special code set (0x102CE7C0, not extracted) is present | `llmr`, `lsvl`, `wss`, `lbox`, `lpp`, runes/rune stacks or jewels/jewel fragments are also present |

Otherwise every enabled row is a candidate if **all** of these hold:
- its min diff ≤ the difficulty;
- its class matches;
- every input is matched with the right count (0x102BBDF0);
- it has at most 2 inputs when a shard is present;
- if it has one input and the cube holds more than one item, none of those items is stackable (0x102BD2D5).

Row order is kept.

**Input test (stock 0x6FC90240).** Each input checks, in this order:
- the item code or ItemTypes type (Equiv chains; type **and** `type2` through #10744);
- the named unique;
- quality;
- `nos`/`sock`;
- `eth`/`noe`;
- `bas`/`exc`/`eli`;
- for input 1 only, the op 15–27 stat test.

`qty=N` counts stack quantity: jewel fragments count as `jewg` and rune stacks as `rNNg`.

### 1.6 Outputs (stock D2Game 0x6FC92220 with PD pre/post steps; READ)

**Output item level** (0x6FC92289–0x6FC9240A):
```
if (lvl)  L = lvl
else      L = trunc(plvl * clvl / 100) + trunc(ilvl * inputLevel / 100)   // inputLevel = level of the input with the
                                                                          // same index as the output (A <-> input 1)
L = clamp(L, 1, 99)                                                       // D2Common #10066 = Experience.txt MaxLvl
```
- `useitem` outputs keep the item and its level; L is not applied.
- `usetype`/code outputs are generated at level L with the output's quality.
- `0xFD` (a type code) picks, uniformly, one of up to 256 spawnable bases of that type whose `level` ≤ L (D2Game 0x6FC91060). "3 T1 maps + upmp → `t2me,nor`" works this way.

**Mods** (0x6FC928BE): each of the 5 mods is applied unless `0 < chance < 100` **and** `seed % 100 > chance`. So a chance `c` succeeds with probability **(c+1)/100**. No live row uses a partial chance; crafted rows leave it blank, which means always. The value is rolled in min..max by the property function.

**Other output rules:**
- **Sockets:** `sock` min..max, capped by D2Common 0x6FD74610 = min(`gemsockets`, ItemTypes MaxSock1/25/40 by item level) (FINDINGS).
- **Upgrades:** `useitem,mod,exc|eli` upgrades the base and keeps the item.
- **PD armour defense (PD 0x102BC396–0x102BC615):** for armour inputs, PD records where the defense sits in the old base's min..max. Ethereal defense is divided by 1.25 first. It rolls a fraction inside the ±0.5 rounding window and writes `min' + (max'−min')·fraction` (×1.25 if ethereal) as base defense 31 on the new base. **An upgrade keeps the defense roll's percentile.**
- **Stacks (PD 0x102BC0D0/0x102BC171):** an output code that is already in the cube as a stack is added to that stack instead of creating a new item. This is how "Jewel → Jewel Fragments" and the rune/gem stack rows work.

---------------------------------------------------------------------------------------------------------------------

## 2. Item corruption (Worldstone Shard `wss`, Corrupted Worldstone Shard `cwss`)

### 2.1 How a corruption works (DATA + READ)
Two stats, both from ItemStatCost:
- **360 `corrupted`**: the outcome code. It is shown as the "Corrupted" line (descfunc 3), and PD refuses a second shard when it is set.
- **361 `corruptor`**: the 1..N roll, then 1001 (items and T4 maps) or 3001 (T1–T3 maps) once corrupted.

Two passes, in one click:
1. **Phase 1** (rows with ladder 2, `op 18 361 = 0`): for example row 419 `weap,nos + wss`, row 420 `armo,nos`, row 423 `ring`, row 341 `map`, row 425 `Annihilus + cwss`, rows 830–844 for socketed magic/rare/set/unique/crafted items.
   - Output A = `useitem {corruptnum 1..1000}` (maps: **1..3000**).
   - Output B gives the shard back.
2. PD re-runs the transmute at once (§1.3 step 8). The cube now holds the item (361 = X) and one shard.
3. **Phase 2:** the first row in file order whose input 1 matches the item and whose `op 16 361 ≤ value` holds wins.
   - Outcome rows (ladder 1) are `useitem { corrupt = code, corruptnum = 1001, the outcome stats }`.
   - Brick rows (ladder 3) are `usetype,rar ilvl=100`.
   - The shard is consumed.

**Resulting odds.** Within the rows that can match the item, P(row k) = (value_k − value_{k−1}) / N. Rows are listed in rising order inside each block. The phase-1 roll is a stock property roll on the item seed, uniform on min..max (READ, stock; bias below 10⁻⁶).

**Items that can be corrupted** (DATA, MPQ):
- unsocketed weapons, armour (including boots, gloves and belts), rings, amulets and quivers, of any quality;
- socketed magic, rare, set, unique or crafted weapons, armour and quivers;
- Annihilus (with `cwss` only);
- T1–T4 maps that are not white.

Items that **cannot** be corrupted:
- socketed white, superior or low-quality items (so no runewords): no phase-1 row matches, and the shard stays in the cube;
- jewels and charms other than Annihilus;
- white maps (op-30 BLOCK row 339);
- anything already corrupted (refused, §1.5).

### 2.2 Outcome tables by item type (DATA, MPQ rows; the first-match rule is READ)
These tables are generated by `PD2Cube.corrupt()`, which walks the rows exactly as §2.1 describes. "p each" is the chance of each listed row.

**Row order matters in two places:**
- Bows, crossbows and two-handers (type `2han`, which PD2 puts in `type2`) meet their own socket rows (453–464) before the generic weapon socket rows 501–503. Those generic rows are never reached for them.
- The throwing-weapon block (rows 468–499) is **disabled**, so throwing weapons use the generic weapon rows.

**Sockets:** the value in the row is the new socket count, capped by the base (`gemsockets`) and by ItemTypes MaxSock for the item level. For example, a Shako (`gemsockets` 2) that rolls "sock 3" gets 2.

**One-hand weapon (e.g. Phase Blade), magic/rare/set/unique/crafted, no sockets** (phase-1 row 419, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 452 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–370 | 12 % | 501 | sock 2 |
| 371–440 | 7 % | 502 | sock 3 |
| 441–500 | 6 % | 503 | sock 4 |
| 501–750 | 2.5 % × 10 | 544–553 | dmg% 40..80; att 150..250; heal-hit 3..6; att-demon 200, dmg-demon 100..150; ease -25..-50; mag% 20..30; heal-kill 3..5; mana-kill 3..5; cast2 10; att-undead 200, dmg-undead 100..150 |
| 751–900 | 1.5 % × 10 | 555–564 | pierce-fire 7..10; pierce-ltng 7..10; pierce-cold 7..10; pierce-pois 7..10; cast2 20; lifesteal 5, dmg% 40..60; dmg-ac -40..-60; deadly 20..30; swing1 30..40; crush 20..30 |
| 901–1000 | 1 % × 10 | 566–575 | swing1 20, dmg% 80..120; swing1 30, crush 20..30; ignore-ac 1, dmg% 60..80; deadly 25, dmg% 50..70; att 250, dmg% 80..120; allskills 1; extra-fire 5, cast2 10; extra-cold 5, cast2 10; extra-ltng 5, cast2 10; extra-pois 5, cast2 10 |

**Bow / crossbow / two-hander (e.g. Crusader Bow), magic..crafted, no sockets** (phase-1 row 419, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 452 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–325 | 7.5 % | 453 | sock 3 |
| 326–395 | 7 % | 454 | sock 4 |
| 396–455 | 6 % | 455 | sock 5 |
| 456–500 | 4.5 % | 456 | sock 6 |
| 501–750 | 2.5 % × 10 | 544–553 | dmg% 40..80; att 150..250; heal-hit 3..6; att-demon 200, dmg-demon 100..150; ease -25..-50; mag% 20..30; heal-kill 3..5; mana-kill 3..5; cast2 10; att-undead 200, dmg-undead 100..150 |
| 751–900 | 1.5 % × 10 | 555–564 | pierce-fire 7..10; pierce-ltng 7..10; pierce-cold 7..10; pierce-pois 7..10; cast2 20; lifesteal 5, dmg% 40..60; dmg-ac -40..-60; deadly 20..30; swing1 30..40; crush 20..30 |
| 901–1000 | 1 % × 10 | 566–575 | swing1 20, dmg% 80..120; swing1 30, crush 20..30; ignore-ac 1, dmg% 60..80; deadly 25, dmg% 50..70; att 250, dmg% 80..120; allskills 1; extra-fire 5, cast2 10; extra-cold 5, cast2 10; extra-ltng 5, cast2 10; extra-pois 5, cast2 10 |

**Weapon, magic..crafted, socketed** (phase-1 row 839, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 850 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–287 | 3.7 % | 966 | dmg% 40..80 |
| 288–325 | 3.8 % | 967 | att 150..250 |
| 326–362 | 3.7 % | 968 | heal-hit 3..6 |
| 363–400 | 3.8 % | 969 | att-demon 200, dmg-demon 100..150 |
| 401–437 | 3.7 % | 970 | ease -25..-50 |
| 438–475 | 3.8 % | 971 | mag% 20..30 |
| 476–512 | 3.7 % | 972 | heal-kill 3..5 |
| 513–550 | 3.8 % | 973 | mana-kill 3..5 |
| 551–587 | 3.7 % | 974 | cast2 10 |
| 588–625 | 3.8 % | 975 | att-undead 200, dmg-undead 100..150 |
| 626–647 | 2.2 % | 977 | pierce-fire 7..10 |
| 648–670 | 2.3 % | 978 | pierce-ltng 7..10 |
| 671–692 | 2.2 % | 979 | pierce-cold 7..10 |
| 693–715 | 2.3 % | 980 | pierce-pois 7..10 |
| 716–737 | 2.2 % | 981 | cast2 20 |
| 738–760 | 2.3 % | 982 | lifesteal 5, dmg% 40..60 |
| 761–782 | 2.2 % | 983 | dmg-ac -40..-60 |
| 783–805 | 2.3 % | 984 | deadly 20..30 |
| 806–827 | 2.2 % | 985 | swing1 30..40 |
| 828–850 | 2.3 % | 986 | crush 20..30 |
| 851–1000 | 1.5 % × 10 | 988–997 | swing1 20, dmg% 80..120; swing1 30, crush 20..30; ignore-ac 1, dmg% 60..80; deadly 25, dmg% 50..70; att 250, dmg% 80..120; allskills 1; extra-fire 5, cast2 10; extra-cold 5, cast2 10; extra-ltng 5, cast2 10; extra-pois 5, cast2 10 |

**Weapon, low-quality / normal / superior, no sockets (armour rows 432–433/441–442/447–448 and quiver rows 434–435 are identical)** (phase-1 row 419, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–500 | 50 % | 429 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 501–1000 | 50 % | 430 | sock 1..6 |

**Body armour, magic..crafted, no sockets** (phase-1 row 420, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 508 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–370 | 12 % | 512 | sock 1 |
| 371–440 | 7 % | 513 | sock 2 |
| 441–500 | 6 % | 514 | sock 3 |
| 501–740 | 3 % × 8 | 578–585 | ac% 50..80; mag% 20..30; balance1 20..30; res-fire 30..35; res-cold 30..35; res-ltng 30..35; res-pois 30..35; mana 30..40 |
| 741–900 | 2 % × 8 | 587–594 | thorns/lvl 32..48; cast1 10; hp% 4..6; move1 20; nofreeze 1; red-dmg 6..10; red-mag 6..10; indestruct 1, ac% 50..80 |
| 901–912 | 1.2 % | 596 | curse-effectiveness 10 |
| 913–925 | 1.3 % | 597 | allskills 1 |
| 926–937 | 1.2 % | 598 | res-all 20..25 |
| 938–950 | 1.3 % | 599 | red-dmg% 6..8 |
| 951–962 | 1.2 % | 600 | res-fire-max 4..5, res-fire 15 |
| 963–975 | 1.3 % | 601 | res-cold-max 4..5, res-cold 15 |
| 976–987 | 1.2 % | 602 | res-ltng-max 4..5, res-ltng 15 |
| 988–1000 | 1.3 % | 603 | res-pois-max 4..5, res-pois 15 |

**Body armour, magic..crafted, socketed** (phase-1 row 840, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 856 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 1115–1122 | ac% 50..80; mag% 20..30; balance1 20..30; res-fire 30..35; res-cold 30..35; res-ltng 30..35; res-pois 30..35; mana 30..40 |
| 611–850 | 3 % × 8 | 1124–1131 | thorns/lvl 32..48; cast1 10; hp% 4..6; move1 20; nofreeze 1; red-dmg 6..10; red-mag 6..10; indestruct 1, ac% 50..80 |
| 851–907 | 1.9 % × 3 | 1133–1135 | curse-effectiveness 10; allskills 1; res-all 20..25 |
| 908–925 | 1.8 % | 1136 | red-dmg% 6..8 |
| 926–982 | 1.9 % × 3 | 1137–1139 | res-fire-max 4..5, res-fire 15; res-cold-max 4..5, res-cold 15; res-ltng-max 4..5, res-ltng 15 |
| 983–1000 | 1.8 % | 1140 | res-pois-max 4..5, res-pois 15 |

**Helm, magic..crafted, no sockets** (phase-1 row 420, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 509 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–370 | 12 % | 516 | sock 1 |
| 371–440 | 7 % | 517 | sock 2 |
| 441–500 | 6 % | 518 | sock 3 |
| 501–740 | 3 % × 8 | 606–613 | ac% 50..80; regen 20..30; balance1 20..30; res-fire 30..35; res-cold 30..35; res-ltng 30..35; res-pois 30..35; mag% 20..30 |
| 741–900 | 2 % × 8 | 615–622 | indestruct 1, ac% 50..80; lifesteal 3..5; manasteal 3..5; hp% 4..6; nofreeze 1; heal-kill 3..4; mana-kill 3..4; att 150..250, light 2..4 |
| 901–912 | 1.2 % | 624 | curse-effectiveness 10 |
| 913–925 | 1.3 % | 625 | allskills 1 |
| 926–937 | 1.2 % | 626 | res-all 15..20 |
| 938–950 | 1.3 % | 627 | red-dmg% 4..6 |
| 951–962 | 1.2 % | 628 | res-fire-max 4..5, res-fire 15 |
| 963–975 | 1.3 % | 629 | res-cold-max 4..5, res-cold 15 |
| 976–987 | 1.2 % | 630 | res-ltng-max 4..5, res-ltng 15 |
| 988–1000 | 1.3 % | 631 | res-pois-max 4..5, res-pois 15 |

**Helm, magic..crafted, socketed** (phase-1 row 840, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 856 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 1252–1259 | ac% 50..80; regen 20..30; balance1 20..30; res-fire 30..35; res-cold 30..35; res-ltng 30..35; res-pois 30..35; mag% 20..30 |
| 611–850 | 3 % × 8 | 1261–1268 | indestruct 1, ac% 50..80; lifesteal 3..5; manasteal 3..5; hp% 4..6; nofreeze 1; heal-kill 3..4; mana-kill 3..4; att 150..250, light 2..4 |
| 851–907 | 1.9 % × 3 | 1270–1272 | curse-res 10; allskills 1; res-all 15..20 |
| 908–925 | 1.8 % | 1273 | red-dmg% 4..6 |
| 926–982 | 1.9 % × 3 | 1274–1276 | res-fire-max 4..5, res-fire 15; res-cold-max 4..5, res-cold 15; res-ltng-max 4..5, res-ltng 15 |
| 983–1000 | 1.8 % | 1277 | res-pois-max 4..5, res-pois 15 |

**Shield, magic..crafted, no sockets** (phase-1 row 420, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 510 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–370 | 12 % | 520 | sock 1 |
| 371–440 | 7 % | 521 | sock 2 |
| 441–500 | 6 % | 522 | sock 3 |
| 501–740 | 3 % × 8 | 746–753 | block2 20; mag% 20..30; balance1 20..30; res-fire 35..40; res-cold 35..40; res-ltng 35..40; res-pois 35..40; hp 30..40 |
| 741–900 | 2 % × 8 | 755–762 | thorns/lvl 32..48; hp% 4..6; block 10..20, block1 10; indestruct 1, ac% 50..80; nofreeze 1; red-dmg 6..10; red-mag 6..10; cast1 10 |
| 901–912 | 1.2 % | 764 | curse-effectiveness 10 |
| 913–925 | 1.3 % | 765 | allskills 1 |
| 926–937 | 1.2 % | 766 | res-all 20..25 |
| 938–950 | 1.3 % | 767 | red-dmg% 6..8 |
| 951–962 | 1.2 % | 768 | res-fire-max 4..5, res-fire 15..20 |
| 963–975 | 1.3 % | 769 | res-cold-max 4..5, res-cold 15..20 |
| 976–987 | 1.2 % | 770 | res-ltng-max 4..5, res-ltng 15..20 |
| 988–1000 | 1.3 % | 771 | res-pois-max 4..5, res-pois 15..20 |

**Shield, magic..crafted, socketed** (phase-1 row 840, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 856 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 1389–1396 | block2 20; mag% 20..30; balance1 20..30; res-fire 35..40; res-cold 35..40; res-ltng 35..40; res-pois 35..40; hp 30..40 |
| 611–850 | 3 % × 8 | 1398–1405 | thorns/lvl 32..48; hp% 4..6; block 10..20, block1 10; indestruct 1, ac% 50..80; nofreeze 1; red-dmg 6..10; red-mag 6..10; cast1 10 |
| 851–907 | 1.9 % × 3 | 1407–1409 | curse-effectiveness 10; allskills 1; res-all 20..25 |
| 908–925 | 1.8 % | 1410 | red-dmg% 6..8 |
| 926–982 | 1.9 % × 3 | 1411–1413 | res-fire-max 4..5, res-fire 15..20; res-cold-max 4..5, res-cold 15..20; res-ltng-max 4..5, res-ltng 15..20 |
| 983–1000 | 1.8 % | 1414 | res-pois-max 4..5, res-pois 15..20 |

**Boots, magic..crafted** (phase-1 row 420, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 505 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 634–641 | ac% 50..80; mag% 10..25; gold% 50..100; res-fire 15..20; res-cold 15..20; res-ltng 15..20; res-pois 15..20; regen-mana 10..15 |
| 611–850 | 3 % × 8 | 643–650 | block2 10; hp 20..40; indestruct 1, ac% 50..80; res-all 5..8; regen 15..25; heal-kill 2..3; mana-kill 2..3; balance1 10 |
| 851–907 | 1.9 % × 3 | 652–654 | move1 15; block 10; curse-res 20 |
| 908–925 | 1.8 % | 655 | red-dmg% 3..4 |
| 926–982 | 1.9 % × 3 | 656–658 | res-fire-max 2..3, res-fire 10; res-cold-max 2..3, res-cold 10; res-ltng-max 2..3, res-ltng 10 |
| 983–1000 | 1.8 % | 659 | res-pois-max 2..3, res-pois 10 |

**Gloves, magic..crafted** (phase-1 row 420, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 506 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 662–669 | ac% 50..80; mag% 10..25; gold% 50..100; res-fire 10..15; res-cold 10..15; res-ltng 10..15; res-pois 10..15; regen-mana 20..30 |
| 611–850 | 3 % × 8 | 671–678 | att 100..150; hp 20..40; pierce 10..15; block1 10..20; regen 15..20; lifesteal 2..3; manasteal 2..3; all-stats 4..6 |
| 851–907 | 1.9 % × 3 | 680–682 | deadly 10; mana-kill 3..4; cast1 10 |
| 908–925 | 1.8 % | 683 | res-all 5..8 |
| 926–982 | 1.9 % × 3 | 684–686 | swing1 10; reduce-ac 15..25; block 10 |
| 983–1000 | 1.8 % | 687 | dmg% 30..40 |

**Belt, any quality** (phase-1 row 420, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 507 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 690–697 | str 7..10; dex 7..10; vit 7..10; enr 7..10; res-fire 10..15; res-cold 10..15; res-ltng 10..15; res-pois 10..15 |
| 611–850 | 3 % × 8 | 699–706 | thorns/lvl 16..32; all-stats 4..6; pierce 10..15; att 100..150; regen 15..20; mag% 20..30; gold% 60..100; balance1 10 |
| 851–907 | 1.9 % × 3 | 708–710 | move1 10; res-all 5..8; curse-res 20 |
| 908–925 | 1.8 % | 711 | red-dmg% 3..4 |
| 926–982 | 1.9 % × 3 | 712–714 | cast1 10; block 10; swing1 10 |
| 983–1000 | 1.8 % | 715 | res-all-max 2 |

**Ring, any quality** (phase-1 row 423, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 529 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 774–781 | str 7..10; dex 7..10; vit 7..10; enr 7..10; res-fire 10..15; res-cold 10..15; res-ltng 10..15; res-pois 10..15 |
| 611–850 | 3 % × 8 | 783–790 | hp 30..40; mag% 15..20; gold% 40..80; red-dmg 4..6; red-mag 4..6; heal-kill 2..3; mana-kill 2..3; att 100..150 |
| 851–907 | 1.9 % × 3 | 792–794 | move1 10; red-dmg% 3; curse-res 10 |
| 908–925 | 1.8 % | 795 | lifesteal 3..4 |
| 926–982 | 1.9 % × 3 | 796–798 | manasteal 3..4; cast1 10; res-all 4..6 |
| 983–1000 | 1.8 % | 799 | all-stats 4..6 |

**Amulet, any quality** (phase-1 row 422, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 531 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 802–809 | str 6..12; dex 6..12; vit 6..12; enr 6..12; res-fire 15..20; res-cold 15..20; res-ltng 15..20; res-pois 15..20 |
| 611–850 | 3 % × 8 | 811–818 | all-stats 6..8; mag% 20..30; gold% 60..100; pierce 10..20; block 10; heal-kill 2..3; mana-kill 2..3; balance1 10 |
| 851–907 | 1.9 % × 3 | 820–822 | move1 10; dmg% 30..40; allskills 1 |
| 908–925 | 1.8 % | 823 | curse-effectiveness 10 |
| 926–982 | 1.9 % × 3 | 824–826 | res-all-max 2; cast1 10; res-all 7..10 |
| 983–1000 | 1.8 % | 827 | nofreeze 1 |

**Quiver (arrows/bolts), magic..crafted, no sockets** (phase-1 row 424, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 524 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–430 | 18 % | 526 | sock 1 |
| 431–500 | 7 % | 527 | sock 2 |
| 501–740 | 3 % × 8 | 718–725 | balance2 20; hp 30..40; mag% 10..25; gold% 50..100; res-fire 10..20; res-cold 10..20; res-ltng 10..20; res-pois 10..20 |
| 741–900 | 2 % × 8 | 727–734 | att 50..100; reduce-ac 10..20; pierce 10..15; pierce-fire 5..10; pierce-cold 5..10; lifesteal 2..4; manasteal 2..4; move1 10 |
| 901–912 | 1.2 % | 736 | dmg-min 10..15 |
| 913–925 | 1.3 % | 737 | res-all 5..10 |
| 926–937 | 1.2 % | 738 | curse-effectiveness 10 |
| 938–950 | 1.3 % | 739 | dmg-max 10..15 |
| 951–962 | 1.2 % | 740 | dmg% 30..40 |
| 963–975 | 1.3 % | 741 | ignore-ac 1 |
| 976–987 | 1.2 % | 742 | swing1 10 |
| 988–1000 | 1.3 % | 743 | allskills 1 |

**Quiver, magic/rare/unique/crafted, socketed** (phase-1 row 424, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 524 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–610 | 4.5 % × 8 | 1472–1479 | balance2 20; hp 30..40; mag% 10..25; gold% 50..100; res-fire 10..20; res-cold 10..20; res-ltng 10..20; res-pois 10..20 |
| 611–850 | 3 % × 8 | 1481–1488 | att 50..100; reduce-ac 10..20; pierce 10..15; pierce-fire 5..10; pierce-cold 5..10; lifesteal 2..4; manasteal 2..4; move1 10 |
| 851–907 | 1.9 % × 3 | 1490–1492 | dmg-min 10..15; res-all 5..10; curse-effectiveness 10 |
| 908–925 | 1.8 % | 1493 | dmg-max 10..15 |
| 926–982 | 1.9 % × 3 | 1494–1496 | dmg% 30..40; ignore-ac 1; swing1 10 |
| 983–1000 | 1.8 % | 1497 | allskills 1 |

**Quiver, set, socketed** (phase-1 row 424, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–250 | 25 % | 524 | **brick** (new rare, same base and ilvl, then an engine outcome, §2.3) |
| 251–1000 | 75 % | – | **nothing happens** (no row matches; the shard is given back and the item keeps stat 361) |

**Annihilus + Corrupted Worldstone Shard (cwss)** (phase-1 row 425, corruptor 1..1000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–1000 | 20 % × 5 | 533–539 | becomes Charm Small (magic) + dye 10; allskills 1, dye 10; vit 20..25, enr 10..15, dye 10; addxp 3..5, dye 10; res-all 5..10, dye 10 |

**T1 map (magic/rare), e.g. Kurast** (phase-1 row 341, corruptor 1..3000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–2970 | 9 % × 11 | 347–357 | map-mon-deadlystrike 4..8, map-glob-density 30..40, map-play-addxp 4..6; map-mon-nofreeze-hp% 10..15, map-glob-density 30..40, map-play-addxp 6..8; map-play-swing-cast -15..-5, map-glob-density 30..40, map-glob-monsterrarity 10..15; map-play-damageresist -10..-5, map-glob-density 30..40, map-glob-monsterrarity 10..15; map-play-speed-all 20..30, map-glob-monsterrarity 10..15; map-play-mag-gold% 50, map-glob-monsterrarity 10..15; map-glob-density 60..80; map-glob-arealevel 1, map-glob-density 15..20; map-glob-monsterrarity 30..40; map-play-res-all-max -3..-2, map-glob-density 30..40, map-glob-monsterrarity 10..15; map-mon-dropjewelry 1 |
| 2971–2978 | 0.13 % × 2 | 358–359 | becomes Zhar Map (unique); becomes Fallen Gardens Map (unique) |
| 2979–2981 | 0.10 % | 360 | becomes Outer Void Map (unique) |
| 2982–2985 | 0.13 % | 361 | becomes Hellcaves Map (unique) |
| 2986–2988 | 0.10 % | 362 | becomes Imperial Palace Map (unique) |
| 2989–3000 | 0.13 % × 3 | 363–365 | becomes Ureh City Map (unique); becomes Djinns Domain Map (unique); becomes Na-Krul's Abyss Map (unique) |

**T2 map (magic/rare), e.g. Arcane** (phase-1 row 341, corruptor 1..3000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–2970 | 9 % × 11 | 371–381 | map-mon-deadlystrike 8..12, map-glob-density 40..50, map-play-addxp 6..8; map-mon-nofreeze-hp% 15..20, map-glob-density 40..50, map-play-addxp 8..10; map-play-swing-cast -25..-15, map-glob-density 40..50, map-glob-monsterrarity 15..20; map-play-damageresist -15..-10, map-glob-density 40..50, map-glob-monsterrarity 15..20; map-play-speed-all 30..40, map-glob-monsterrarity 15..20; map-play-mag-gold% 75, map-glob-monsterrarity 15..20; map-glob-density 80..100; map-glob-arealevel 1, map-glob-density 20..25; map-glob-monsterrarity 40..50; map-play-res-all-max -4..-3, map-glob-density 40..50, map-glob-monsterrarity 15..20; map-mon-dropjewelry 1 |
| 2971–2978 | 0.13 % × 2 | 382–383 | becomes Zhar Map (unique); becomes Fallen Gardens Map (unique) |
| 2979–2981 | 0.10 % | 384 | becomes Outer Void Map (unique) |
| 2982–2985 | 0.13 % | 385 | becomes Hellcaves Map (unique) |
| 2986–2988 | 0.10 % | 386 | becomes Imperial Palace Map (unique) |
| 2989–3000 | 0.13 % × 3 | 387–389 | becomes Ureh City Map (unique); becomes Djinns Domain Map (unique); becomes Na-Krul's Abyss Map (unique) |

**T3 map (magic/rare), e.g. River of Blood** (phase-1 row 341, corruptor 1..3000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–2970 | 9 % × 11 | 396–406 | map-mon-deadlystrike 12..16, map-glob-density 50..60, map-play-addxp 8..10; map-mon-nofreeze-hp% 20..25, map-glob-density 50..60, map-play-addxp 10..12; map-play-swing-cast -35..-25, map-glob-density 50..60, map-glob-monsterrarity 20..25; map-play-damageresist -20..-15, map-glob-density 50..60, map-glob-monsterrarity 20..25; map-play-speed-all 40..50, map-glob-monsterrarity 20..25; map-play-mag-gold% 100, map-glob-monsterrarity 20..25; map-glob-density 100..120; map-glob-arealevel 1, map-glob-density 25..30; map-glob-monsterrarity 50..60; map-play-res-all-max -5..-4, map-glob-density 50..60, map-glob-monsterrarity 20..25; map-mon-dropjewelry 1 |
| 2971–2978 | 0.13 % × 2 | 407–408 | becomes Zhar Map (unique); becomes Fallen Gardens Map (unique) |
| 2979–2981 | 0.10 % | 409 | becomes Outer Void Map (unique) |
| 2982–2985 | 0.13 % | 410 | becomes Hellcaves Map (unique) |
| 2986–2988 | 0.10 % | 411 | becomes Imperial Palace Map (unique) |
| 2989–3000 | 0.13 % × 3 | 412–414 | becomes Ureh City Map (unique); becomes Djinns Domain Map (unique); becomes Na-Krul's Abyss Map (unique) |

**T4 map (dungeon, e.g. Torment)** (phase-1 row 341, corruptor 1..3000)

| corruptor | p each | rows | outcomes |
|---|---|---|---|
| 1–1000 | 3.3 % × 10 | 2077–2086 | map-mon-deadlystrike 15..20, map-glob-density 50..60, map-play-addxp 8..10; map-mon-nofreeze-hp% 25..35, map-glob-density 50..60, map-play-addxp 10..12; map-play-swing-cast -35..-25, map-glob-density 50..60, map-glob-monsterrarity 20..25; map-play-damageresist -20..-15, map-glob-density 50..60, map-glob-monsterrarity 20..25; map-play-speed-all 40..50, map-glob-monsterrarity 20..25; map-play-mag-gold% 100, map-glob-monsterrarity 20..25; map-glob-density 100..120; map-glob-arealevel 1, map-glob-density 25..30; map-glob-monsterrarity 50..60; map-play-res-all-max -5..-4, map-glob-density 50..60, map-glob-monsterrarity 20..25 |
| 1001–3000 | 66.7 % | – | **nothing happens** (no row matches; the shard is given back and the item keeps stat 361) |

**Djinn/Na-Krul (maps):**
- The MPQ adds rows 364/365, 388/389 and 413/414, which turn a T1/T2/T3 map into **Djinn's Domain (t57)** or **Na-Krul's Abyss (t58)** with 4/3000 = 0.133 % each.
- All eight unique-map outcomes together are 30/3000 = **1 %**: Zhar, Fallen Gardens, Hellcaves ("warlord of blood"), Ureh, Djinn and Na-Krul are 0.133 % each; Outer Void and Imperial Palace are 0.1 % each.
- The new unique map is generated at the old map's level (`t5x,uni ilvl=100`).

### 2.3 What a brick does: PD's corruption engine 0x102BB5F0 (**VERIFIED**)
A "destroyed (rare)" row is `usetype,rar ilvl=100`. By §1.6, that is **a new rare item of the same base, at the old item's level**. It is ethereal if the old one was (§1.3 step 9), and it has no sockets. Then PD runs its engine on it:
```
engine(item, flag):                        // ecx = item, edx = flag
  if stat360(item): if flag == 0 or stat206 (desecrated): return 0     // flag 0 needs an uncorrupted item
  elif flag and stat206: return 0
  roll = 0; base = 0
  for each CubeMain record in order:
     skip unless enabled, op is 16 or 18, param == 361, value != 0, and input 2 is empty or "wss "
     skip unless input 1 matches (PD matcher 0x102BB3F0: type/item code, quality, nos/sock, eth/noe;
          bas/exc/eli and named uniques are ignored)
     if output A is "usetype,rar" with fewer than 3 mods:               // a brick row
          base = value; roll = value + rand() % (1000 - value)          // rand() = CRT rand, 0..32767
          if flag: base = 851; roll = 851 + rand() % 149                 // flag 1 draws a second number
          continue                                                      // (roll >= value, never applied)
     if base == 0 or roll >= value: continue
     if flag and mod1.prop == stat360(item):                            // dead: compares a property id to
          roll = base + rand() % (1000 - base); i = 351; continue       //       an outcome code
     apply the row's output-A mods; with flag 1 rename corrupt -> desecrate and corruptnum -> desecratenum
     return 1
  return 0
```
**Native check (VERIFIED).** `harness/cube/cube.c` loads PD.img and the real MPQ `cubemain.bin` (2,341 records), fills the lazy-import slots with stubs, and calls 0x102BB5F0 on 40,000 random items. The items cover 46 base types, qualities 1–8, eth, 0–6 sockets, flag 0/1, and random stats 360/206/361.
- The C model matched **40,000/40,000**: return value, number of `rand()` calls, and every applied property (id, param, min, max).
- Mutations are caught:

| mutation | match |
|---|---|
| roll range `1001−V` | 32,415/40,000 |
| `≤` instead of `<` | 39,247/40,000 |
| no desecrate rename | 34,085/40,000 |

- `cube.js` `engine()` reproduces 20,000 of those native runs exactly (`node adv/re/test_cube.js`).

So a bricked item is **not lost**. It is re-created as a rare and then gets an outcome drawn from the rows *above* the brick threshold V, because the roll starts at V. Sockets are therefore likely:
- a 1-hand weapon is 16.1 % "2 sockets";
- a bow is 10.1 % "3 sockets" and 6.0 % "6 sockets";
- with 32768 = 43·750 + 518, the first 518 roll values are 1/43 more likely than the rest, which is where 16.113 % instead of 16 % comes from.

For white items the brick row is 429 (V = 500), but the new rare then meets row 452 (V = 250) first. So white and rare weapons get the same brick table.

**Bricked one-hand weapon** (brick row 452; engine roll = 250 + rand() % 750)

| p each (given a brick) | rows | outcomes |
|---|---|---|
| 16.113 % | 501 | sock 2 |
| 9.399 % | 502 | sock 3 |
| 8.057 % | 503 | sock 4 |
| 3.357 % × 10 | 544–553 | dmg% 40..80; att 150..250; heal-hit 3..6; att-demon 200, dmg-demon 100..150; ease -25..-50; mag% 20..30; heal-kill 3..5; mana-kill 3..5; cast2 10; att-undead 200, dmg-undead 100..150 |
| 2.014 % | 555 | pierce-fire 7..10 |
| 1.978 % | 556 | pierce-ltng 7..10 |
| 1.968 % × 8 | 557–564 | pierce-cold 7..10; pierce-pois 7..10; cast2 20; lifesteal 5, dmg% 40..60; dmg-ac -40..-60; deadly 20..30; swing1 30..40; crush 20..30 |
| 1.312 % × 10 | 566–575 | swing1 20, dmg% 80..120; swing1 30, crush 20..30; ignore-ac 1, dmg% 60..80; deadly 25, dmg% 50..70; att 250, dmg% 80..120; allskills 1; extra-fire 5, cast2 10; extra-cold 5, cast2 10; extra-ltng 5, cast2 10; extra-pois 5, cast2 10 |

**Bricked bow/2h** (brick row 452; engine roll = 250 + rand() % 750)

| p each (given a brick) | rows | outcomes |
|---|---|---|
| 10.071 % | 453 | sock 3 |
| 9.399 % | 454 | sock 4 |
| 8.057 % | 455 | sock 5 |
| 6.042 % | 456 | sock 6 |
| 3.357 % × 10 | 544–553 | dmg% 40..80; att 150..250; heal-hit 3..6; att-demon 200, dmg-demon 100..150; ease -25..-50; mag% 20..30; heal-kill 3..5; mana-kill 3..5; cast2 10; att-undead 200, dmg-undead 100..150 |
| 2.014 % | 555 | pierce-fire 7..10 |
| 1.978 % | 556 | pierce-ltng 7..10 |
| 1.968 % × 8 | 557–564 | pierce-cold 7..10; pierce-pois 7..10; cast2 20; lifesteal 5, dmg% 40..60; dmg-ac -40..-60; deadly 20..30; swing1 30..40; crush 20..30 |
| 1.312 % × 10 | 566–575 | swing1 20, dmg% 80..120; swing1 30, crush 20..30; ignore-ac 1, dmg% 60..80; deadly 25, dmg% 50..70; att 250, dmg% 80..120; allskills 1; extra-fire 5, cast2 10; extra-cold 5, cast2 10; extra-ltng 5, cast2 10; extra-pois 5, cast2 10 |

**Bricked body armour** (brick row 508; engine roll = 250 + rand() % 750)

| p each (given a brick) | rows | outcomes |
|---|---|---|
| 16.113 % | 512 | sock 1 |
| 9.399 % | 513 | sock 2 |
| 8.057 % | 514 | sock 3 |
| 4.028 % × 8 | 578–585 | ac% 50..80; mag% 20..30; balance1 20..30; res-fire 30..35; res-cold 30..35; res-ltng 30..35; res-pois 30..35; mana 30..40 |
| 2.686 % | 587 | thorns/lvl 32..48 |
| 2.649 % | 588 | cast1 10 |
| 2.625 % × 6 | 589–594 | hp% 4..6; move1 20; nofreeze 1; red-dmg 6..10; red-mag 6..10; indestruct 1, ac% 50..80 |
| 1.575 % | 596 | curse-effectiveness 10 |
| 1.706 % | 597 | allskills 1 |
| 1.575 % | 598 | res-all 20..25 |
| 1.706 % | 599 | red-dmg% 6..8 |
| 1.575 % | 600 | res-fire-max 4..5, res-fire 15 |
| 1.706 % | 601 | res-cold-max 4..5, res-cold 15 |
| 1.575 % | 602 | res-ltng-max 4..5, res-ltng 15 |
| 1.706 % | 603 | res-pois-max 4..5, res-pois 15 |

**Bricked ring** (brick row 529; engine roll = 250 + rand() % 750)

| p each (given a brick) | rows | outcomes |
|---|---|---|
| 6.042 % × 8 | 774–781 | str 7..10; dex 7..10; vit 7..10; enr 7..10; res-fire 10..15; res-cold 10..15; res-ltng 10..15; res-pois 10..15 |
| 4.028 % × 5 | 783–787 | hp 30..40; mag% 15..20; gold% 40..80; red-dmg 4..6; red-mag 4..6 |
| 3.961 % | 788 | heal-kill 2..3 |
| 3.937 % × 2 | 789–790 | mana-kill 2..3; att 100..150 |
| 2.493 % × 3 | 792–794 | move1 10; red-dmg% 3; curse-res 10 |
| 2.362 % | 795 | lifesteal 3..4 |
| 2.493 % × 3 | 796–798 | manasteal 3..4; cast1 10; res-all 4..6 |
| 2.362 % | 799 | all-stats 4..6 |

### 2.4 Where the result is stored (READ + DATA)
- **Outcome rows** set stat 360 = outcome code, for example 32 = sockets, 2 = +ED, 78–88 = map outcomes. They set stat 361 = 1001 (items, T4 maps) or 3001 (T1–T3 maps), plus the listed stats.
- **After phase 1 only**, the item carries 361 = X and 360 = 0. That happens when no phase-2 row matched (T4 maps with X > 1000, unique maps, socketed set quivers).
  - Every `op 18 361 = 0` row then refuses it: every map orb and every re-roll. So does a second corruption.
  - The stat is not displayed (descfunc 0).
- **Saving:** stat 361 is compiled with **Save Bits = 1** (`itemstatcost.bin` +0x19 = 1; the txt has "Save Bits S12" = 12). If the saver uses these bits, 1001/3001 are saved as 1, and a leftover X is saved as `X & 1`. An even leftover would then read as "never touched" after the next load. This was not run.

### 2.5 Maps (DATA + READ; replaces maps.md §4's zip-based table)
- White maps: BLOCK (row 339).
- T1–T3 magic/rare: see the tables above. Each of the 11 outcomes is 9 %, and unique maps are 1 %. There is no gap: the last unique row is 3000.
- **T4 (dungeons: Torment, Sanctuary of Sin, Kanemith, Mesa):**
  - Phase 1 is row 341 `map + wss → corruptnum 1..3000`, but the T4 rows (2077–2086) stop at 1000.
  - So each T4 outcome is **1/30 = 3.33 %**, and **2/3 of the attempts do nothing**: the shard is given back and the map keeps 361 = X, which locks it (§2.4).
  - The zip copy used 1..1000 for every map, which gave 10 % each. The MPQ re-scaled T1–T3 to 3000 but not T4.
- **Unique maps (t5me):** row 341 matches, but no phase-2 row exists. Nothing happens except that 361 is set.
- "Random rare / down tier / up tier" rows 343–345, 367–369 and 392–394 are **disabled**.

### 2.6 Other callers of the engine (READ)
| caller | flag | when | effect |
|---|---|---|---|
| PD 0x102BDE3E (cube) | 0 | after a ladder-3 (brick) row | §2.3 |
| PD 0x102C8D30 (`map_glob_dropcorrupted`, map key 6) | 0 | `rand() % 100 < value` on a drop | the drop is **pre-corrupted with a normal outcome**. It never bricks: the roll starts at the brick threshold. |
| PD 0x102C89C1 (hook 0x102EE580 on D2Game 0x6FD097D7) | 0 | a quest-reward rare ring (`rin `, ilvl 21/35/75 by difficulty, quest-state checks on quest 0x13) | the ring always comes corrupted with a ring outcome |
| PD 0x102C2DC0 | 0 | an item created at level 99 by PD 0x102EEF00 (source not identified) | pre-corrupted |
| PD 0x102C93AC / 0x102C93B8 | 0 then 1 | an event reward (PD 0x102C92C0, runs when `game+0x262A == 8`) | with 50 % (60 % if `uber_difficulty` > 0) an existing reward item is **desecrated** (flag 1). Otherwise a new amulet is created, corrupted (flag 0) and then desecrated (flag 1). |

**Desecration (flag 1):**
- The roll is `851 + rand() % 149`, so only the top-tier rows (values 869–1000) can be applied.
- The outcome's `corrupt`/`corruptnum` become `desecrate`/`desecratenum` (stats 206/207).
- A corrupted item *can* be desecrated. An already-desecrated item cannot.

---------------------------------------------------------------------------------------------------------------------

## 3. PD2 currency and special items (DATA; the refusal rules are READ, §1.5)
Rows are MPQ line numbers.

**Common rules for the map orbs:**
- Every map orb requires the map to be uncorrupted: `op 18 361 = 0`.
- `usetype` outputs keep the map's base and level. `t?me` type outputs pick a random base of that tier with base level ≤ L (§1.6).
- The number of affixes on a re-rolled rare map follows PD 0x102D57C0 (maps.md §2.2) at the map's own level.

| item (code) | recipe (rows) | result |
|---|---|---|
| Upgrade Magic (`upma`) | magic T1–T3 map + `upma` + 1 rune (from a stack, `runs`) + 1 perfect gem (3, 9, 15) | new **rare** map, same base and level |
| Upgrade Magic Ready (`urma`) | magic T1–T3 map + `urma` (2101, 2105, 2109) | same. `urma` ×N is made from N × (`upma` + rune + perfect gem) (2113–2126). |
| Reroll Rare (`rera`) / Ready (`rrra`) | rare map + `rera` + rune + perfect gem (4, 10, 16); or + `rrra` (2102, 2106, 2110) | new rare map, same base and level |
| Imbue Magic (`imma`) / Ready (`irma`) | white map + `imma` + jewel (5, 11, 17); or + `irma` | **magic** map |
| Reroll magic | magic map + `imma` or `irma` (2297–2302) | new magic map |
| Imbue Rare (`imra`) / Ready (`irra`) | white map + `imra` + jewel + rune (6, 12, 18); or + `irra` | rare map |
| Scour (`scou`) | magic or rare map (7, 8, …) | white map |
| Upgrade Map (`upmp`) | 3 × T1 (or T2) maps of any quality + `upmp` (21–34) | a random **white** T2 (T3) base, level L = 100 % of the first map's level |
| 3 → 1 | 3 maps of one tier (2261–2275) | a random base of the same tier. T2/T3: 3 white → white, 3 magic → magic, 3 rare → rare. **T1: rows 2261/2263 take any quality and come first, so 3 magic or 3 rare T1 maps also give a *white* T1.** Rows 2270/2271 are never reached. |
| Dungeon Scarab (`scrb`) | any-quality T3 map + `scrb` (2169–2171) | a **rare T4** with fixed extras: monster FRW +20–30, AR%/pierce 80–100, splash, +10–20 % life and cannot be frozen, +40–50 % XP. 3 rare T4 give a new rare T4 with the same extras (2260). |
| Fortify (`fort`) | non-white T1–T3 map (2056–2058): skirmish 50 and player FRW +30. Unique map (2280): skirmish 50 only. | refused if already fortified. White maps are blocked (2053–2055). |
| Standard of Heroes (`std`) | any non-white map (340, needs stat 458 = 0) | `heroed` + 20 MF, 20 GF, 20 density, 10 XP |
| Force Mapevent Shard (`iwss`) | non-white T1–T3 map (2090–2092) | `map_force_event` 63. Refused if already set. |
| Invader ears (`ivez` / `ivea` / `iveb` / `ived` / `iven` / `ives` / `ivep`) | magic/rare map + ear (2324 … 2342, needs stat 276 = 0) | rarity +25, **drop bonus 75**, `treacherous` 1–7, plus a class-themed hard mod (e.g. Necromancer: player regen −100, monster 50 % poison). White and unique maps are blocked. |
| Worldstone Shard (`wss`) | §2 | corruption |
| Corrupted Worldstone Shard (`cwss`) | Annihilus (§2.2) | 20 % each: +1 skills, 20–25 vit + 10–15 energy, 3–5 % XP, 5–10 all res, or "replaced by a new **magic** small charm" (row 533). All add `dye 10` and "Corrupted". |
| Puzzlebox (`lbox`) | unsocketed item + `lbox` (328–335, 2303) | sockets: 2-handers/bows/crossbows **2–4**; 1-hand weapons, throwing weapons, helms, body armour, shields and quivers **1–2**. Any quality, including set and unique. Capped by the base. |
| Puzzle piece (`lpp`) | same bases (2043–2050) | same socket ranges, but **refused for set and unique items** |
| Larzuk's Malus (`lmal`) | unsocketed weapon/helm/shield/body/quiver (2071–2075) | +1 socket (sets it to 1), any quality |
| Lilith's Mirror (`llmr`) | magic/rare/crafted weapon, armour, ring, amulet, arrows/bolts of any tier (1601–1614, 2304–2321; jewels disabled) | A: `dye 4` + `mirrored`. B: `useitem dye 21 + mirrored`. **Mirrored items are then refused by the cube** (stat 486). Which unit receives output B was not traced (the stock rule maps B to input 2). |
| Vial of Lightsong (`lsvl`) | non-ethereal weapon or armour (1599–1600) | becomes **ethereal**. Refused for Phase Blades, indestructible items, and items whose own type is bow, crossbow, Amazon bow, missile weapon or throwing potion. |
| Demonic Cube (`imrn`) | unique (staff, spiked/bone shield, quiver, weapon, armour, amulet, ring, jewel, Gheed's, Torch, Annihilus) or set (weapon, armour, amulet, ring) + `imrn` (2281–2296, op 31) | PD 0x102BD5E0: 1 `imrn` is used. The **same unique/set** is created again at the item's level (PD 0x102D5980 / 0x102D5870), so its variable stats re-roll. It keeps the base and upgrade tier, ethereal, sockets and their contents, durability, quantity, base defense and indestructible. Refused for corrupted items. |
| Jewel → fragments | Key + N jewels (1618–1632) | N Jewel Fragments (`jewf`, a stack). A fragment counts as `jewg`, i.e. as a jewel, in crafting and in map imbues. |
| Torch → fragments | Key + N Hellfire Torches (2093–2099) | N Torch Fragments (`cm2f`) |
| Rathma | Splinter `rtmv` + Hellfire Torch (or `cm2f`) + `rtmo` → Rathma map `rtma` (1616, 2100). 2 Bone Fragments `rtmf` + Torch → `rtmo` (1617). | |
| Lucion / Uber Ancients / Uber Diablo | `lucb+lucc+lucd → luca` (2276); `ubaa+ubab+ubac → uba` (1634); `dcbl+dcho+dcso → dcma` (336) | |
| Uber difficulty | `dcma` / `rtma` / `luca` alone (2186–2191, 2277–2279) | `uber_difficulty` 0 → 1 → 2 → 0 |
| Pandemonium keys | `pk1+pk2+pk3` (317, op 28) | a portal to one unopened mini-uber level (133/134/135, chosen by `rand()%3`) |
| Organs | `dhn+bey+mbr` (320) | output B: Uber Tristram map `ubtm`. Output A is `useitem`, so whether an organ is kept was not traced. |
| Token of Absolution | the four essences `tes+ceh+bet+fed` (321) | `toa` |
| Cow level | Wirt's Leg + Tome of TP, or + unlimited TP book `rtp`, which is given back (37, 1633; op 28 param 2) | Cow portal |

**Not present in the live MPQ:**
- no sunder charms (UniqueItems has only Gheed's, Annihilus, Torch and four disabled trophies);
- no cube recipe for skill charms (magic grand charms). They re-roll with the generic "3 perfect gems + magic item" (row 2066).
- The debug "Generate Uniques" rows 1586–1598 are disabled.

---------------------------------------------------------------------------------------------------------------------

## 4. Crafting, re-rolls, upgrades and sockets

### 4.1 Crafted items (DATA + READ + **VERIFIED**)
**The seven families** (476 enabled rows):
- the four stock families: Hit Power, Blood, Caster, Safety;
- three PD2 families: **Vampiric** (perfect skull, ingredient `crfv`), **Bountiful** (perfect topaz, `crfu`), **Brilliant** (perfect diamond, `crfp`).

**Recipe:** a magic *or rare* base + either (jewel or jewel fragment) + a rune + the family's perfect gem, **or** the family's single craft ingredient (`crfh`/`crfb`/`crfc`/`crfs`/`crfv`/`crfu`/`crfp`).
- Unlike stock D2, **any base of the slot type works**. The input is the ItemTypes slot (`helm`, `boot`, `glov`, `belt`, `shld`, `misl`, `tors`, `amul`, `ring`, `weap`), not a fixed base.
- Ethereal and non-ethereal bases have separate rows. The ethereal rows add the `ethereal` property, so **an ethereal base gives an ethereal crafted item**.
- **Helm rows have version 200, and PD 0x102BB9C0 refuses circlets** (`ci0`–`ci3`) for them.

Runes by family and slot (jewel route; the gem is the family's perfect gem):

| family | ingredient | gem | helm | boots | gloves | belt | shield | quiver | body | amulet | ring | weapon |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Hit Power | `crfh` | `gg4s` | Ith | Ral | Ort | Tal | Eth | Eth | Nef | Thul | Amn | Tir |
| Blood | `crfb` | `gg4r` | Ral | Eth | Nef | Tal | Ith | Ith | Thul | Amn | Sol | Ort |
| Caster | `crfc` | `gg4a` | Nef | Thul | Ort | Ith | Eth | Eth | Tal | Ral | Amn | Tir |
| Safety | `crfs` | `gg4e` | Ith | Ort | Ral | Tal | Nef | Nef | Eth | Thul | Amn | Sol |
| Vampiric | `crfv` | `gg4z` | Lum | El | Io | Eld | Sol | Sol | Tir | Dol | Hel | Shael |
| Bountiful | `crfu` | `gg4t` | Tir | Io | Hel | El | Dol | Dol | Eld | Ko | Lum | Shael |
| Brilliant | `crfp` | `gg4d` | Eld | Ko | Shael | Io | Ort | Ort | El | Tal | Hel | Sol |

Fixed mods added to the new crafted item (always applied: the chance column is blank):

| family | helm | boots | gloves | belt | shield | quiver | body | amulet | ring | weapon |
|---|---|---|---|---|---|---|---|---|---|---|
| Hit Power | gethit-skill[44] 5–4, balance1 10–20, att 100–200 | gethit-skill[44] 5–4, balance1 10–20, ac-hth 60–120 | gethit-skill[44] 5–4, reduce-ac 10–20, knock 1 | gethit-skill[44] 5–4, balance1 10–20, block2 10 | gethit-skill[44] 5–4, block2 20–30, block 10–20 | gethit-skill[44] 8–4, reduce-ac 10–20, knock 1 | gethit-skill[44] 8–4, ac% 75–100, balance1 30–40 | gethit-skill[44] 5–4, att 150–250, balance1 10–20 | gethit-skill[44] 5–4, dmg-max 2–4, dex 5–10 | gethit-skill[44] 5–4, reduce-ac 10–20, dmg% 50–80 |
| Blood | lifesteal 2–4, hp 10–20, crush 10–15 | lifesteal 2–4, hp 10–20, openwounds 5–10 | lifesteal 2–4, hp 10–20, crush 5–10 | lifesteal 3–6, hp 10–20, openwounds 10–20 | lifesteal 3–6, hp% 5–8, block 10–20 | lifesteal 3–6, hp 10–20, crush 10–20 | ac% 30–50, hp% 5–8, heal-kill 4–6 | heal-kill 3–6, hp 10–20, move1 10 | lifesteal 2–3, hp 10–20, str 5–10 | lifesteal 3–6, hp 10–20, dmg% 50–80 |
| Caster | cast2 10, enr 10–20, regen-mana 5–10 | regen-mana 4–10, mana 10–20, mana% 10–15 | cast2 5–10, mana 10–20, mana-kill 1–3 | regen-mana 4–10, mana 10–20, cast1 10 | cast1 10, mana% 20–30, block 10–20 | cast1 10, mana 10–20, mana-kill 2–4 | cast1 10–15, enr 15–25, mana-kill 3–6 | regen-mana 4–10, mana 10–20, cast1 5–10 | mana-kill 1–2, mana 10–20, enr 5–10 | allskills 1, mana 10–20, cast1 10–20 |
| Safety | red-dmg% 4–6, res-ltng 10–20, ac% 30–60 | red-dmg 2–5, red-mag 2–5, res-fire 10–20, ac% 20–60 | red-dmg 2–5, red-mag 2–5, res-cold 10–20, ac% 20–60 | red-dmg 2–5, red-mag 2–5, res-pois 10–20, ac% 20–60 | red-dmg% 4–6, ac% 50–75, ac-hth 100–150 | move2 10, red-mag 4–7, red-dmg 4–7 | red-dmg% 4–6, ac-hth 100–150, ac% 50–75 | red-dmg 4–7, red-mag 4–7, block 10 | red-dmg 2–5, red-mag 2–5, vit 5–10 | red-dmg 2–5, red-mag 2–5, dmg% 50–80, knock 1 |
| Vampiric | manasteal 2–4, deadly 10–15, regen 10–20 | manasteal 2–4, res-pois-len 15–25, regen 10–20 | manasteal 2–4, deadly 5–10, regen 10–20 | manasteal 3–6, res-pois-len 15–25, regen 10–20 | manasteal 3–6, res-pois-len 30–50, block 10–20 | manasteal 3–6, res-pois-len 30–50, regen 15–25 | manasteal 3–6, res-pois-len 30–50, ac% 30–50 | manasteal 3–6, curse-res 15–25, regen 20–30 | manasteal 2–3, res-pois-len 15–25, dmg-min 2–4 | manasteal 3–6, regen 5–10, dmg-undead 100–140, dmg-demon 100–140 |
| Bountiful | mag% 20–30, light 1–3, ac% 25–50 | mag% 15–25, light 1–3, ac% 25–50 | mag% 15–25, light 1–3, ac% 25–50 | mag% 20–30, light 1–3, ac% 25–50 | mag% 20–30, gold% 30–50, block 10–20 | mag% 15–25, gold% 20–30, vit 5–10 | mag% 20–30, gold% 30–50, ac% 30–50 | mag% 20–30, light 1–3, vit 10–20 | mag% 10–20, light 1–3, vit 5–10 | mag% 20–30, light 1–3, allskills 1 |
| Brilliant | half-freeze 1, swing2 5–10, ac% 25–50 | half-freeze 1, move2 5–10, ac-miss 100–150 | swing2 5–10, att% 10–20, ac% 25–50 | half-freeze 1, swing2 5–10, ac% 25–50 | half-freeze 1, res-all 5–10, block 10–20 | ac-miss 100–150, balance2 10, res-all 5–10 | block 10–20, res-all 5–10, ac% 25–50 | all-stats 4–8, res-all 5–10, ac-miss 100–150 | half-freeze 1, res-all 3–5, ac-miss 50–100 | dmg% 20–40, swing2 20, att 50–100 |

`gethit-skill[44] 5–4` = 5 % chance to cast level 4 Frost Nova (skill 44) when struck; 8–4 on body armour and quivers is 8 %.

**What item-generation call is made:**
- Output A is `usetype,crf plvl=50 ilvl=50` → stock D2Game 0x6FC92220 creates **a new item of the same base** (class id), quality 8 (crafted), at
  ```
  L = trunc(50 * clvl / 100) + trunc(50 * baseItemLevel / 100)          (clamped 1..99)
  ```
- D2Game then runs the stock crafted roller **0x6FC34BF0** (the only PD change is the suffix duplicate/group test at 0x6FC34E36, which PD 0x102C5B30 re-implements with the same logic):
  ```
  tier = L<=30 ? 1 : L<=50 ? 2 : L<=70 ? 3 : 4
  n    = max(tier, seed % 5)                          // item seed LCG: lo*0x6AC690C5 + hi
  repeat n times:
      side = seed & 1 (0 = prefix, 1 = suffix); a side with 3 affixes forces the other
      pick with the stock picker 0x6FC344D0; on a duplicate id or group retry, up to 252 picks,
      after which that round is lost
  ```
- The affix pool, alvl and picking weights are the stock picker's (see `items.md`). Then the 3–4 fixed cube mods are added.

| L | 1 affix | 2 | 3 | 4 |
|---|---|---|---|---|
| ≤ 30 | 40 % | 20 % | 20 % | 20 % |
| 31–50 | – | 60 % | 20 % | 20 % |
| 51–70 | – | – | 80 % | 20 % |
| ≥ 71 | – | – | – | 100 % |

**Native check (VERIFIED).** `harness/cube/craft.c` loads D2Game.img, applies PD's patch bytes at 0x6FC34E36, and stubs the picker with small id pools so that duplicates and retries happen.
- 40,000/40,000 cases match on picks, prefix/suffix order, slot contents and the final seed state.
- Mutations are caught:

| mutation | match |
|---|---|
| thresholds `< 30/50/70` | 39,291/40,000 |
| `n = tier` | 28,037/40,000 |

### 4.2 Re-rolls (DATA; output level READ)
| recipe | row | output | level L |
|---|---|---|---|
| 3 perfect gems (any colour) + magic item (not corrupted; maps and PvP charms blocked) | 2066 | new **magic** item, same base | 100 % of the item's level (this also covers skill grand charms) |
| 3 perfect skulls + rare item (not corrupted) | 2068 | new rare, same base | `trunc(0.4·clvl) + trunc(0.4·ilvl)` |
| 1 perfect skull + rare + Stone of Jordan | 2069 | new rare, same base | `trunc(0.66·clvl) + trunc(0.66·ilvl)` |
| 3 perfect skulls + unsocketed rare + SoJ | 2070 | the same item with 1 socket (`sock=1`) | – |
| 3 magic rings → magic amulet; 3 magic amulets → magic ring | 47, 48 | random base of that type | `trunc(0.75·clvl)` |
| perfect gem of each type + magic amulet → prismatic; ring + perfect gem + potion → garnet/cobalt/coral/jade | 41–45, 2172–2183 | fixed prefix | lvl 50 / 30 (fixed) |
| Demonic Cube | 2281–2296 | same unique/set, stats re-rolled | – (§3) |
| map orbs | §3 | | map level |

New rares follow PD's rare-count rule 0x102D57C0 at level L (maps.md §2.2): 3 + r%4 below 45, 4 + r%3 from 45, 5 + r%2 from 65, 6 from 85, 4 for jewels.

### 4.3 Upgrades normal → exceptional → elite (DATA; mechanism READ)
All are `useitem,mod,exc|eli`: the same item is moved to its `ubercode`/`ultracode`, keeping its mods. PD keeps the armour defense percentile (§1.6).

| item | bas → exc | exc → eli | extra mod on the result |
|---|---|---|---|
| unique / set weapon | Ral + Sol + perfect emerald (266–267, 322–323) | Lum + Pul + perfect emerald (271–273, 325–326) | staves: FCR 20 (`cast2`) |
| unique / set armour | Tal + Shael + perfect diamond (268, 324) | Ko + Lem + perfect diamond (274, 327) | arrows: max damage +8; bolts: `pierce-phys` 2 |
| rare / crafted weapon | Ort + Amn + perfect sapphire (277–278, 291–292) | Fal + Um + perfect sapphire (284–285, 298–299) | staves: FCR 20 |
| rare / crafted armour | Ral + Thul + perfect amethyst (281, 295) | Ko + Pul + perfect amethyst (288, 302) | spiked shields: thorns 14–16 / 64–84; bone shields: magic damage reduced 2; arrows/bolts as above |

Also:
- Low quality → normal: Eld + chipped gem + low weapon, or El + chipped gem + low armour (264, 265). The output is `usetype,nor` with no level fields, so **the new normal item is ilvl 1**.
- Repair and recharge: Ort + weapon, Ral + armour (non-ethereal); Zod + perfect skull + any weapon or armour (305–308).

### 4.4 Sockets and unsocketing (DATA)
| recipe | rows | sockets |
|---|---|---|
| Tal + Thul + perfect topaz + normal/superior body armour | 255–256 | 1–4 |
| Ral + Amn + perfect amethyst + normal/superior weapon | 257–258 | 1–6 |
| Ral + Thul + perfect sapphire + normal/superior helm | 259–260 | 1–3 |
| Tal + Amn + perfect ruby + normal/superior shield; the same + normal quiver | 261–263 | 1–4; quiver 1–2 |
| Puzzlebox, Puzzle piece, Larzuk's Malus | §3 | |
| 3 standard gems + socketed weapon / 3 flawless gems + magic weapon / 3 chipped gems + magic weapon | 50, 51, 315 | new magic socketed weapon (1–2), level 30 / 30 / 25 (fixed `lvl`) |
| **Unsocket:** Hel + Scroll of Town Portal, or + unlimited TP book (given back), + any socketed item | 309, 2051 | `uns`: the socketed items are **destroyed**, the sockets stay |

Socket counts are uniform within the range. Stock property roll; READ, not traced. They are capped by the base's `gemsockets` and by MaxSock for its item level.

---------------------------------------------------------------------------------------------------------------------

## 5. Runes, gems and runewords
**Runes:**
- A Key plus 3 runes gives the next rune, up to Lem → Pul: El…Lem, rows 191–219.
- A Key plus **2** runes gives the next rune from Pul to Zod (220–231).
- The same works for rune stacks (`r01s`…`r32s`, 200–254).
- **No gem is needed**, although the row descriptions still name gems.
- Output A is `useitem` on the Key. Output B is the new rune.

**Gems and skulls:**
- 3 chipped → flawed → standard → flawless, with no Key.
- Flawless → perfect only through stacks: Key + 3N flawless in one stack gives N perfect (N = 1…16; 61–189).

**Runewords:**
- There is no cube recipe that makes a runeword. Runewords are made by socketing, which is not a cube action.
- A runeword (socketed white, superior or low-quality item) **cannot be corrupted**: no phase-1 row matches `sock` with those qualities.
- Unsocketing it (Hel + TP) destroys the runes.
- Upgrades need `mod` rows. There is none for normal-quality bases, so runewords cannot be upgraded.

---------------------------------------------------------------------------------------------------------------------

## 6. Worked examples (real MPQ numbers)
1. **Rare Crusader Bow + Worldstone Shard.**
   - Corruptor X is uniform on 1..1000 (row 419).
   - X ≤ 250 → brick (row 452); 251–325 → 3 sockets; 326–395 → 4; 396–455 → 5; **456–500 → 6 sockets (4.5 %)**; 501–1000 → one of 30 mods.
   - A brick gives a new rare Crusader Bow at the same level, and the engine then gives it 6 sockets with 6.042 %.
   - Total chance to end with a 6-socket bow: 4.5 % + 25 % × 6.042 % = **6.01 %**. The 1.51 % from the brick is a different rare.
   - Example roll: X = 440 → first matching row is 455 (threshold 455) → 5 sockets.
2. **Brick roll by hand.** A bricked one-hand weapon draws `rand() = 12345`:
   - r = 250 + 12345 % 750 = 250 + 345 = **595**;
   - the first following weapon row with a threshold above 595 is row 547 (600): +200 AR vs demons, +100–150 % damage vs demons.
3. **Unique Shako (no sockets) + shard:**
   - 25 % brick (a new *rare* Shako);
   - the socket rows are sock 1 (12 %), sock 2 (7 %) and sock 3 (6 %). Shako's `gemsockets` is 2, so the result is **12 % one socket, 13 % two sockets**;
   - 3 % each for the 8 tier-1 helm mods, 2 % each for tier 2, 1.2–1.3 % each for tier 3 (e.g. +1 all skills 1.3 %).
4. **T4 Torment map + shard:** each of the 10 outcomes is **3.33 %**. With **66.7 %** nothing happens, the shard comes back, and the map can no longer be corrupted or re-rolled with orbs.
5. **T1 Kurast map + shard:** each of the 11 mods is 9 %; Djinn's Domain is 4/3000 = 0.133 %; any unique map is 1 %.
6. **Blood gloves:**
   - clvl 90, magic Heavy Gloves of ilvl 70 + Blood ingredient (`crfb`, row 1724) → L = 45 + 35 = **80** → always **4** random affixes, plus life leech 2–4, +10–20 life and crushing blow 5–10.
   - The same at clvl 60 with ilvl-30 gloves → L = 30 + 15 = 45 → 2 affixes 60 %, 3 affixes 20 %, 4 affixes 20 %.
7. **3 perfect skulls + rare armour** (clvl 90, ilvl 85) → L = 36 + 34 = 70 → a new rare of the same base with 5 or 6 affix attempts (PD 0x102D57C0: 5 + r%2 at 65–84).
8. **Annihilus + Corrupted Worldstone Shard:** 20 % +1 all skills (row 536). 20 % it is replaced by a new *magic* small charm (row 533), which destroys the Annihilus.

---------------------------------------------------------------------------------------------------------------------

## 7. JS model (`adv/re/cube.js`)
```js
const C = require('./cube.js');                    // Node: auto-loads cube-data.json; browser: PD2Cube.setData(json)
const it = C.makeItem({ code: '6l7', quality: 'rar', ilvl: 85, sockets: 0, eth: false });
C.corrupt(it)            // {phase1Row, range, outcomes:[{p,row,kind:'modify'|'replace'|'brick',label,mods,brick?}], nothing, notes}
C.brickReroll(item)      // exact engine distribution (15-bit rand, modulo bias included)
C.engine(item, 0|1, rand)// one engine run (matches the native code 20,000/20,000)
C.cubeLevel({plvl:50, ilvl:50}, 90, 70)  // -> 80
C.craftAffixOdds(80)     // -> {1:0,2:0,3:0,4:1}
C.crafts()               // families -> [{row, base, ingredients, plvl, ilvl, mods, noCirclets}]
C.recipesFor(item)       // every row whose input 1 matches, in game order (BLOCK rows included)
C.recipeOutcome(1724, C.makeItem({code:'xhg', quality:'mag', ilvl:70}), 90)
```
- Item codes are ItemsTxt codes. Quality is `low|nor|hiq|mag|set|rar|uni|crf`. `unique` is the UniqueItems name, needed only for `Annihilus`.
- `corrupt()` refuses corrupted or mirrored items, exactly as PD does.

---------------------------------------------------------------------------------------------------------------------

## 8. Left unverified

- The phase-1 corruptor roll and the `sock min..max` rolls are stock property rolls on the item seed, assumed uniform. They were not run.
- The stock input test 0x6FC90240, the output code 0x6FC92220, the level formula and the refusal rules in PD 0x102BCC00 are READ only. Only the engine (0x102BB5F0) and the crafted roller (0x6FC34BF0) were run.
- Stack-quantity use by PD 0x102BBFF0, and whether a Key or other `useitem` catalyst is used up.
- The contents of PD's special code set (0x102CE7C0).
- The source of the item pre-corrupted at 0x102C2DC0, and which quest gives the corrupted ring.
- The full details of the Demonic Cube re-creation (PD 0x102D5980/0x102D5870): which stats besides those listed are carried over.
- op 28 param 2 (cow portal) conditions; the getters behind item ops 19–26.
- Realm-side behaviour: the server may run different tables. Every probability here is for the MPQ copy.
