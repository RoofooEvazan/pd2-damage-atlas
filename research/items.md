# Item property generation (PD2 = 1.13c D2Game/D2Common + ProjectDiablo.dll)

This picks up where `drops.md` stops: base, quality, ethereal and sockets are known. It covers how the game turns that into
affixes and stat values. Engine: `items.js` (browser `window.PD2Items`, Node `module.exports`). Data: `items-data.json`, built by
`extract_items.py`. Worked examples: `test_items.js`. Harness: `harness/itemaffix.c` with the drivers `itemaffix_driver.js`,
`itemfull_driver.js`, `itemstaff_driver.js`, `itemauto_driver.js` and `itemaffix_probe.js` (shared tables in `itemaffix_lib.js`).

Status key:
- **VERIFIED** means the real code ran natively and matched `items.js` on every case.
- **READ** means the claim comes from the disassembly only.
- **DATA** means the claim comes from the tables.

Crafted, tempered and cube outputs belong to the cube agent. Only the shared picker is described here.

## Data source (important)

1.13c loads the compiled `.bin` tables. The model uses `excel_mpq/` (the live `bin/pd2data.mpq`). I checked it field by field
against the `.bin` files in the same MPQ: `magicprefix/magicsuffix/automagic.bin` (every field of every record),
`weapons/armor/misc.bin` (type, qlvl, magic lvl, auto prefix, sockets, durability, defense, damage, inventory size),
`itemtypes.bin`, `properties.bin` and `qualityitems.bin`. **0 differences** remain after two compiler rules are applied:
- **Rows named `Expansion` are dropped by the txt compiler.** They exist in ItemTypes, Properties, MagicPrefix, MagicSuffix, Weapons,
  Armor, Misc, UniqueItems and SetItems. So every index after such a row is one lower than its txt row number. For example,
  `jewl` = ItemTypes 58 and `map` = 105, which are exactly the constants PD's code tests. Tools that index the txt row
  directly are off by one after the separator.
- **Property codes are linked case-insensitively.** Eight MagicSuffix rows (of Bleeding … of Hematic) spell the code
  `Deep-Wounds`. It still resolves to Properties 408 `deep-wounds`, which the `.bin` confirms.

## Harness results (all on the real PD2 tables in the 1.13c record layouts)

| driver | real code run | cases | result |
|---|---|---|---|
| `itemaffix_driver.js` mode 0 | magic item D2Game 0x6FC303C0 with PD's force wrapper 0x102D7170 patched in at 0x6FC3040C/0x6FC3047C, the real picker 0x6FC344D0, D2Common #10461 0x6FD95E50 (itype/etype/socket test on an Equiv bit matrix), group test 0x6FC331E0, prefix/suffix slots | 10,000 (random base, ilvl 1–99, seeds) | **10,000/10,000** (return, 3+3 slots, affix ids, apply order, final item seed) |
| same, mode 1 | rare loop 0x6FC35240 with PD count 0x102D57C0 patched at 0x6FC352BF (static patch bytes), PD rand 0x102C5D10 | 10,000 | **10,000/10,000** (+ unit seed word) |
| same, mode 2 | D2Common #10996 0x6FD960B0 → 0x6FD95AA0 → 0x6FD95990 → the property function table 0x6FDEB0C8 (funcs 1–21), random range 0x6FD51180 | 10,000 random affix mods | **10,000/10,000** (every stat/param/value written + final seed) |
| `itemfull_driver.js` | magic and rare generation with the mod application **not** stubbed | 20,000 | **20,000/20,000** (affixes, all stat writes in order, final seed) |
| `itemstaff_driver.js` | PD class-skill roll 0x102C5DF0 (replaces stock 0x6FC33320) | 20,000 | **20,000/20,000** |
| `itemauto_driver.js` | automagic pick (picker with a2=0, a3=1, a7=Items `auto prefix`) | 20,000 | **20,000/20,000** |
| `itemaffix_probe.js` | the picker with seeds chosen so that its weighted draw is exactly r = 0..W+40 | all r for a circlet (W 3411) and an amulet (W 159) | reconstructs the weight table. **r = W returns the last candidate** (see §2.4) |

Mutation checks. The same run fails when the model is changed on purpose:

| mutation | mismatches |
|---|---|
| no PD forcing | 3,449/30,000 |
| "r == W picks nothing" | 542 |
| alvl = ilvl | 3,686 |
| no magic-lvl weighting | 3,405 |
| roll min..max−1 | 6,208 value cases |
| staffmods without the reqlevel filter | 8,495/20,000 |
| staffmods skill level +1 | 14,475/20,000 |

Stubs:
- modes 0/1: the mod application (checked separately in mode 2 and together in `itemfull`), rare names 0x6FC34210, and the post
  step 0x6FC33DE0.
- stat-list getter 0x6FD97020.
- stat setters 0x6FD8A280/0x6FD8A330/0x6FD8A740 (they record what is written).
- base-code lookup 0x6FD59240 (a scan of the item table).

The ItemTypes Equiv closure matrix is built by the driver. The real loader was not run.

## 0. Order of work in the item-creation core, D2Game 0x6FC30A20 (READ)

1. Zero the affix slots (+0x38 prefixes, +0x3E suffixes of the item data). Adjust quality (drops.md §4).
2. Quality switch (table 0x6FC31038):

   | quality | handler |
   |---|---|
   | 1 low | 0x6FC34AF0 → 0x6FC33E60 |
   | 2 normal | 0x6FC307B0 |
   | 3 superior | 0x6FC340B0 |
   | 4 magic | 0x6FC303C0 |
   | 5 set | 0x6FC341E0 (PD wrapper 0x102D7210) |
   | 6 rare | 0x6FC354F0 → 0x6FC35240. On failure → magic (0x6FC304F0) → further fallbacks 0x6FC301E0 / 0x6FC309B0 |
   | 7 unique | 0x6FC2F370. On failure → rare with ×3 durability |
   | 8 crafted | 0x6FC34BF0 |
   | 9 tempered | 0x6FC34B30 |

   Every non-unique/set handler ends with the class-skill step 0x6FC33DE0 (§8.2).
3. Expansion items: ethereal roll 0x6FC2EBB0 (§7.3).
4. Quality 1–3: sockets (PD 0x102F0660 → 0x102CB370 → stock 0x6FC2EC90, §7.4).
5. Quality ≠ set/unique (byte table 0x6FC31064): automagic 0x6FC30FCB (§8.1).

Two RNG streams are involved:
- the **item seed** (item data +4/+8; #10411) is used by the affix picks, the value rolls, ethereal and the socket chance;
- the **unit seed** (unit +0x20) is used by base defense, durability and quantity.

PD's `pdRand` 0x102C5D10 on an item takes the item seed's low word plus the *unit* seed's high word (unit+0x24). It is
used by the rare count and the class skills.

## 1. Item level and affix level

`ilvl` is the monster level (drops.md). #10086 0x6FD73810 clamps the stored value to ≥ 1. `qlvl` is Items `level`, a byte at
record +0xFD (#10395 0x6FD73050). `magic lvl` is at Items +0x140. The picker computes (D2Game 0x6FC345DB–0x6FC3465F,
**VERIFIED**):

```
i = max(ilvl, qlvl)
if magic_lvl:            alvl = i + magic_lvl
elif i < 99 - qlvl/2:    alvl = i - qlvl/2            (integer division)
else:                    alvl = 2*i - 99
alvl = clamp(alvl, 1, 99)
```
- An affix row is eligible when `level ≤ alvl` and (`maxlevel` = 0 or `maxlevel ≥ alvl`).
- There is no separate 99 cap on ilvl. The clamp is on alvl.
- PD2 does not patch this code. There are no patch records in 0x6FC344D0–0x6FC34896.
- In PD2 the only items with `magic lvl` are wands, staves and orbs (1) and circlets. Circlet 3, Coronet 8, Tiara 13 and
  Diadem 18 (DATA).

Examples (items.js `alvl`):

| base | qlvl | alvl at ilvl 60 / 85 / 90 / 99 |
|---|---|---|
| Amulet, Ring, Jewel, Grand Charm (cm3) | 1 | = ilvl |
| Small Charm (cm1) | 28 | 46 / 71 / **81** / 99 |
| Large Charm (cm2) | 14 | 53 / 78 / 83 / 99 |
| Circlet | 24 | 63 / 88 / 93 / 99 |
| Diadem | 85 | always 99 |

## 2. Magic items

### 2.1 How many affixes (**VERIFIED**)

Stock 0x6FC303C0 makes two picker calls, prefix then suffix. Each picker call first draws `rand & 1` (0x6FC34583). A 0
returns "no affix" unless the call is **forced** (argument 3). The suffix is forced when no prefix was taken. Stock odds:
prefix+suffix 25 %, prefix only 25 %, suffix only 50 % (1.25 affixes on average).

A drop may force an affix index (drop +0x68 / +0x74). A negative index forbids that side.

**PD2 (0x102D7170, wrapping both calls)** sets "force" on *both* calls when:

| item | forced from |
|---|---|
| map (ItemTypes 105) | always |
| charm (type 13 and children) | ilvl ≥ 90 |
| ring / amulet / jewel (10/12/58) | ilvl ≥ 85 |
| everything else | ilvl ≥ 65 |

Above the threshold a magic item always has one prefix and one suffix, unless a side has no candidate at all.
Examples: ring at ilvl 84 → 1.25 affixes on average, at 85 → 2. Small charm at 89 → 1.25, at 90 → 2.

### 2.2 Eligibility (picker loop 0x6FC34674, **VERIFIED**)

A MagicPrefix/MagicSuffix row (record 0x90) is a candidate when:
1. `spawnable` (only checked when argument 2 is set: magic/rare yes, automagic no).
2. `version ≥ 100` → the item must be an expansion item (always true for drops).
3. `level ≤ alvl` and `maxlevel` is 0 or ≥ alvl.
4. `rare` = 1 when the item's quality is rare, crafted or tempered (6/8/9).
5. The item-type test #10461 0x6FD95E50:
   - **Socket affixes:** when the row's first mod writes stat 194 (`item_numsockets`), the item needs Items `hasinv` and
     max sockets > 0. Max sockets = min(Items `gemsockets`, ItemTypes MaxSock1/25/40 for ilvl ≤ 25 / ≤ 40 / > 40)
     (0x6FD74610).
   - **etype:** no etype may match (at most 5, the list stops at the first empty entry).
   - **itype:** at least one itype must match (at most 7).
   - **"Match"** means the item's type1 or type2 *is-a* that type (the Equiv closure, D2Common 0x6FD74430).
6. The automagic call (argument 7 = group) only takes rows of that group.
7. `frequency` > 0.
8. `classspecific` (rec +0x66): allowed when the item's own ItemTypes `Class` is none, or equals it (#10822 0x6FD74280).
   So class-specific rows can roll on classless items whose type is in their itype list. Example: Monk's (+Paladin skills)
   has itype swor/mace/hamm/shld.
9. **group:** the row's group must not already be on the item. 0x6FC331E0 scans the prefix slots and then the suffix slots.
   Prefix and suffix groups share one namespace.

`levelreq`, `classlevelreq`, `class`, `divide`, `multiply` and `add` play no part in the roll. They are the
requirement and price columns.

### 2.3 Weights (**VERIFIED**)

```
weight = frequency                    normal items
weight = frequency * level            items with Items 'magic lvl' (circlets, wands, staves, orbs)   (0x6FC34798)
```
This is stock code. It strongly favours high-level rows on circlets. For example, on a Diadem, Valkyrie's (+2 Amazon,
level 90, freq 2) weighs 180 against Maiden's (+1, level 36, freq 4) at 144.

### 2.4 The draw (**VERIFIED**, including the probe)

```
r = rand(W + 1)                       W = sum of weights (0x6FC347FC, D2 LCG on the item seed)
walk the candidates in table order: r -= weight; take the first with r < 0
if the walk ends (r == W): take the LAST candidate          (fall-through into 0x6FC34853)
```
The last eligible row in table order therefore has weight `w+1`. A pick never fails when there is at least one
candidate.

> **Correction to maps.md §2.3:** a draw of r = W does *not* "pick nothing". The probe (`itemaffix_probe.js`) shows the
> game returning the last candidate at r = W and the first again at r = W+1.

The list stores at most 511 entries, but the weight sum does not stop. No PD2 base comes close: the largest pool is 125
(Circlet prefixes).

## 3. Stat value rolls (D2Common, **VERIFIED**)

Applying an affix (#10996 0x6FD960B0, mode 0 → 0x6FD95AA0 → 0x6FD95990) does this for each of the 3 mods in order:
- It stops at the first mod whose property is −1 (empty).
- It runs Properties `func1..func7` from the table at 0x6FDEB0C8.
- The value returned by func1 is passed as `prev` to func2..7.

The random range is 0x6FD51180 / 0x6FD965B0:
```
roll(min,max) = min                          if min == max (no RNG used)
              = min + rand(max-min+1)        (swapped if max < min), D2 LCG on the item seed: uniform
```

| func | used by | effect (value v) |
|---|---|---|
| 1 | most stats | v = roll; stat[i] = v, param = Properties `val` (the mod `param` is **not** used) |
| 2 | `ac%` | v = roll → stat 16; **also sets base defense to maxac+1** (0x6FD96E50, seen natively: Shako 141 → 142) |
| 3, 8 | res-all, cast/balance, mapped pairs | v = prev, or a roll when prev = 0 (so res-all writes the same v to 4 stats) |
| 5 / 6 | `dmg-min` / `dmg-max` | v = prev or roll, written to the 1-hand (21/22), 2-hand (23/24) and throw (159/160) damage stats the base has. On non-weapons all three are written |
| 7 | `dmg%` | v = prev or roll. **Weapon: if trunc(v·maxdam/100) = 0 the item gets +1 max damage (func 6) instead of any ED stat**; otherwise stats 18 and 17 = v. The base damage stats are reset to the Items values first |
| 10 | `skilltab` | v = prev or roll, stat 188 with param = tab%3 + (tab/3)·8 |
| 11 | ctc (`hit-skill` etc.) | no roll: chance = min (≤0 → 5), level = max; 0 → (ilvl−reqlvl)/4+1 capped at the skill's maxlvl; <0 → scaled by ilvl. Param = skill·64 + level |
| 14 | `sock` | n = min(invw·invh (≤6), max sockets by ilvl, v = roll, or param when ≤ 0) |
| 15 / 16 / 17 | elemental min / max / length, per-level props | 15 = min, 16 = max (no roll); 17 = the mod `param` if nonzero (e.g. `hp/lvl` param 6), else a roll |
| 19 | `charged` | level as in func 11; max charges = min (0 → 5; <0 → −min + (−min·level)/8), capped 255; current = max/8 + 1 + rand(max − max/8) |
| 20 | `indestruct` | stat 152 = 1 |
| 21 | class skills (`ama`..`ass`) | v = roll, param = Properties `val` (the class) |

- Writing stat 58 (poison max) also writes stat 326 = 1.
- Values are stored shifted by ItemStatCost `ValShift`, e.g. life <<8.

## 4. Rare items

### 4.1 Affix count: PD 0x102D57C0 applies to **every** rare (**VERIFIED**)

The patch at 0x6FC352BF replaces the stock table read inside the only rare loop, 0x6FC35240. It is not map-specific:
```
jewel (ItemTypes 58):   4
ilvl >= 85:             6
ilvl >= 65:             5 + pdRand % 2
ilvl >= 45:             4 + pdRand % 3
else:                   3 + pdRand % 4
stock (replaced):       [3,4,4,5,5,5,6,6][rand & 7]     (stock jewels: 3 + (rand & 1))
```
This count is the number of *successful rounds*. The 10,000 harness rares had 3/4/5/6 affixes 773/1,691/2,249/5,287
times.

### 4.2 The loop (0x6FC35240, **VERIFIED**)

1. Pick the rare names first (0x6FC34210, twice).
2. Then, for each round:
   - If both sides are open, draw `rand & 1`: 1 = suffix, 0 = prefix. A closed side sends the round to the other side.
   - Call the picker with force = 1, apply = 0.
   - A side closes after its **3rd** affix, or when its pick returns nothing (then the round is repeated).
   - When both sides are closed the loop ends early.
3. Mods are applied after the loop in the order P0, S0, P1, S1, P2, S2.
4. When no affix at all was taken, the item falls back to magic (0x6FC304F0).

The 3+3 cap is the only split rule, so a 6-affix rare is always 3 + 3.

### 4.3 Rare names (READ)

0x6FC34210 builds a list of RarePrefix/RareSuffix rows (record 0x48) allowed for the item (#10110 itype/etype test) and takes
`rand(n)`: uniform, with no frequency. The first call uses the second block of the name table (RarePrefix), the second the
first block (RareSuffix). This order is inferred from the table layout (READ).

### 4.4 Jewels

- Rare jewels always make 4 rounds (**VERIFIED**): 4 affixes, split 1+3, 2+2 or 3+1.
- Magic jewels are forced to prefix+suffix at ilvl ≥ 85.
- In PD2 data no IAS, FHR or −res jewel rows exist.
- The best rows (DATA, ilvl 99):

  | side | rows |
  |---|---|
  | prefix | Ruby 31–40 % ED (L66), Vermillion +11–15 max dmg (L58), Fortuitous 11–15 % MF (L46), Gorelust's 5 % open wounds + 95–125 deep wounds (L68) |
  | suffix | of Burning/of Thunder elemental (L57), of Virility +7–9 str (L50) |

## 5. Superior and low quality (READ + DATA)

**Superior** (0x6FC340B0):
- It draws `rand(n)` over the QualityItems rows until one fits the item (#11094 0x6FD95BE0). Every row is equally likely.
- n = 8, or 4 (only the rows without `dur%`) when the item has `nodurability` or its type is Throwable (#10711).
- The row's mods are applied with #10996 mode 1 (the mods are at record +0xC).

Fit rules:
- weapon rows need a weapon that is not exactly staf/bow/xbow/scep/wand;
- armor rows need armor that is not exactly shie/boot/glov/belt;
- those exact types use their own flag.

Rows (`qualityitems.bin`):

| row | mods | fits |
|---|---|---|
| 0 | att% 15–25 | weapons |
| 1 | dmg% 10–20 | weapons |
| 2 | ac% 10–20 | armor |
| 3 | att% 10–20 + dmg% 5–15 | weapons |
| 4 | dur% 15–25 | all |
| 5 | att% 10–20 + dur% 10–20 | weapons |
| 6 | dmg% 5–15 + dur% 10–20 | weapons |
| 7 | ac% 5–15 + dur% 10–20 | armor |

So:
- A normal weapon gets rows 0/1/3/4/5/6 (1/6 each). For example, P(superior weapon has ED) = 3/6.
- Armor gets rows 2/4/7.
- A `nodurability` weapon (62 PD2 weapons, e.g. Crystal Sword) only gets rows 0/1/3.
- Superior `ac%` goes through func 2, so **base defense = maxac + 1**.
- Superior ED on a weapon with a tiny max damage becomes +1 max damage (§3 func 7).

**Low quality** (0x6FC33E60): uniform name from LowQualityItems. Then:
- max durability = max(1, durability·33/100), current = max/2 + rand(max/2);
- weapon damage ×75/100 (min 1/2);
- armor defense ×75/100.

## 6. Sets and uniques (READ)

- Values roll through the same property functions (§3), so every range is uniform min..max on the item seed.
- Uniques: #10996 mode 3 applies all 12 UniqueItems props (0x6FC2F51F, 0x6FC2F6D2).
- Sets: mode 4 applies the 9 own props, then the 10 green bonus props, which are rolled at creation too (0x6FC33C09,
  0x6FC33DBE). Here an empty prop is skipped, not a stop.
- Set and unique items get no automagic and no class skills.
- Ethereal applies after the properties: PD2 removed the stock "sets cannot be ethereal" test (NOP at 0x6FC2EBF7).
- The stock socket roll only runs for normal/superior. Unique/set sockets come only from their own `sock` property.
  func 14 caps these by Items `gemsockets`, the ilvl MaxSock column and the inventory size.
- The `ethereal` property (func 23, 0x6FD97290) calls the same PD2 ethereal routine as the drop roll.

## 7. Base rolls

### 7.1 Defense and durability (READ; ED → max+1 VERIFIED)

0x6FC31070 runs on the **unit** seed.

Armor:
- block = Items `block`, speed penalty = −Items `speed`;
- current durability = d/2 + rand(d/2), max = d (both ≤ 255);
- defense = minac + rand(maxac − minac + 1) (0x6FC2F700).

Weapons: the same durability rule, the quantity for stackables, and −speed.

The final defense a player sees is `base + trunc(base·ED/100) + flat` (ItemStatCost op 13 on `armorclass`). The base is
maxac+1 as soon as any ac% mod was applied.

### 7.2 Durability changes

| case | durability |
|---|---|
| unique → rare fallback | ×3 (cap 255) |
| set → magic fallback | ×2 |
| low quality | 33 % |
| superior / affix `dur%` | stat 75 |
| PD2 ethereal | not halved (below) |

### 7.3 Ethereal, PD2 version (READ)

Roll (0x6FC2EBB0): weapon or armor, with durability, not low quality, not a quest item → `rand(100) < 5` on the item seed,
or forced by drop flags. Changes:
- **PD2 replaces the stock effect D2Common 0x6FD96A40 with PD 0x102680C0** (call patched at 0x6FC2EC42 and in func 23 at
  0x6FD972C4 / 0x6FD97771). The routine sets the item's ethereal flag.
- Armor: base defense (stat 31) × **5/4**.
- Weapons: stats 21/22/23/24/159/160 × **5/4**. Stock is ×3/2.
- **PD2 also skips the stock durability halving** (`je` → `jmp` at 0x6FC2EC4F). Ethereal items keep their full maximum
  durability.

### 7.4 Sockets (READ)

drops.md has the chance and the count: 33 %, count = init seed % max + 1, difficulty cap 3/4/6. The PD2 change is the
wrapper 0x102CB370 at both call sites (0x6FC306AF, 0x6FC30CA8), together with the NOP of the stock stackable test at
0x6FC2ECB5. Stackable items may socket only when their type is Throwing Knife, Throwing Axe, Javelin, Amazon Spear, Bow
Quiver or Crossbow Quiver. Non-stackable items are unchanged.

## 8. Automagic and class skills

### 8.1 Automagic (0x6FC30FCB → 0x6FC34B50, **VERIFIED**)

When Items `auto prefix` ≠ 0, the picker runs over the AutoMagic rows with:
- `group = auto prefix`;
- no `spawnable` test;
- forced, and never failing;
- normal alvl and `rare` rules: the 6 rows with `rare` = 0 (Archer's, Athlete's, Lancer's, of the Colossus, Great Wyrm's,
  Chromatic) cannot appear on rares;
- the frequency weighting (×level on wands, staves and orbs).

It runs for every quality except set and unique, after ethereal and sockets.

PD2 groups (DATA):

| items | automagic |
|---|---|
| most melee weapons and throwing axes | 308 Splashing |
| daggers | 309 Piercing (splash + 20 % deadly) |
| staves | 312–314 +10/30/50 % FCR + splash |
| orbs | 303 +life or +mana |
| Paladin shields | 304 res-all or AR/ED |
| Necromancer heads | 305 poison damage |
| Amazon bows, spears, javelins | 300/302 +skill tab |
| shields | 315–320 thorns / magic damage reduction |
| quivers | 321–326 |

### 8.2 PD2 class skills ("staffmods", **VERIFIED**)

The gate 0x6FC33DE0 needs ItemTypes `StaffMods` = a class (#10957, ItemTypes +0x1F). This is true for scep, wand, staf,
club, knif, h2h, h2h2, orb, head, phlm, pelt and cloa. It runs for low, normal, superior, magic and rare (and crafted).

PD2 replaces the stock routine (0x6FC33320/0x6FC335E0 → PD 0x102C5DF0 via 0x102ED270/0x102ED290, patched at
0x6FC33E21/3C/4D). All rolls use `pdRand`:
```
if ilvl == 1 and quality == low: none
r = pdRand % 100 + bonus;  n = r>=80 ? 3 : r>=50 ? 2 : r>=10 ? 1 : (bonus ? 1 : 0)      bonus = 0 (ilvl when drop flag 0x20)
list = PD list[class]; daggers (ItemTypes knif) use list 7
per skill (n times): up to 4 tries of list[pdRand % len]; reject if reqlevel != 1 and reqlevel + 7 > ilvl,
                     if already taken, Poison Dagger on wands, Holy Sword on scepters, Blade Dance always
                     (4 failures leave the slot empty)
level: low quality -> 1; else r = pdRand % (min(ilvl/3,10) + 90) + bonus/2 -> r>=75: 3, r>=40: 2, else 1
stat 107 item_singleskill (param = skill)
```
So with bonus 0: 10 % none, 40 % one skill, 30 % two, 20 % three. At ilvl ≥ 30 each skill is +1/+2/+3 with 40/35/25 %.

The lists were decoded from the static initialiser 0x10127B20 into container 0x104E3178:
- lists 0–6: each class's 30 skill-tree skills plus PD2's added skills. For example, the Sorceress list has Ice Barrage,
  Combustion and Lesser Hydra.
- list 7 (daggers): Poison Dagger, Tiger Strike, Blades of Ice, Fists of Fire, Claws of Thunder, Cobra Strike, Royal Strike,
  Dragon Talon, Dragon Tail, Dragon Flight, Blade Sentinel, Blade Fury, Blade Shield.
- list 8 (Sorceress elemental skills) is built but never selected (READ).

The list contents are READ; the logic is VERIFIED.

## 9. PD2-specific changes at item creation (summary)

| change | where | status |
|---|---|---|
| Magic items: both affixes forced at ilvl ≥ 65 (rings/amulets/jewels 85, charms 90, maps always) | PD 0x102D7170 at 0x6FC3040C/0x6FC3047C | VERIFIED |
| Rare affix count by ilvl (jewels 4) for **all** rares | PD 0x102D57C0 at 0x6FC352BF | VERIFIED |
| Class-skill lists and rules (daggers get their own list) | PD 0x102C5DF0 | VERIFIED |
| Ethereal = ×5/4 base damage/defense, no durability halving; sets can be ethereal | PD 0x102680C0; 0x6FC2EC4F; 0x6FC2EBF7 | READ |
| Socket filter for stackables | PD 0x102CB370; NOP 0x6FC2ECB5 | READ |
| Unique rarity without the min-1 clamp; TC-forced unique/set indices | 0x6FC2F5CE; 0x102D7270/0x102D7210 | drops.md |
| Maps dropped white | PD 0x102C8A54 | drops.md |
| A PD stat 437 on some drop-created uniques | PD 0x102C8C2F (per-unique PD table, `pdRand % n < 1`) | not traced |

Not changed by PD2: alvl, eligibility, weights, the draw, all property functions, superior, low quality, automagic and
the defense/durability rolls. There are no patch records in those functions. No corruption happens at item creation.
Corruption comes from the cube and map mods (maps.md).

## 10. Worked examples (PD2 data; `node test_items.js`, 10^6 rares each, ±95 % CI)

Rare Amulet, ilvl 99 (alvl 99, always 6 affixes):

| outcome | chance |
|---|---|
| +2 to any class skills (Valkyrie's … Witch-hunter's, level 90) | **14.71 %** (±0.07) |
| +2 Sorceress skills | **2.11 %** (±0.03) |
| +1 or +2 to any class | 44.08 % |
| +2 to a skill tab | 20.93 % |
| +2 class and ≥ 16 all res (Prismatic) | 1.13 % |
| +2 Sorc and ≥ 16 all res | 0.164 % |
| +2 class and +21–30 str | 0.79 % |

The class skills, the tabs and every other skill prefix share group 125, so an amulet has at most one of them.

Rare Diadem, ilvl 99 (alvl 99; weights ×level because of magic lvl 18):

| outcome | chance |
|---|---|
| +2 any class | **29.78 %** |
| +2 Sorceress | **4.28 %** |
| +2 class and 30 % FRW (of Speed) | 1.81 % |
| +2 Sorc and 30 % FRW | 0.26 % |
| +2 skill tab | 17.93 % |

A Circlet at ilvl 60 (alvl 63) cannot roll +2 class (level 90). It rolls +1 or +2 class 34.3 % of the time.

Rare Jewel, ilvl 99 (4 affixes):

| outcome | chance |
|---|---|
| 31–40 % ED | **5.93 %** |
| 40 % ED | 0.59 % |
| 31–40 % ED and +11–15 max damage | 0.26 % |
| 11–15 % MF | 6.31 % |

Rare Ring, ilvl 85: **0 %** for any +skills, because no skill affix lists rings. It rolls some FCR 13.7 % of the time.

Magic Grand Charm (cm3, exact):

| ilvl | forced | any skiller | a given tab (21 tabs, level 50, weight 1 each) | skiller + any +life suffix | skiller + 41–45 life |
|---|---|---|---|---|---|
| 50 | no | 4.93 % | 0.235 % | 0.58 % | – |
| 89 | no | 4.55 % | 0.216 % | 0.66 % | – |
| **90** | yes | **9.05 %** | **0.431 %** | 2.61 % | – |
| 99 | yes | 9.01 % | 0.429 % | 2.86 % | 0.357 % |

The skiller chance doubles at ilvl 90 because the prefix is then forced.

Worked value roll: a Prismatic res-all 16–20 is one `roll(16,20)` = 16 + rand(5), written to all four resist stats by funcs
1/3/3/3. The four values are always equal.

## 11. API (`items.js`)

```js
PD2Items.setData(await (await fetch('items-data.json')).json());
PD2Items.alvl('ci3', 80)                                  // 99
PD2Items.generate('amu', 99, 'rare', { seed: 1 })         // {name, prefixes, suffixes, mods:[{code,param,value}], stats, rareCount, ...}
   // opts: seed|rng, stock (1.13c rules), names, staffmods, automagic, ethereal, sockets, difficulty, staffBonus
PD2Items.magicOdds('cm3', 99)                             // exact per-affix probabilities {alvl, forced, affixes:[{name,side,p,mods}]}
PD2Items.rareOdds('amu', 99, 200000)                      // Monte-Carlo per-affix probabilities
PD2Items.chance('ci3', 99, 'rare', it => it.prefixes.includes("Arch-Angel's"), 200000)
PD2Items.candidates('jew', 99, { side: 'P', quality: 'rare' })   // [{affix, weight}]
PD2Items.rareCountDist('amu', 50)                         // {4: 1/3, 5: 1/3, 6: 1/3}
PD2Items.rollBase('uap', null, { ed: true, ethereal: true })     // base defense/durability (unit-seed rolls)
```
- Seeded runs reproduce the game's RNG stream when `rng = makeRng(itemSeedLo, itemSeedHi, unitSeedHi)`. That is how the
  harness compares.
- The rare odds are Monte-Carlo. 200k items give about ±0.06 % absolute on a 2 % event.

## 12. Bugs and quirks

| # | finding | status | verdict |
|---|---|---|---|
| 1 | The draw is `rand(W+1)`; r = W falls through to the **last** candidate, which gets weight w+1. The pick never fails. maps.md's "1/(W+1) no-pick" reading is wrong, and its "fewer than 6 affixes" can only come from empty pools. | VERIFIED | intended but surprising (stock off-by-one, harmless) |
| 2 | `dmg%` on a weapon whose trunc(ED·maxdam/100) is 0 gives **+1 max damage and no ED%**. Example: 10 % ED on a base with max damage ≤ 9. This also hits superior dmg% 5–15 on small weapons. | VERIFIED | intended but surprising (stock) |
| 3 | Any `ac%` mod (magic, rare, superior, unique, set) resets base defense to **maxac+1**. The unit-seed defense roll only matters for items without ac%. | VERIFIED (native write seen) | intended but surprising (stock) |
| 4 | Circlets, wands, staves and orbs weight affixes by frequency × level, so their high rows are much more likely. For example, P(+2 class) on a rare Diadem is 29.8 % against 14.7 % on an amulet. | VERIFIED | intended but surprising (stock) |
| 5 | Class-specific rows are allowed on any classless item in their itype list (e.g. +Paladin skills on swords) and are blocked only on other classes' class items. | VERIFIED | intended (PD2 data design) |
| 6 | The `Expansion` separator rows are dropped by the compiler. Every index after them (ItemTypes ≥ 58, Properties ≥ 121, affix and item rows) is one lower than the txt row. | DATA (bins) | not a game bug; a trap for tools |
| 7 | MagicSuffix "of Anima" lists itype `amu` (a typo for `amul`). The compiled `.bin` drops it, so it never rolls on amulets. | DATA (bin) | likely bug |
| 8 | Affix mods are applied until the first empty or unresolved property (−1), so a bad code in mod 1 would drop mods 2–3. Unique/set application only skips. No spawnable PD2 affix is affected. `map-mon-extra-mag` is unresolved in unspawnable rows. | READ + DATA | unclear (latent) |
| 9 | PD2 ethereal gives only +25 % base damage/defense and keeps full durability. The stock ×1.5 / half-durability code is bypassed. | READ | intended (PD2 design) |
| 10 | PD2 rare jewels always get 4 affix rounds regardless of ilvl. At ilvl < 85 other rares get a random 3–6 / 4–6 / 5–6. | VERIFIED | intended (PD2 design) |
| 11 | Staffmods: 4 failed tries leave the slot empty, so a "3-skill" roll can give 2 (or 1/0 at low ilvl). Blade Dance is in the Assassin list but always rejected. List 8 is built and never used. | VERIFIED (logic) / READ (lists) | unclear (probably intended filters; the dead list is a leftover) |
| 12 | Magic charms are forced only at ilvl ≥ 90, so the grand charm skiller chance jumps from 4.5 % (ilvl 89) to 9.05 % (ilvl 90). | VERIFIED | intended but surprising |
| 13 | The picker always draws its `rand & 1` even when forced, so forced calls still use RNG. | VERIFIED | neutral (matters only for seed replays) |
| 14 | Small Charm qlvl 28 → alvl = 2·ilvl − 99 above ilvl 85. An ilvl-90 small charm has alvl 81, so level-82+ small-charm rows need ilvl ≥ 91. | VERIFIED formula, DATA | intended but surprising (stock formula) |

## 13. Not verified / open

- Rare name order (which block is prefix) and the names' own RNG. The harness stubs the names; the model draws them
  first, as the code does. READ.
- Superior, low quality, set/unique application and the base defense/durability rolls were not run natively. The shared
  property functions they call were.
- PD2 ethereal (0x102680C0) and the socket filter (0x102CB370) are READ.
- The staffmods list contents come from a static decode of 0x10127B20 and were not produced by running the initialiser.
- The source of drop flag 0x20 (class-skill bonus = ilvl) is unknown.
- PD stat 437 on drop-created uniques is not traced.
- The Equiv closure matrix was built by the driver, not by D2Common's loader.
- Crafted/tempered items (0x6FC34BF0 / 0x6FC34B30) are left to the cube agent.
