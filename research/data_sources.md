# Which data tables does live PD2 use? (data.zip vs pd2data.mpq vs the loose excel folder)

Status key (as in FINDINGS.md): **READ** = from the disassembly, **DATA** = from comparing the table files,
**VERIFIED** = run natively. Nothing in this report was run natively, so there are no VERIFIED claims here.

## 1. Verdict

**The live game uses the tables inside `pd2data.mpq`.** This is `ProjectD2/Live/pd2data.mpq`, which is byte-identical
to `bin/pd2data.mpq` (MD5 `74131305932854a2803db4a0c9272ac9`).

**Status:** READ for the loader path, DATA for the table evidence. Four pieces of evidence:

1. **The MPQ has priority over the stock MPQs.** PD2 opens `pd2data.mpq` with load priority 6000 (READ).
   - It also opens `pd2maps.mpq`, `pd2assets.mpq` and `pd2monchars.mpq`. None of these three were uploaded.
   - Section 2.2 has the details.
2. **The `-txt` flag cannot change any value.** In the MPQ, every compiled `.bin` is exactly what the MPQ's own `.txt`
   compiles to, using the MPQ's own `patchstring.tbl` (DATA).
   - Across 30 tables, no field from the MPQ .bin disagreed with the MPQ .txt that the compiler model can check. The
     few leftover mismatches (§3) are model limits, and they show up identically for the zip.
   - So the game gets the same values whether it loads the `.bin` files (the default) or compiles the `.txt` files
     (`-txt`).
3. **`data.zip` is an older snapshot.** Its `.bin` files match its own `.txt` files, so it is internally consistent too.
   But it lacks content that the live DLL hard-codes (DATA + READ):
   - PD's level→map-item switch (PD `0x102CEB40`) returns the codes `t57`, `t58` and `t3b` for levels 197, 200 and 201.
     These codes exist only in the MPQ's Misc.txt.
   - PD's Corrupted-player ear drop (PD `0x102C377E`…`0x102C3C4B`) creates `ivea`…`ives`. These also exist only in
     the MPQ.
   - PD's built-in "T1 map levels" set `{143,145,146,148,151,155,158,169}` (built at PD `0x10126E30` from
     `0x1037FE10`/`0x1037FE40`) matches the MPQ's `t1m` maps (t11, t21, t22, t24, t26, t33, t34, t36). It does not
     match the zip's `t1m` maps (t12, t15, t17, t22, t23, t25, t26, t36).
   - The zip is not even consistent with itself. Its own `TreasureClassEx.txt`, which is identical to the MPQ's, lists
     `t3b` in "Map Tier 3", but the zip's `Misc.txt` has no `t3b` row.
4. **The loose folder `ProjectD2/data/global/excel/` is not the live data.** It is a different branch: Diablo 1 /
   Tristram content and "Corpse Instability" (DATA + READ).
   - Its Misc.txt uses the old (zip) map tiers plus three extra maps, `pwl`, `dtm` and `dhm`. Its Levels.txt adds
     levels 202–211.
   - The live ProjectDiablo.dll never mentions any of these:
     - No `dhm`/`pwl`/`dtm` codes.
     - The map switch stops at level 201.
     - The T1 set contradicts the loose tiers.
   - A 1.13c client reads loose files only with `-direct`.
   - The uploaded saves were written by that branch. `RoofooSinElev.d2s` holds `dhm`, `dtm` and `pwl` items and has
     1000 unspent stat points, so it looks like a test character.

**What this means for our models:**
- The only tables that really changed are Misc, Levels (one row), CubeMain, AutoMap, LvlPrest, LvlTypes and the
  string table `patchstring.tbl`.
- The numeric values that matter for combat and speed are the same in the zip and in the live data. That covers
  skills, missiles, monster stats, MonLvl, weapons, armor, ItemStatCost, uniques, sets, runewords, affixes and
  treasure classes.
- Most other `.bin` differences are only **string-table indices**, because the MPQ's `patchstring.tbl` has 103 extra
  keys, which shifts the indices of later strings.

## 2. How PD2 loads its tables

### 2.1 Stock 1.13c loader: D2Common `#10849` (0x6FDAEF40, DATATBLS_CompileTxt) (READ)
```
if ([0x6FDF145C] != 0)                       // "compile txt" flag
    read  DATA\GLOBAL\EXCEL\<name>.txt       // "leveldefs" reads levels.txt (0x6FDDF8E8 / 0x6FDDFA90)
    parse with the column descriptor          // Fog #10208 / #10210 / #10207, then #10209 frees
    0x6FDAD770: write DATA\GLOBAL\EXCEL\<name>.bin to disk ("%s\%s", "wb")
if ([0x6FDE9E20] != 0)                       // static .data value 1, never written
    load DATA\GLOBAL\EXCEL\<name>.bin        // 0x6FDAF09D -> 0x6FD59900 (file read through Fog/Storm = MPQ chain)
    records = file+4, count = *(u32*)file
```
- **Who sets the compile flag:** `#10563` (0x6FDAD750) sets `[0x6FDF145C] = (arg == 0)`.
- **Its only stock caller** is D2Client 0x6FAF1C30. That function passes `!cfg->byte[0x211]`, the client-config "txt"
  byte that the launcher's command-line parser fills in for `-txt`.
- **So by default the game loads the `.bin` files.** With `-txt` it compiles the `.txt` files first. Either way the
  files come through the Storm MPQ chain, and loose files are used only with `-direct`.

### 2.2 What PD2 changes (READ)
- **MPQs.** PD 0x102AD5D8–0x102AD647 calls D2Win's MPQ loader (through `[0x104E4DE8]`) four times with priority
  `0x1770` (6000):
  - `pd2data.mpq` "PD2DATA"
  - `pd2assets.mpq`
  - `pd2maps.mpq`
  - `pd2monchars.mpq`
  - With the higher priority, PD2 files override `patch_d2.mpq`, `d2exp.mpq` and `d2data.mpq`, which all contain
    their own `monstats.bin` etc.
- **`-compile` switch.** PD 0x102AD6A0 scans the command line for `-compile` and `-plugy`. `-compile` (PD 0x102AD530)
  does the following, then exits: `D2Lang #10008("ENG")`, `D2Common #10563(0)` (compile txt), `#10943(0,0,0)` (load
  every table, so all `.bin` files are rewritten), then `ExitProcess(0)`. This is a developer tool for producing the
  `.bin` files, not a runtime mode.
- **ItemStatCost has its own loader.** The D2Common call at 0x6FDB6196 is redirected to PD `0x10250DE0`. That function
  calls `#10849("itemstatcost")` with PD's own descriptor and record size 0x150 (stock is 0x144).
  - It adds three columns, `save bits s12` (+0x144), `save add s12` (+0x148) and `save param bits s12` (+0x14C).
  - It still goes through the stock txt/bin switch.
- **Sounds.** PD reads `sounds.txt` and `soundenviron.txt` directly (PD 0x102334DC). This has no gameplay effect.
- **No other changes.** No PD patch record touches the D2Common loader (0x6FDAD000–0x6FDB0000), the flag
  0x6FDF145C / 0x6FDE9E20, or D2Client 0x6FAF1C30. PD has no `-txt` or `-direct` string.

### 2.3 Where each copy fits
| copy | where | what it is |
|---|---|---|
| `bin/pd2data.mpq` = `Live/pd2data.mpq` | shipped with the live DLLs | **live** |
| `excel_mpq/` | extracted `.txt` of the MPQ | = live (.txt) |
| `data.zip` (`tools/ias_sources.T()` default) | uploaded launcher archive, .txt dated 29 Aug–6 Sep, .bin 6 Sep | older snapshot |
| `excel_live/` | **byte-identical to data.zip** (Levels.txt is the zip's) | older snapshot. Its name is misleading |
| `ProjectD2/data/global/excel/` (loose) | 28 .txt files | D1/Tristram test branch (not live) |
| `excel/` | copies of the loose MonStats, MonStats2, Levels, Skills, States, SuperUniques | not live. No tool reads it |

## 3. The .bin decoder

**Files:**
- `tools/bindesc.py` recovers the real column descriptors.
  - It walks each of the 92 `#10849` call sites in D2Common and tracks the esp offset through pushes, `_chkstk`
    (0x6FD55AD0) and stdcall cleanup.
  - Skills and skilldesc share one huge frame, so it reuses `tools/txtdesc.py` at 0x6FDB3645 for them, as auras.md did.
  - `pd_isc_desc()` reads PD2's ItemStatCost descriptor from PD 0x10250DE0.
- `tools/bintables.py` reads any source (`zip`, `mpq`, `loose`) and decodes `.bin` records.

**Descriptor entry format:** `{name, type, param, offset, link}`. The type → storage mapping below was recovered from
the descriptors and confirmed against the data:

| type | storage | type | storage |
|---|---|---|---|
| 1 | char[param] | 13 / 12 | u8 index of a code list (PlayerClass, ElemTypes, MonMode…) |
| 2 | i32 | 14 / 15 | u16 index of a code list (ItemTypes) |
| 3 | u16 | 17 / 18 | row key (not stored) |
| 4, 6 | u8 | 19 | i32 link index (Properties) |
| 8 | u32 flag word | 20 | u16 link index (MonStats, Skills, ItemStatCost, States…) |
| 9 | raw 4-char code | 21 | u8 link index (PetType) |
| 10 | i32 code (item `code` = raw space-padded; other lists = index) | 22 | u16 string-table index |
| 11 | u32 item code (Runes) | 25 | i32 offset into `*code.bin` (calc byte code) |
| 23 | custom (CubeMain inputs) | 26 | bit `param&31` of the u32 at `offset + 4*(param>>5)` |

Record sizes match every file: `4 + count*recsize == len`. Examples: monstats 0x1A8, skills 0x23C, levels 0x220,
weapons/armor/misc 0x1A8, PD2 itemstatcost 0x150. The row count equals the `.txt` row count minus the `Expansion`
divider rows.

**Validation (DATA).** `tools/compare_sources.py` compiles each `.txt` cell the way the loader does:
- numbers, bits and strings directly;
- name → row index through the key column of the linked table;
- string keys → the D2Lang index, where string.tbl is 0–, patchstring 10000– and expansionstring 20000–, looked up in
  the order patch → expansion → base;
- an empty string key → index 5382 (`"dummy"`).

It then compares the result with the `.bin`. Results for mpq.bin vs mpq.txt, zip.bin vs zip.txt, and mpq.bin vs
zip.txt (matching cells / mismatching cells):

| table | mpq.bin vs zip.bin | mpq.bin ↔ mpq.txt | zip.bin ↔ zip.txt | mpq.bin vs zip.txt mismatches |
|---|---|---|---|---|
| automap | 1198 fields differ | 43650 / 0 | 43911 / 0 | 1198 |
| charstats | 2 (strings) | 497 / 0 | 497 / 0 | 2 |
| cubemain | 777 (+39 rows) | 217713 / 0 | 214086 / 0 | many (rows shifted) |
| itemstatcost | 439 (strings) | 27083 / 0 | 27069 / 0 | 425 |
| leveldefs | 4 | 8484 / 0 | 8484 / 0 | 4 |
| levels | 41 | 28482 / 0 | 28482 / 0 | 41 |
| lvlprest | 5 | 26611 / 0 | 26611 / 0 | 5 |
| lvltypes | 22 (−1 row) | 1564 / 0 | 1598 / 0 | 22 |
| misc | 522 (+10 rows) | 57695 / 6* | 55815 / 6* | 359 |
| monstats | 146 (strings) | 294799 / 1* | 294776 / 1* | 124 |
| pettype | 6 (strings) | 348 / 0 | 348 / 0 | 6 |
| skilldesc | 641 (strings) | 18130 / 0 | 18117 / 0 | 628 |
| superuniques | 1 (string) | 1081 / 0 | 1081 / 0 | 1 |
| uniqueprefix | 2 (strings) | 53 / 0 | 53 / 0 | 2 |
| weapons | 18 (strings) | 59972 / 0 | 59972 / 0 | 18 |
| skills, uniqueitems, states, monstats2, missiles, treasureclassex, armor, setitems, runes, magicprefix/suffix, monlvl, hireling, itemtypes, properties | byte-identical | 0–1* (TC 92, armor 108, itemtypes 239: model limits) | same | same |

`*` marks leftover mismatches that are identical for both sources, so they are limits of my compiler, not real
differences:
- TC entries are written with quotes.
- Shield `mindam`/`maxdam`.
- The ItemTypes `code` column.
- A cell containing a single space, which the game parses as −16 (see issue 7).
- Missiles `*16` comment cells.

`itemscode.bin` (calc byte code for weapons/armor/misc) differs because Misc's `calc1` changed. For example, map t11
compiles to `07 01 00` (= 1) in the MPQ and `07 02 00` (= 2) in the zip. The zip copy is also 1160 bytes against 631
in the MPQ, because it holds unreferenced code from an earlier compile.

The full per-table, per-column report is in **`adv/re/data_sources_diff.txt`**. Regenerate it with
`python3 tools/compare_sources.py > adv/re/data_sources_diff.txt`.

## 4. What differs, table by table (MPQ = live vs data.zip)

### 4.1 Real content changes (.txt differs, and the .bin follows it)
- **Misc** (DATA):
  - The MPQ has 10 extra rows:
    - maps `t57` Djinn's Domain (t5m), `t58` Na-Krul's Abyss (t5m) and `t3b` Kyovashad (t3m);
    - Corrupted-player ears `ivea`, `iveb`, `ived`, `iven`, `ivep`, `ives`, `ivez` (type `ubr`).
  - **Map tiers are reassigned.** 22 maps change `type`/`type2`, and `calc1` (the tooltip tier) changes with them.
    Zip → MPQ:
    - t11 Kurast t2m→**t1m**
    - t12 Arcane t1m→**t2m**
    - t13 Fortress t2m→**t3m**
    - t14 Library t3m→t2m
    - t15 Crypts t1m→t3m
    - t16 Cistern t3m→t2m
    - t17 Halls of Torture t1m→t2m
    - t21 Lava t3m→t1m
    - t23 Siege t1m→t3m
    - t24 Tomb t2m→t1m
    - t25 Sewers t1m→t2m
    - t27 Skovos t2m→t3m
    - t28 Demon Road t2m→t3m
    - t31 River of Blood t2m→t3m
    - t32 Throne t3m→t2m
    - t33 Spider t2m→t1m
    - t34 Ice t2m→t1m
    - t37 Pandemonium t3m→t2m
    - t38 Frozen Forest t3m→t2m
    - t3a Ashen Plains t3m→t2m
    - t51 Zhar calc1 3→1
    - t52 Hellcaves calc1 3→2
  - **Amulets and rings:** `amu`/`rin` have `hasinv`, `gemsockets` and `gemapplytype` 1 → **0**. Live jewelry has no
    socket capacity.
  - `t17` `invfile` changes from `invmap_newmap` to `invmap_torturehalls`.
- **Levels** (DATA): only row 193 **Halls of Torture Map** changes.
  - Zip: an unfinished placeholder (NumMon 0, no monsters, EntryFile M8L7).
  - MPQ:
    - MonLvl3Ex 87→**88**;
    - Unique counts N/NM 8–16, Hell 40–60;
    - 7 monsters: `sandleaper3TortureHalls`, `zealot1TortureHalls`, `sk_archer2TortureHalls`,
      `blunderbore1TortureHalls`, `vampire3TortureHalls`, `councilmember3TortureHalls`, `regurgitator1TortureHalls`;
    - rangedspawn 1, cpct1 40, object groups 46/47, lighting;
    - EntryFile M9L4.
  - leveldefs.bin for the same row: OffsetX 5000→4000, OffsetY 1500→2000.
- **LvlPrest / LvlTypes / AutoMap** (DATA): the Halls of Torture preset and its tiles.
  - LvlPrest: Files 2→3, `NewMap*.ds1` → `TortureHalls{,2,3}.ds1`, Dt1Mask 4194303→15.
  - LvlTypes: row 42 now uses `PD2assets/torturehalls/*.dt1` instead of the Dark Temple tiles. The zip has one extra
    blank row.
  - AutoMap: 29 rows are removed and 245 Type/Cel cells are re-pointed.
  - None of this affects combat or drops.
- **CubeMain** (DATA): the MPQ has 2341 rows, the zip 2302.
  - New in the MPQ:
    - corruption rows for T1–T3 Djinn's Domain, Na-Krul's Abyss and City of Ureh;
    - 12 "Lillith's Mirror" arrow/bolt rows;
    - 21 "map + ear <class>" rows.
  - Zip only: `Testing New map` and a duplicate `t2 > city of ureh`.
  - Map corruption: the `corruptnum` roll range is 1..1000 → **1..3000** for map inputs T1–T3.
  - Each outcome's window scales ×3. Example: `map-mon-deadlystrike` value 90 → 270, id 1001 → 3001. T4 rows stay at
    1001/100.
  - About 820 cells differ in total. See `data_sources_diff.txt` and maps.md §corruption, which already uses the MPQ.

> **Correction (audit):** maps.md §4 (corruption) did **not** use the MPQ: its table is the zip's 1..1000 version. `cube.md` §2.5 has the live numbers, and a correction note was added to maps.md.


### 4.2 String-table shifts only (.txt identical, .bin differs) (DATA)
- The MPQ's `patchstring.tbl` has 168 keys that differ from the zip's (103 new, 0 removed, 65 re-worded), so string
  indices ≥ ~11300 move.
- The affected `.bin` files differ **only** in type-22 (string index) columns:
  - `monstats.bin`: NameStr ×140, DescStr ×6. Example: `ubermephisto` 12660→12708, `boneprison1` 11880→11906.
  - `itemstatcost.bin`: descstrpos/neg/2 and dgrp* ×439. Example: `normal_damage_reduction` 12028→12059.
  - `skilldesc.bin`: str name/short/long and desc/dsc2/dsc3 texts ×641. Example: `critical strike` str short
    12296→12329.
  - `charstats.bin`: StrAllSkills for Druid and Assassin (12030→12061).
  - `pettype.bin`: name ×6.
  - `superuniques.bin`: Wisp Boss Name.
  - `uniqueprefix.bin`: Cold, Plague.
  - `weapons.bin`: namestr/spelldescstr of the tp* potions (11334→11336 …).
- The **numeric fields of these tables are identical.**
- Some re-worded strings: "Invader Amazon" → "Corrupted Amazon", "Gimli - Map Boss" → "Gimli the Summoned One",
  `ExtraGolem` "Additional Golem" → "to Maximum Golems", and `Attack1`/`Attack2` fixed.
- `data_ext/strings_all.json` already matches the MPQ (2853 of 2854 patch keys).

### 4.3 Identical in the zip and the MPQ (.txt and .bin) (DATA)
- Skills, SkillCalc, Missiles, MissCalc, MonStats (numbers), MonStats2, MonLvl, MonProp, MonUMod, SuperUniques
  (numbers), Hireling.
- ItemStatCost (numbers), Properties, UniqueItems, SetItems, Sets, Runes, MagicPrefix, MagicSuffix, Gems, Weapons
  (numbers), Armor, ItemTypes, ItemRatio, TreasureClassEx, Experience, DifficultyLevels, States, Shrines, and every
  `*code.bin` except `itemscode.bin`.

### 4.4 Loose folder vs MPQ (not live; for reference) (DATA)
- **Identical** to both zip and MPQ: Armor, CharStats, DifficultyLevels, Gems, Hireling, ItemStatCost, ItemTypes,
  MagicPrefix/Suffix, MonLvl, MonMode, MonType, MonUMod, PlrMode, Properties, Runes, SetItems, Sets, Weapons.
- **Levels:** adds ids 202–211 (Poisoned Well Map, Dark Temple Map, D1 Hell Preset/Depths, D1 Cathedral 1–2,
  Labyrinth 1–2, Caves 1–2).
  - It rewrites the spawn lists of 19 levels. For example, Act 1 Wilderness 1 `mon1` changes from `zombie1` to
    `d1r_000`.
  - Row 193 stays at the MPQ's finished version.
- **Misc:** the zip's map tiers, plus `pwl`, `dtm` and `dhm`. It lacks t57, t58, t3b and the ears. amu/rin keep
  sockets = 1.
- **MonStats / MonStats2:** 223 / 219 extra D1 rows. `griswold` changes Code GZ→BV and sounds zombie→Butcher.
- **SuperUniques:** 60 cells change. The Act bosses are re-classed to D1 monsters, e.g. Bishibosh →
  `D1RName118`/`d1r_118`.
- **Skills / Skilldesc / States / UniqueItems:** rows are only appended (11 skills incl. Corpse Instability, 1 desc,
  7 states, 1 unique). Existing rows are unchanged, apart from the quoting of calc cells: an Excel re-save turns
  `"min(14,2+(lvl/par2))"` into `min(14,…)`.

## 5. Impact on our outputs

Method: `tools/source_impact.py` builds every output twice, once from its current source and once from the live MPQ,
into a scratch directory, then diffs the JSON. The results are in **`adv/re/data_sources_impact.txt`**.
- Every current file except `site/ias-data.json` rebuilds byte-for-byte from its current source.
- `ias-data.json` has since been post-processed by `extract_fast.py`, `add_multi.py`, `add_oskills.py` and
  `add_uar.py`. All four read only tables that are identical in the zip and the MPQ.

| output | built from | changes when rebuilt from the live MPQ |
|---|---|---|
| `site/ias-data.json` (`tools/extract_ias_data.py` + the add_*/extract_fast chain) | zip Weapons, Skills, ItemTypes, UniqueItems, SetItems, Runes, Sets | **none** (0 differences). Weapon speed, skill speed params and IAS mods are all live |
| `site/hitcalc-data.json` (`tools/extract_hitcalc_data.py`) | zip MonStats, MonLvl, Levels, SuperUniques, MonUMod, MonType, DifficultyLevels + zip `patchstring.tbl` | 24 differences. **Monster display names**: 21 of them, e.g. `mon[990..998]` "Invader X" → "Corrupted X", `mon[999]` "Kyovoshad Boss" → "The Tainted Hive", `mon[1001]` "Nakrul Boss" → "Na-Krul", `mon[1002]` "Djinn Light Boss" → "Djinn of a Thousand Blades". **Area 185 (Halls of Torture Map)**: Hell level `ex[2]` 87→88 and the spawn lists `sn`/`sh` 0 → 7 monsters. All monster stats (AR, defense, resistances, HP, levels) are unchanged |
| `adv/re/drops-data.json` (`adv/re/extract_drops.py`) | zip (/tmp/dz) | 44 differences. `items` 820→830 (t57, t58, t3b and 7 ears appended). **22 map item types** change (fields 2/3 = ItemTypes ids 106–112, i.e. the tier reassignment in §4.1). `amu`/`rin` gemsockets 1→0. TC "Map Tier 3" entry `t3b` now resolves (`'?'`→`'i'`). TC probabilities, uniques, sets, ItemRatio and the monster/level blocks are unchanged. `mapTiers` is hard-coded from PD's lists and already matched the live data, so the map items and `mapTiers` disagreed in the zip build |
| `adv/re/monsters.json` (`adv/re/extract_monsters.py`, source `excel_live` = zip) | zip | 4 differences, all for level 193 Halls of Torture Map: `MonLvlEx[2]` 87→88, `mon`/`nmon`/`umon` 0 → 7 entries. Everything else is identical |
| `adv/re/skilldmg-data.json` (`adv/re/extract_skilldmg.py`, default = **loose** folder) | loose Skills/Skilldesc + zip Missiles | The 11 non-live D1/Corpse Instability skills (ids 603–613) disappear from `skillByName`. Two desc calc strings get the game's quotes back (`"min(2+(lvl/10), 6)" - 1`), which skilldmg.js already reads as brackets (skilldmg.js:51). No player skill value changes |
| `adv/re/minions.json` (`adv/re/extract_minions.py`) | zip skills.bin, skillscode.bin, monstats.bin, monlvl.bin, hireling.bin | **none** (0 differences). The bins are identical, or monstats differs only in NameStr, which minions does not use |
| `adv/engine/engine-data.json` (`adv/engine/extract_engine_data.py`) | zip | 50 differences. 10 new items (t57, t58, t3b, ivea…ives), and `t`/`t2` for the 22 re-tiered maps. Skills, ISC, properties, sets, uniques and gems are unchanged |
| `adv/engine/engine-data-local.json` (not asked, but used by `tools/build_adv.py`) | **loose** folder (verified: rebuilt byte-identical) | Built for the uploaded saves' branch. Against live it has 11 extra skills, pwl/dtm/dhm and 1 extra unique, and it lacks t57, t58, t3b and the ears. Keep it only if the site is meant to decode those test saves |

What matters most, re-checked (DATA):
- Skill speed and damage params: unchanged.
- Monster stats: unchanged, only names.
- Weapon speed and damage: unchanged.
- ItemStatCost numbers: unchanged. Only the description string ids move, and the PD s12 save columns are unchanged.
- Uniques: unchanged.
- Treasure classes: unchanged. But in the zip, `t3b` did not resolve.
- Levels: one row.
- Map tiers: changed (22 maps).
- Jewelry sockets: changed.

Other research that used the zip:
- drops.md §2 lists `t3b` among "unknown names dropped" in TC resolution. That is a zip artefact; in live, `t3b` is a
  valid item.
- `harness/pd2_skilldesc.bin` is the zip copy. It differs from live only in string ids, so the skilldmg harness results
  still stand.
- maps.md, items (`extract_items.py`) and experience/shrines already used the MPQ, or tables that are identical.

> **Correction (audit):** true for maps.md §1–3 and items, but not for maps.md §4 (corruption table from the zip, see above). Also, `drops-data.json` has since been rebuilt (26 Sep 00:12) and now carries the live map items.


## 6. Source switch and rebuild commands

**`tools/ias_sources.py`** now takes a source. The default is still `zip`, so every existing caller is unchanged.
```
T(name)                     # data.zip (default, unchanged)
T(name, source='mpq')       # live bin/pd2data.mpq
PD2_DATA_SOURCE=mpq  ...    # whole run; 'zip' | 'mpq' | 'loose' (loose falls back to mpq for missing files)
raw(name, source, sub=...)  # bytes of any excel .txt/.bin (or e.g. patchstring.tbl with sub='data/local/lng/eng/')
```
**Extractors.** Four extractors now read through `ias_sources.raw`: `extract_ias_data.py`, `extract_hitcalc_data.py`,
`extract_drops.py` and `extract_minions.py`.
- With `PD2_DATA_SOURCE` unset they read the zip exactly as before; `extract_drops.py` still reads `/tmp/dz`.
- `extract_drops.py` and `extract_minions.py` also take an optional output path as `argv[1]`.
- The other three extractors already accepted an excel directory, so `excel_mpq/` works with them.

**Rebuild commands.** I did not run these; the outputs were only built in a scratch directory:
```
cd /home/claude/pd2re
PD2_DATA_SOURCE=mpq python3 tools/extract_hitcalc_data.py site/hitcalc-data.json
python3 tools/build_hit.py                                   # re-embed into site/hit-chance.html
PD2_DATA_SOURCE=mpq python3 adv/re/extract_drops.py           # -> adv/re/drops-data.json
python3 adv/re/extract_monsters.py --src /home/claude/pd2re/excel_mpq                      # -> adv/re/monsters.json
python3 adv/re/extract_skilldmg.py /home/claude/pd2re/excel_mpq adv/re/skilldmg-data.json
python3 adv/engine/extract_engine_data.py adv/engine/engine-data.json /home/claude/pd2re/excel_mpq
# no change expected (0 differences), rebuild only for provenance:
PD2_DATA_SOURCE=mpq python3 adv/re/extract_minions.py         # -> adv/re/minions.json
# site/ias-data.json: no change; if ever rebuilt, run the whole chain with the same env:
#   PD2_DATA_SOURCE=mpq python3 tools/extract_ias_data.py site/ias-data.json   (then extract_fast/add_multi/add_oskills/add_uar, build_ias)
# then re-embed pages that include these files (tools/build_adv.py, tools/build_pages.py) and re-run the tests (adv/re/test_drops.js)
# check: python3 tools/source_impact.py /tmp/pd2_source_impact  -> every "zip -> live mpq" block should now read 0 against the rebuilt files
```
- **`engine-data-local.json`.** Rebuild it from `excel_mpq` only if the advanced page should show live data rather
  than the test branch that the uploaded saves come from:
  `python3 adv/engine/extract_engine_data.py adv/engine/engine-data-local.json /home/claude/pd2re/excel_mpq`
- **Save decoding.** The save decoder (`adv/extract_save_data.py`) needs the loose branch for `dhm`/`pwl`/`dtm`.

## 7. Issues and quirks

| # | issue | status | verdict |
|---|---|---|---|
| 1 | `tools/ias_sources.T()` and the extractors default to `data.zip`, an older snapshot. Misc, Levels, CubeMain, AutoMap, LvlPrest, LvlTypes and patchstring.tbl are stale | DATA + READ | tooling bug. The switch is added and the default is kept |
| 2 | `excel_live/` is named "live" and AGENT_CONTEXT says `T()` "picks the right one", but excel_live is byte-identical to data.zip (e.g. Levels.txt) and T() always read the zip | DATA | tooling/documentation bug |
| 3 | The zip's own TreasureClassEx refers to `t3b`, which its Misc lacks. drops.md §2 lists `t3b` among "unknown names dropped", but in live `t3b` (Kyovashad map, t3m) resolves | DATA | stale-data artefact, not a game bug |
| 4 | Map tiers in drops-data/engine-data (from the zip) contradict PD's hard-coded T1–T3 level sets, which match the MPQ | DATA + READ | stale-data bug in our outputs |
| 5 | hitcalc-data monster names come from the zip `patchstring.tbl` (e.g. "Invader Amazon" instead of the live "Corrupted Amazon") | DATA | stale-data bug in our outputs |
| 6 | `extract_skilldmg.py` defaults to the loose folder (D1 test branch). No player value changes, but it adds 11 non-live skills | DATA | tooling issue |
| 7 | The loader parses a cell that holds a single space as −16 (a u8 field becomes 240, a bit field becomes 1). Live effects: Misc `qey` TMogMin/Max = 240 (inert without TMogType), MonStats `willowisptotem` rangedtype = 1, Skills `MaggotEgg` scroll = 1, and the placeholder UniqueItems `Nethercrow` carry1 = 1 (row has no item code) | DATA (the .bin holds these values; the parser itself was not traced) | intended but surprising. Harmless except possibly willowisptotem AI; unclear |
| 8 | The zip's `itemscode.bin` (1160 bytes) carries unreferenced byte code from an earlier compile; the MPQ's is 631 bytes | DATA | harmless |
| 9 | The uploaded saves and `engine-data-local.json` belong to the loose D1/Corpse Instability branch, not to the Live DLL + MPQ pair | DATA + READ | unclear. Keep the two data sets separate |
| 10 | `pd2maps.mpq`, `pd2assets.mpq` and `pd2monchars.mpq` are opened at the same priority but were not uploaded. If one of them contained `data\global\excel\*` files, the MPQ order would decide which copy is used | READ | unverified. pd2data.mpq is the only PD2 MPQ we have, and PD's hard-coded lists agree with it |
| 11 | Whether PD2's launcher passes `-txt` or `-direct` is not visible in our binaries (Game.exe/launcher not uploaded). `-txt` cannot matter (MPQ bin == MPQ txt). `-direct` would only matter if loose files sat in the live install, and PD's hard-coded data contradicts the loose folder | READ + DATA | unlikely to matter |
