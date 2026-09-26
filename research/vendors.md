# Gambling, vendors, prices and gold (PD2 = 1.13c D2Game/D2Common + ProjectDiablo.dll)

JS model: `vendors.js` (browser `window.PD2Vendors`, Node `module.exports`).
Harnesses: `harness/vend/gamble.c` with `gamble_check.py` and `gamble_check_js.js`; `harness/vend/price.c` with `price_check.js`.
Worked examples: `harness/vend/examples.js`.
Affix and stat rolling on the created items is covered in `items.md` (item-generation agent) and is not re-derived here.

Status labels:
- **VERIFIED**: the real code ran natively and matched the model on every case.
- **READ**: from the disassembly only.
- **DATA**: from the tables.

Table sources:
- Table facts come from `bin/pd2data.mpq`: the compiled `.bin` files the game loads, and `excel_mpq/*.txt` extracted from it.
- For every value used here, `Gamble`, `Npc`, `DifficultyLevels`, `Weapons`, `Armor`, `ItemTypes` and `ItemStatCost` are identical in `data.zip` and in `excel_mpq`. The `.bin` files agree with the txt for the rows that were spot-checked (ci0–ci3, aqv/cqv, tkf, jav, rin, amu).
- Stock 1.13c values come from the vanilla `patch_d2.mpq` / `d2exp.mpq` `.bin` files, read with `tools/mpq.py`.

## Native verification

| harness | real code run | cases | result |
|---|---|---|---|
| `vend/gamble` | Whole gamble-window fill D2Game `0x6FCDE2A0`: item-level roll, base pick from the gamble table, ring/amulet slots, classic/expansion filter, quality roll. Also the exceptional/elite upgrade `0x6FCDD040` and the RNG `0x6FC211D0` | 30,000 windows (420,000 slots). Random clvl, difficulty, classic/expansion, seed, and all five DifficultyLevels gamble columns (PD2 values and random ones) | **0 mismatches** (item id, ilvl, quality, final seed). The JS `gambleFill` also gives 0 mismatches on 20,000 windows. Mutations: an ilvl offset of −4 instead of −5 fails 1,957/2,000. The stub aborts if the fill reads any stat other than player level, so **MF is never read** |
| `vend/price` | PD2's price function PD `0x102529F0` with all of PD's helpers: `0x10251B90` gamble price, `0x10251D60` stat costs, `0x10252510` staff-mod skills, `0x10252380` charge repair, `0x10252820` sockets, `0x102528A0` affix cost, `0x10252960` vendor multiplier, the real D2Common muldiv `0x6FD511E0` and CRT `_alldiv` | 40,000 random synthetic items. Modes buy/sell/gamble/repair; qualities 0–10; identified/ethereal/starter/ear flags; armor, weapon, book, ammo, stackable, class and throwable items; affixes, set/unique records, up to 8 stats (encodes 0/1/2/3/5, ValShift), staff mods, charges, sockets, quest multipliers, reduced prices (including negative values and values >99), durability and replenish | **0 mismatches** against `vendors.js itemPrice`. Mutations: "superior counts stats" fails 1,003; "no 250k repair cap" fails 3,568; "ethereal treated as class item" fails 843 |

Stubs:
- **gamble:** #10041 gamble table, #10237 DifficultyLevels row, #10450/#10695 item records, #10973 (stat 12 only), Fog alloc, item creation `0x6FC31980` (records id/ilvl/quality), store placement `0x6FCF76A0`, and the post-creation item helpers (no-ops).
- **price:** all D2Common imports of PD's code: item fields, flags, types, stats, records and the Npc record. The data tables (ItemStatCost at PD2's 0x150 stride, Skills, ItemTypes, SetItems, UniqueItems) are laid out in memory at the offsets PD reads.

---

## 1. Gambling

### 1.1 Who runs it (READ)

The gamble window opens through D2Game `0x6FCDF730` → `0x6FCDF1C0`. For NPC classes 147 Gheed, 199 Elzix, 254 Alkor, 405 Jamella, 512 Anya and 514 Nihlathak, the NPC-interaction switch at `0x6FCE0223` sends interaction type 2.

- **PD2 hook.** PD2 replaces the tail jump at `0x6FCDF1EE` with a call to PD `0x102EF010` → `0x102D5E80`.
- **The event store.** That function builds a special store only when all of these hold:
  - the NPC is Gheed (class 0x93);
  - the NPC stands in a map level 137–201 (`0x102CE890`);
  - PD's game field `game+0x1DF4 == 0x12` (meaning unknown);
  - the per-game counter `game+0x2628` is not yet 2.

  Details are in 1.6.
- **Every other gamble window** goes to the **stock** fill `0x6FCDE2A0`. The pointer is resolved through PD's table `0x103D3E04` = D2Game+0xBE2A0. No PD2 patch lies inside `0x6FCDE2A0` or `0x6FCDD040`.
- **Regeneration.** A player's gamble list is freed when the NPC interaction ends (`0x6FD00C5E` → `0x6FCAFAF0`). It is rebuilt the next time the window opens (`0x6FCDF1C0` only fills when the player has no list). So **every opening of the gamble window rolls a new set of items**.

### 1.2 The 14 slots: D2Game `0x6FCDE2A0` (VERIFIED)

```
seed      = game+0x1D24 -> +8   (the game RNG; x = lo*0x6AC690C5 + hi, rand(n) = lo' mod n)
clvl      = player stat 12
for slot = 0 .. 13:
    ilvl  = clamp(clvl + rand(10) - 5, 5, 99)                  // clvl-5 .. clvl+4
    base  = gamble[ rand(count[ilvl]) ]                        // uniform over bases with Items.level <= ilvl
    if classic game and base.version >= 100: redo this slot (no slot used)
    if slot == 0: base = ring  ('rin ');  if slot == 1: base = amulet ('amu ')   // the random pick above is discarded
    base  = upgrade(base, ilvl)                                // expansion only, 1.3
    q     = magic
    r     = rand(100000)       if Rare+Set+Unique > 0
    if r < Unique -> unique ; elif r < Unique+Set -> set ; elif r < Unique+Set+Rare -> rare
    create(base, ilvl, q)                                      // D2Game 0x6FC31980, spawn type 4, unidentified
```

- The gamble table is D2Common #10041 `0x6FDC1280`, built at `0x6FDC2C50` from gamble.txt. It is sorted by the base's Items `level` and has `count[L]` = the number of bases with level ≤ L.
- Each slot uses **one** item level for both the base filter and the created item.
- Nothing else is read. **MF, the difficulty and the NPC have no effect.** The DifficultyLevels row is fetched for the current difficulty, but all three rows hold the same values.

### 1.3 Exceptional / elite upgrade: D2Game `0x6FCDD040` (VERIFIED)

```
if !expansion: keep
exc   = record of base.ubercode ; if none: keep
c     = (ilvl - exc.level) * GambleUber + 1          ; if c <= 0: keep (no elite roll either)
if rand(10000) < c: return exc
elite = record of base.ultracode ; if none: keep
c2    = (ilvl - elite.level) * GambleUltra + 1       ; if c2 > 0 and rand(10000) < c2: return elite
keep
```

The elite roll happens only when the exceptional roll fails:

```
P(elite) = (1 - Pexc) * Pelite
```

The game seed is the same one the whole fill uses.

### 1.4 The numbers (DATA)

| DifficultyLevels | GambleRare | GambleSet | GambleUnique | GambleUber | GambleUltra |
|---|---|---|---|---|---|
| PD2 (all difficulties) | 10000 | 100 | 50 | 90 | 33 |
| stock 1.13c (`patch_d2.mpq`) | 10000 | 100 | 50 | 90 | 33 |

Requested quality per slot:

| unique | set | rare | magic |
|---|---|---|---|
| 0.05 % (1 in 2,000) | 0.10 % (1 in 1,000) | 10 % | 89.85 % |

**What the requested quality becomes** (item creation `0x6FC30A20` switch, READ; the pickers are VERIFIED in `drops.md`):
- **Unique:**
  - The picker (`0x6FC2F370` via PD's wrapper `0x102D7270`) takes uniques with the base's code, `lvl ≤ ilvl`, enabled, and ladder-only in ladder games.
  - It weights them by `rarity`, and PD2 removes the minimum-1 weight.
  - It **fails if the picked unique was already generated in this game**, the once-per-game bit at game+0x1B24.
  - A failed unique becomes **rare with ×3 durability**, capped at 255 (`0x6FC30D64`).
- **Set:** the picker (`0x6FC33C20`) weights sets by `rarity` (0 counts as 1). A failed set becomes **magic with ×2 durability**.
- There is no MF anywhere in this chain.

**PD2 changes to the gamble list and bases** (DATA; PD2 `gamble.bin` / `armor.bin` / `misc.bin` compared with stock):
- **gamble.bin: 114 entries (stock has 125).**
  - Removed: Club, Spiked Club, Dagger, Dirk, Kriss, Blade, Katar, Wrist Blade, Hatchet Hands (`axf`), Cestus, Claws, Blade Talons, Scissors Katar.
  - Added: **Arrows `aqv` and Bolts `cqv`**.
- **Circlet `ci0`:**
  - PD2: ubercode = Coronet `ci1`, **no ultracode**.
  - Stock: ubercode Tiara `ci2`, ultracode Diadem `ci3`.
  - A gambled Circlet can therefore only become a Coronet. Diadems come only from Coronet slots.
- **Arrows/Bolts:** PD2 gives them ubercode/ultracode `aqv2`/`aqv3` and `cqv2`/`cqv3` (levels 25/45). Stock has none, so gambled quivers can be upgraded in PD2.
- **Throwing weapons:** throwing knife cost 6 → 352, javelin cost 5 → 300. Stack sizes changed too (the prices below depend on this).

### 1.5 Concrete odds (VERIFIED model, `vendors.js`)

A window has slot 0 = ring and slot 1 = amulet, plus 12 random slots. At clvl ≥ 57 every base is eligible, so each random slot is a given base with probability **1/114 = 0.877 %**.

**A gambled ring at clvl 90** (ilvl 85–94; every ring unique and set has lvl ≤ 84, so the pickers never fail on level):

| outcome | chance |
|---|---|
| unique | 0.05 % (1 in 2,000), if no ring unique was generated yet this game |
| set | 0.10 % |
| rare | 10 %, plus a failed unique |
| magic | 89.85 % |

Unique ring weights (PD2 rarity):

| unique | weight |
|---|---|
| Nagelring | 15 |
| Manald Heal | 15 |
| Dwarf Star | 10 |
| Raven Frost | 10 |
| Nature's Peace | 3 |
| Carrion Wind | 3 |
| Stone of Jordan | 1 |
| Constricting Ring | 1 |
| Bul-Kathos' Wedding Band | 1 |
| Wisp | 1 |
| **total** | **60** |

So an SoJ is 1/60 of unique rings: **1 in 120,000 gambled rings**.

Set ring weights: Cathan's Seal 70, Angelic Halo 30, Bul-Kathos' Death Band 2. So a Death Band is **1 in 51,000**.

**Circlets at clvl 90** (ilvl 85–94):
- A **Coronet slot** becomes a Tiara 17.56 % of the time and a **Diadem 1.208 %** of the time.
  - Diadem chance by ilvl: 0.01 % at ilvl 85, rising to 2.34 % at ilvl 94.
  - Worked example: at ilvl 90, exc = (90−70)·90+1 = 1801, so 18.01 %. Elite = (90−85)·33+1 = 166, so 1.66 %. Diadem = 0.8199 × 1.66 % = 1.36 %.
- A **Circlet slot** becomes a Coronet 33.76 % of the time and a Diadem **0 %** (PD2 data).
- Per random slot, a Diadem appears 1/114 × 1.208 % = 0.0106 %, so about **0.127 % per window**. A unique Diadem (Griffon's Eye, lvl 84) additionally needs the 0.05 % unique roll.
- Circlets and Coronets have **no unique** in PD2 UniqueItems. A unique roll on them always becomes a rare with ×3 durability.

Coronet → Diadem by clvl:

| clvl | 85 | 90 | 95 | 99 |
|---|---|---|---|---|
| Coronet → Diadem | 0.28 % | 1.21 % | 2.43 % | 3.10 % |

Other bases at clvl 90 (normal / exceptional / elite):

| base | normal | exceptional | elite |
|---|---|---|---|
| Cap | 44.9 % | 50.0 % | 5.18 % |
| Leather Boots | 46.8 % | 48.2 % | 5.03 % |
| `hbl` belt | 67.0 % | 32.0 % | 0.99 % |
| Arrows | 35.8 % | 58.1 % | 6.14 % |

### 1.6 PD2's map-level Gheed event store: PD `0x102D5E80` (READ)

- **When it is built:** under the conditions in 1.1, and only once per game (it sets `game+0x2628 = 2`).
- **Ring:** 1 time in 200 (C `rand()%200`), it first adds a ring with a fixed PD2 stat list: stats 264 = 75, 255 = 5–10, 58 = 50–75, 59 = 20–30, plus one of six extra stat pairs.
- **Uniques:** it then adds up to **5 uniques**, with up to 20 attempts:
  - Each attempt picks a UniqueItems row uniformly with C `rand()`.
  - The row is rejected if it is a jewel, is disabled, has rarity < 1, or has lvl outside 1..99.
  - If lvlreq ≥ 60, the row is also rejected when `rand()%100 < lvlreq − 59`.
  - Accepted rows are created at ilvl 99 (`0x102D5980`).
- **Price:** the PD price hook `0x102D6C60` charges these gamble-mode buys a flat price:

  ```
  1,000,000 · (100 − reducedPrices%) / 100
  ```

  This applies when the player is in a map level, the NPC is Gheed and the mode is 2.

### 1.7 Gamble price: PD `0x10251B90` (VERIFIED) vs stock `0x6FD74840`

The price uses the item's **normcode** record (#10059 → #10450). So Circlet and Coronet are separate families, but Coronet, Tiara and Diadem all cost the Coronet price. Likewise Cap, War Hat and Shako cost the same.

```
if code == 'rin ' or 'amu ':   price = base."gamble cost"           (ring 50,000, amulet 63,000)
X = uber  ? ((clvl - uber.level)*50 + 1, 0 if negative)  : 0        // uses clvl and 50, not GambleUber
E = ultra ? ((clvl - ultra.level)*25 + 1, 0 if negative) : 0
L = clvl < 6 ? 5 : clvl
W = trunc( ((10000 - E - X)*base.cost + E*ultra.cost + X*uber.cost) / 10000 )
lv = trunc( (L + max(0, base.level - 45) - floor(base.level/2)) * 250 / 3 )
price = trunc( (W + lv) * (trunc((2L+1)/3) + 20) / 15 )
then: price -= muldiv(reducedPrices%, price, 100)     (no minimum)
```

- **Classic items.** Items with a zero format word (itemData+0x30, which is 0 for classic-game items) instead pay the table "gamble cost" of their normcode record, with no reduction.
- **PD2 change.** Stock multiplies `base.cost` by `max(1, (minstack+maxstack)/2)`. PD2 dropped that factor, which matters for Arrows, Bolts and throwing weapons. Everything else is identical: the same constants and the same rounding (READ comparison).

Gamble prices at clvl 1 / 30 / 60 / 85 / 90 / 99 (PD2 data, no reduced prices):

| family | clvl 1 | 30 | 60 | 85 | 90 | 99 |
|---|---|---|---|---|---|---|
| Ring | 50,000 | 50,000 | 50,000 | 50,000 | 50,000 | 50,000 |
| Amulet | 63,000 | 63,000 | 63,000 | 63,000 | 63,000 | 63,000 |
| Circlet | 17,506 | 36,000 | 65,764 | 102,148 | **109,818** | 125,193 |
| Coronet / Tiara / Diadem | 33,478 | 63,776 | 105,664 | 150,940 | **162,976** | 187,107 |
| Cap / War Hat / Shako | 736 | 6,837 | 28,424 | 73,873 | 84,528 | 105,906 |
| `hbl` belt (Armor.txt "Girdle(H)") | 2,149 | 9,290 | 26,136 | 55,989 | 65,696 | 85,180 |
| Arrows | 1,030 | 7,349 | 21,024 | 37,673 | 41,365 | 48,767 |
| Throwing Knife | 1,050 | 7,381 | 24,824 | 50,948 | 57,061 | 69,338 |

Worked example, Circlet at clvl 90:
- X = (90−52)·50+1 = 1901.
- W = (8099·12000 + 1901·23000)/10000 = 14,091.
- lv = (90 − 12)·250/3 = 6,500.
- Price = (14,091 + 6,500)·(60+20)/15 = **109,818**.

With 10 % reduced prices, a Coronet costs 146,679.

---

## 2. Vendor inventories

### 2.1 Tables (READ + DATA)

At game start, D2Game `0x6FCAF1B0` builds one list per vendor index from Weapons, Armor and Misc. Items with `spawnable` and a non-zero `<Vendor>Max` or `<Vendor>MagicMax` are included.

- **PermStoreItem = 1** goes to a code list of permanent items.
- **Everything else** goes to a list of `{Min, Max, MagicMin, MagicMax, code, MagicLvl}`.

Column offsets: vendor index v at +0x146/+0x157/+0x168/+0x179/+0x18A + v. The index order is:

| v | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| vendor | Akara | Gheed | Charsi | Fara | Lysander | Drognan | Hratli | Alkor | Ormus | Elzix | Asheara | Cain | Halbu | Jamella | Malah | Larzuk | Drehya |

PD2 wraps each per-game copy (`0x6FCAF4E0` → PD `0x102F0F70` → `0x102CBB20`) and **rewrites the permanent arrow and bolt codes by difficulty**:

| difficulty | arrows | bolts |
|---|---|---|
| Normal | `aqv` | `cqv` |
| Nightmare | `aqv2` | `cqv2` |
| Hell | `aqv3` | `cqv3` |

PD2 misc items sold by vendors (DATA; PermStoreItem = 1; Misc.txt cost):

| vendor | item | cost |
|---|---|---|
| Akara | Token of Absolution `toa` | 500,000 |
| Akara | PvP maps `t61`/`t62` | 10 |
| Anya (Drehya) | Imbue Magic | 50,000 |
| Anya (Drehya) | Imbue Rare | 200,000 |
| Anya (Drehya) | Reroll Rare | 150,000 |
| Anya (Drehya) | Scour | 50,000 |
| Anya (Drehya) | Upgrade Magic | 125,000 |
| Anya (Drehya) | Upgrade Map | 200,000 |
| Anya (Drehya) | Fortify Map | 15,000 |
| Larzuk | Larzuk's Malus `lmal` | 654,500 |
| Arrow/bolt vendors | Arrows2/3, Bolts2/3 | as above |

### 2.2 Stock generation: D2Game `0x6FCDF2E0` (READ)

**Store level** (`0x6FCDCB20`, input clvl+5):

```
storeLevel = Normal : min(clvl + 5, [12, 20, 28, 36, 45][act])
             NM/Hell: clvl + 5            (no cap; can exceed 99)
```

For each random entry whose base `level ≤ storeLevel` (classic games also skip expansion bases):

- **White items:**
  - Count: `rand in [Min, Max]`, but **none at all when storeLevel ≥ 25**.
  - Each one's quality is rolled with `r = rand(100)`:

    | store level | low | normal | superior |
    |---|---|---|---|
    | < 5 | 9 % (r > 90) | 91 % | – |
    | 5–9 | – | 86 % | 14 % (r > 85) |
    | ≥ 10 | – | 75 % | 25 % (r ≥ 75) |

- **Magic items**, only if the base can be magic (`0x6FC2AC16`) and `MagicLvl ≤ storeLevel`:
  - `bonus = 1` when storeLevel < 25, otherwise `bonus = 2 + rand(2)`.
  - Count: `MagicMin + rand(MagicMax + bonus − MagicMin)`, which gives `MagicMin .. MagicMax + bonus − 1`.
- **Item level** of every random item = storeLevel. The affixes of magic items follow `items.md`.
- **Base upgrade** (`0x6FCDEE30`), only if difficulty > 0 **and clvl > 25**. One roll `r = rand(100000)`:
  - **NM:** `r < storeLevel·64 + 4000` → exceptional; otherwise the NightmareUpgrade code.
  - **Hell:** `r < storeLevel·16 + 1000` (expansion only) → elite; else `r < storeLevel·128 + 5000` → exceptional. The HellUpgrade code (Items +0x1A0), when not `xxx`, then overrides the result.
  - Example, Hell at clvl 90 (storeLevel 95): elite 2.52 %, exceptional 14.64 %. NM at storeLevel 95: exceptional 10.08 %.
- **Retries:** up to 5 creation retries (low-quality results of one base type are rejected), and 2 retries if the created code differs. The whole fill stops after 32 failures.
- **Permanent items** are created normal (quality 2). PD2 routes them through `0x102EEEE0` → `0x102D70A0`, which is only a null check. Arrows and bolts get a full stack.

**No rares.** Vendors never sell rares, sets or uniques in PD2. The store path only passes qualities 1–4, and no PD2 patch touches `0x6FCDEE30` or the random loop. The only exceptions are the two special PD2 stores:
- the Gheed event store (1.6);
- the **PvP-map vendors** (levels 157/159/166), PD `0x102D5BF0`. This store always refreshes. It sells PvP Mana Potions `pvpp` (ilvl 99) and PvP small/medium/large charms `cm1p`/`cm2p`/`cm3p` with preset stat lists from a PD table.

### 2.3 Refresh rules (READ)

- The store is **per NPC and per game**, shared by all players.
- It is filled on the first trade (`0x6FCDF730`, flag `store+0x20`).
- When a player leaves or enters a town, `0x6FCB0560`/`0x6FCB0C80` → `0x6FCB0320` visits every filled store of that act.
  - **If no player is left in that town:** each store whose NPC nobody is trading with is cleared and refilled on its next opening. If someone is trading, the store is marked pending (`+0x27`).
  - **Otherwise:** a store is only marked pending once **4 minutes** (240,000 ms, `+0x28`) have passed since its last mark.
- A pending store is cleared and refilled the next time any player opens the trade window (`0x6FCDF810`).
- **PD2:** the stock code clears `+0x27` in `0x6FCDF88C`; PD2 NOPs that clear and does it inside its fill hook instead, setting it to 0 for normal stores and 1 for the always-refreshing PvP stores. Normal vendors behave as in stock.
- The **gamble** list is per player and regenerates on every opening (1.1).

---

## 3. Prices: PD `0x102529F0` (VERIFIED)

PD2 routes D2Common #10107 (`0x6FD79D60`, used by D2Game and D2Client) to PD `0x102EF070` → `0x102D6C60`. That function handles the event-Gheed special case (1.6) and otherwise calls PD's own copy `0x102529F0` of the stock `0x6FD78E80`.

The repair-all functions also go to PD `0x10253300`, from D2Game `0x6FCDDF33`/`0x6FCDDFC9` and from D2Client `0x6FAF7DDB`/`0x6FB3F391`. That function sums `0x102529F0` in mode 3 over the 13 equipped slots that need repair. The client display and the server charge therefore agree.

Arguments are `(player, item, difficulty, questData, npcClass, mode)`, with mode 0 = buy, 1 = sell, 2 = gamble, 3 = repair.

### 3.1 Formula

All divisions truncate. `m(v, M) = v ≥ 65536 && M ? (v >>> 10)·M : trunc(v·M/1024)` (PD `0x10252960`).

**1. Early exits**
```
mode 3 and item cannot be repaired (#10087)        -> 0
item flag 0x20000 (starter item)                    -> 1
mode 2 -> gamble price (1.7)
```

**2. Base values A (buy), B (sell), C (repair)**
```
ear (flag 0x10000):      A = B = earLevel·cost, C = 0
body part (type 40):     A = B = cost + 8·MonStats word(+0xAA + 2·difficulty), C = 0
book (type 18):          A = B = Books.costPerCharge·qty + cost, C = 0
ammo (ItemTypes quiver): A = B = trunc(cost·qty/1024), C = trunc(maxQty·cost/1024)
otherwise:               A = B = C = cost ; stackable -> divisor = max(1, maxQty) (only if maxQty >= 2)
armor (type 50), maxac != 0 and maxac-minac != -1:  A = B = C = trunc(cost·defense/maxac)
```
Here `qty = max(1, stat 70)`, `maxQty` = #10463 and `defense` = base stat 31.

**3. Staff mods (non-magic items)**
```
if quality not in 4..9: staff-mod skills (PD 0x10252510)
```
This applies when ItemTypes StaffMods ≠ 7. For each stat 107 entry (skill, v), with the skill's cost mult M and add D and f = 2v−1:
```
A += (trunc(A·M/1024)+D)·f / divisor
B += (trunc(B·M/4096)+D)·f / divisor
C += (trunc(C·M/1024)+D)·f / divisor
```

**4. Identified items only**
```
acc = 0
automagic affix, then by quality:
   1 low      acc = (-A/2, -B/2, -C/2)           (replaces the automagic cost)
   2 normal, 3 superior: nothing                 <- PD2: superior no longer adds stat costs
   4 magic    prefix0 + suffix0 records, then stat costs
   5 set      SetItems cost mult/add
   6 rare, 8 crafted: prefixes 0-2 + suffixes 0-2, then stat costs
   7 unique   UniqueItems cost mult/add (a unique without a record is priced as magic)
   9 tempered stat costs
record cost (PD 0x102528A0): acc += trunc(X·mult/1024) + add for X = A, B, C
stat costs (PD 0x10251D60), for each item stat (v >> ValShift), added to A, B, C (divided by divisor):
   default encodes : trunc(X·Multiply·v/1024) + Add            (ItemStatCost Multiply/Add)
   encode 1        : Skills cost mult/add of the skill; B uses /4096
   encode 2, 3     : as encode 1 with v = skill level (param & 63)
   (when X·v >= 65536 the product is taken as trunc(X·v/1024)·Multiply)
A += acc.a/divisor ; B += acc.b/divisor ; C += acc.c/divisor ; magic+ items: staff-mod skills now
```
An unidentified item keeps its bare base value. For example, an unidentified rare ring sells for 1800/2 = 900.

**5. Adjustments to B and C**
```
sockets: A, B, C += trunc(childBaseCost/2) for every socketed item (runes, gems, jewels)
B /= 4 if ethereal ; B /= 4 if class-specific type (ItemTypes Class)
sell of a broken ethereal (durability < 1): B = 0
repair (not ammo, not throwable, has durability):
   C = maxDur <= dur ? 0 : (maxDur - dur)·C / maxDur                   (64-bit)
   replenishing durability (stat 252): C = (maxDur-1)·C / maxDur (0 if maxDur-1 <= dur)
```

**6. Vendor multipliers**
```
A' = m(A, "sell mult"), B' = m(B, "buy mult"), C' = m(C, "rep mult")         (Npc record +4/+8/+0xC)
for each quest slot with flag != 0 and quest state bit 0 or 1 set (#10174):
   A' = m(A', questsellmult), B' = m(B', questbuymult), C' = m(C', questrepmult)
```

**7. Quantity** (not book, not ammo, and **PD2: not a weapon**)
```
A' ·= qty
stackable and repairable: B' = maxQty·B' - (maxQty-qty)·C'  (no replenish)  or  B' = maxQty·B', C' = 0
otherwise B' ·= qty
```

**8. Charges** (repair mode, not ethereal)
```
C' += sum over item_charged_skill entries with cur < max:
      (cost add + (lvl + 2·trunc(reqlevel/6) + 2)·10000·costmult/1024) · (max-cur)/max
```

**9. Result**
```
sell   : max(1, min(B', Npc "max buy"[difficulty]))
repair : C' -= muldiv(reduced%, C', 100); C' <= 0 -> 1; min(C', 250,000)      <- PD2 cap
buy    : A' -= muldiv(reduced%, A', 100); max(1, A')
```

`reduced%` is player stat 87 (`item_reducedprices`), capped at 99. It does not apply to sell prices. `muldiv` (D2Common `0x6FD511E0`) computes `(A/100)·%` once the price exceeds 1,048,575.

### 3.2 Npc.txt (DATA; identical to stock 1.13c `npc.bin`)

Names are from the vendor's point of view: "sell mult" is what you pay, and "buy mult" is what you get.

| vendors | sell (you pay) | buy (you get) | repair | quest slots (flag: questsell) | max buy N / NM / H |
|---|---|---|---|---|---|
| Gheed | 1088 (106 %) | 512 (50 %) | 128 (12.5 %) | 4: 922 | 5,000 / 30,000 / 35,000 |
| Charsi | 960 (94 %) | 512 | 128 | 4: 922 | 5,000 / 30,000 / 35,000 |
| Akara | 1024 | 512 | 128 | 4: 922 | 5,000 / 30,000 / 35,000 |
| Act 2 | 1024 | 512 | 128 | 9: 922 | 10,000 / 30,000 / 35,000 |
| Act 3 | 1024 | 512 | 128 | 17: 922 | 15,000 / 30,000 / 35,000 |
| Act 4 (Jamella, Halbu) | 1024 | 512 | 128 | 41: 922 | 20,000 / 30,000 / 35,000 |
| Act 5 | **2048 (200 %)** | 512 | 128 | 41: 922, **35: 512** | 25,000 / 30,000 / 35,000 |

The question columns hold quest-record flag numbers; which quests they are was not traced. All questbuymult and questrepmult values are 1024. PD2 adds a row `gheedEventInventory` with the Act 5 values; it is used only by the event store.

### 3.3 Worked examples (PD2 numbers, VERIFIED model)

- **Strong Healing Potion** (cost 250):
  - Jamella: 250.
  - Malah: 500.
  - Malah after quest flag 41: 450.
  - Malah after flags 41 and 35: 225.
- **Anya's map items** (Act 5, no quests / both quests):

  | item | no quests | both quests |
  |---|---|---|
  | Imbue Magic | 100,000 | 44,544 |
  | Imbue Rare | 399,360 | 179,712 |
  | Reroll Rare | 299,008 | 134,144 |
  | Upgrade Map | 399,360 | 179,712 |

  Larzuk's Malus costs 1,308,672, or 588,800 with both quests.

  The odd last digits come from the `(v>>10)·M` step for values ≥ 65,536: 500,000 → 499,712.
- **Token of Absolution** at Akara: 499,712, or 449,936 after quest 4.
- **Harlequin Crest** (Shako: cost 56,307, defense 141 of max 141; UniqueItems cost mult 3, add 5000):
  - Value A = 56,307 + trunc(56,307·3/1024) + 5000 = 61,471.
  - Buy value 61,471. Sell: 5,000 (Normal act-1 cap) / 30,000 (NM cap) / **30,735** (Hell, under the 35k cap).
  - Repair at 0/12 durability: 61,471·128/1024 = **7,683**. At 6/12: 3,841. At 11/12: 640.
  - Ethereal: the sell value drops to 7,683 before the cap.
- **Magic ring "Shimmering … of Fortune"** (all resistances +5, MF +20):
  - The prefix record adds trunc(1800·1280/1024)+4000 = 6,250.
  - The four resistances add 4·(trunc(1800·20·5/1024)+43) = 872, and MF adds trunc(1800·102·20/1024)+577 = 4,162.
  - A = 1800 + 5,034 + 6,250 = **13,084**. Buy 13,084 at Akara; sell 5,000 (Normal cap) or 6,542 (Hell).
- **Uniques and sets** are priced only by their record's cost mult/add (a few %, plus 5000). Their stats never count.
- **Superior armor** (cost 40,000, +15 % defense stat): buy **37,500** from Charsi. The same item with its stat counted (the stock rule for superior, computed with the model) would be 48,512.

### 3.4 PD2 changes versus stock `0x6FD78E80` (READ side-by-side; the PD2 side is VERIFIED)

1. **Superior** items no longer add their stat costs. Stock jump table `0x6FD79CDC` sends quality 3 to the stat accumulator `0x6FD79823`; PD sends it past (`0x10252F31`).
2. **Repair costs are capped at 250,000** per item (`0x1025328C`). The repair-all sum is the sum of the capped items.
3. **Weapons** (ItemTypes `weap`, which includes throwing knives, axes, javelins and throwing potions) skip the quantity step. The buy price is not multiplied by quantity, and the sell price does not use the stack formula. PD2 data raised those bases' costs instead (tkf 6 → 352, jav 5 → 300).
4. **Gamble price:** the stack factor is removed (1.7).
5. **Repair** uses a 64-bit `(maxDur−dur)·C/maxDur`. Stock uses a 32-bit `imul`/`idiv` with a stale `edx`.
6. **Event Gheed** gamble price is a flat 1,000,000 (1.6).

Inventory and stash gold both pay (the buy handler compares the price with stat 14 + stat 15).

---

## 4. Gold

- **Drops:** already in `drops.md`:
  - `amount = ilvl + rand(5·ilvl)`;
  - then the TC entry `mul` (`>>8`);
  - then gold find `max(0, trunc(g·(100+GF)/100))`, including the PD map mods.
- **Inventory gold cap:** PD2 patches D2Common #10049 at `0x6FD81944` to call PD `0x10269130`, which returns a flat **5,000,000**. Stock is `clvl·10,000`.
- **Personal stash cap:** stat 15, D2Common #11060 `0x6FD7E9C0` = **2,500,000** (stock, unpatched). BH's "Stash Gold" line prints stat 15 only.
- **Shared stash gold (PD2):** a separate counter at PlayerData+0x50E (PD `0x102D0430`/`0x102D0490`). It is moved by PD packet handler `0x102E61A4`:
  - **deposit (sub-command 0x1A):** takes from inventory gold, up to a shared cap of **10,000,000** (a partial deposit fills to the cap).
  - **withdraw (0x19):** limited by the 5,000,000 inventory cap; the rest stays. It errors when the inventory is already full.
  - The counter is not a stat, so BH's panel and the death penalty ignore it.
- **Stat setters:** `0x6FC21040`/`0x6FC21670` set stat 14 or 15 to **0** when a new value would exceed its cap (READ). Callers normally check the cap first.
- **Gold lost on death:** D2Game `0x6FC58180` (READ). PD2 skips it in the PvP levels 157, 159 and 166 (`0x102C9B50`), as `experience.md` notes.

  ```
  total = inventory + stash(stat 15)
  loss  = trunc(min(clvl,20) · total / 100)
  if game type (game+0x6A) == 3:  loss = min(loss, inventory) and total-loss >= clvl·500 is kept
  killed by a player or a player's minion: the loss is dropped as a pile (taken from the stash if the inventory is short)
  otherwise: the rest of the inventory gold (inventory - loss) is dropped as a pile at the corpse,
             the loss disappears (from the stash when it exceeds the inventory, outside game type 3)
  ```

  Example: clvl 90 with 1,000,000 carried and 2,500,000 in the stash loses 700,000 (20 % of 3.5 M). The remaining 300,000 lie on the ground.

---

## 5. JS API (`vendors.js`)

```js
PD2Vendors.gambleQualityOdds()                      // {unique:.0005, set:.001, rare:.1, magic:.8985}
PD2Vendors.upgradeOdds(ilvl, excLvl, eliteLvl)      // {normal, exceptional, elite}
PD2Vendors.gambleSlotOdds(clvl, 'ci1', tables)      // per-slot odds averaged over the ilvl roll
PD2Vendors.gambleFill({clvl, expansion, seed:{lo,hi}, tables})   // exact window (VERIFIED)
PD2Vendors.gamblePrice(base, clvl, reducedPct)      // base = normcode record {code,cost,gambleCost,level,uber,ultra}
PD2Vendors.itemPrice(item, {mode, difficulty, clvl, reducedPct, npc: PD2Vendors.npcCtx('act5',[1,1])})
PD2Vendors.storeLevel(diff, act, clvl); storeQualityOdds(storeLevel); storeUpgradeOdds(diff, storeLevel, clvl)
PD2Vendors.deathGoldLoss(clvl, inventory, stash, realm); PD2Vendors.GOLD
```

`tables` = `harness/vend/gamble_tables.json`, built by `gamble_data.py` from `excel_mpq`: `{items, gamble (ids sorted by level), counts}`.

---

## 6. Bugs and quirks

1. **Circlet → Diadem is impossible in PD2** (ci0 has ubercode `ci1` and no ultracode; stock had ci2/ci3). Only Coronet slots can become Diadems (1.2 % at clvl 90). *Intended but surprising* (a data change).
2. **Unique rolls on Circlets/Coronets always turn rare.** There is no unique for `ci0`/`ci1`. The same happens for any base whose uniques are all above the slot's ilvl, or are already generated in this game. *Intended but surprising.*
3. **Gamble price uses clvl and fixed 50/25, not the ilvl and DifficultyLevels 90/33** of the real upgrade roll. The price's "expected base value" does not match the actual odds (price assumes an exceptional share of (clvl−lvl)·0.5 %; the roll gives 0.9 %). It is stock behaviour. *Intended but surprising.*
4. **Superior items are priced like normal items** (PD2 skips their stat costs). *Intended* (it removes the stock superior-armor buy/sell margin).
5. **Replenish-durability repair quirk (stock):** with stat 252 the repair cost is `(maxDur−1)/maxDur` of the full cost whenever the item is below max−1, however little is missing. *Likely bug* (stock, kept by PD2).
6. **Three identical quivers in NM/Hell.** Vendors carry `aqv`, `aqv2` and `aqv3` as permanent items, and PD2's per-difficulty rewrite maps all three to the same code. *Unclear* (cosmetic).
7. **The gamble window re-rolls on every open**, and the RNG is the shared game seed. *Intended* (stock).
8. **Unidentified items sell for base value only** (no affix, stat or unique cost). *Intended* (stock).
9. **Class items and ethereal items sell for ¼** (both apply: ethereal class items sell for 1/16). *Intended* (stock).
10. **The buy transaction mode comes from the client packet.** In the 0x32 buy packet (+9 upper word), `0x6FCDE640` only checks ownership for modes 0 and 2; any other mode goes straight to pricing. The rest of the purchase path (`0x102D6BA0`) was not traced, so exploitability is unknown. *Unclear.*
11. **Throwing weapons ignore quantity** when bought or sold in PD2, so a full stack and a nearly empty one are worth the same. *Intended but surprising.*
12. **The stat setters zero gold on overflow** (`0x6FC21040`/`0x6FC21670`). *Unclear* (depends on the callers' checks).
13. **Shared stash gold is invisible to BH's "Stash Gold" line** (stat 15 only) and is never lost on death. *Intended but surprising.*

## Unverified

- **READ only:**
  - store generation, refresh and timers;
  - the Gheed event store and the PvP store;
  - the death gold loss and the shared-stash handlers;
  - the unique→rare / set→magic fallbacks (the pickers themselves are VERIFIED in `drops.md`);
  - the stock-vs-PD2 price diff (only the PD2 side ran).
- **Stub semantics:** the meaning of each D2Common helper stubbed in the price harness (e.g. #10087 = "can be repaired", #11144 = ItemTypes quiver field) is READ. The harness verifies PD's arithmetic and control flow given those semantics.
- **Quest identities:** which quests the Npc quest flags 4/9/17/41/35 are, and what quest states 0/1 mean.
- **Game field:** the meaning of `game+0x1DF4 == 0x12` (the event condition) and of `game+0x6A == 3` (the death-gold rule).
