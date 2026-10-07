# PD2 map system: map items, map mods, density, currency and map bosses

Scope: the PD2 client install (1.13c D2Game/D2Common + ProjectDiablo.dll, "PD", base 0x10000000). Realm servers may run different server code.

Status labels (AGENT_CONTEXT.md):
- **VERIFIED**: run natively in the harness and matched a model.
- **READ**: from the disassembly only.
- **DATA**: from the tables.

Files:
- `adv/re/maps.js`: data plus the mod roller, stat router, applier and zone model (UMD, `window.PD2Maps`).
- `harness/mapapply.c`: native run of the map-mod applier.
- `harness/mapaffix.c`: native run of the rare-count and magic-force hooks.

Related notes reused, not redone: `drops.md` (map drops, `map_glob_drop*`, map TCs), `dmg_D_incoming.md` §5 (map/zone damage mods), `uber_review.md` (uber bosses, `uber_difficulty`).

> **Data source warning (important).** The tables in `data.zip` (used by `tools/ias_sources.T()` and `excel_live/`) are **not** the live data for these tables:
> - `Misc.txt`, `Levels.txt` and `CubeMain.txt` differ from the ones inside `bin/pd2data.mpq`, which ships next to the live DLLs.
> - The MPQ copy is newer: it adds maps t57/t58/t3b and the Djinn/Na-Krul corruptions.
> - It also **reassigns map tiers**. For example, Kurast is `t1m` in the MPQ but `t2m` in the zip, and Arcane is `t2m` in the MPQ but `t1m` in the zip.
> - Evidence that PD's code follows the MPQ: its built-in "T1 levels" set 0x104E30B4 = {143,145,146,148,151,155,158,169} matches the MPQ tiers exactly.
> - This report reads the MPQ (`bin/pd2data.mpq`, via `tools/mpq.py`).
> - The two copies agree on `MagicPrefix`, `MagicSuffix`, `ItemTypes`, `ItemStatCost`, `Properties`, `MonStats`, `SuperUniques`, `UniqueItems` and `TreasureClassEx`.
> - Also: game ItemTypes ids skip the `Expansion` row. So `map` = 105, `t1m`..`t4m` = 106..109, `t1me`..`t4me` = 110..113, `upmp` = 124, `t5m` = 211, `t5me` = 212 (checked against `itemtypes.bin`).

---------------------------------------------------------------------------------------------------------------------

## 1. Map items

### 1.1 Bases and tiers (DATA, MPQ Misc.txt)
- Every usable map is a Misc item with `pSpell` = 12.
  - D2Game's item-use table 0x6FD27238 is patched so that entry 12 points to **PD 0x102D4820**, the map-open handler (patch at PD 0x102BE30A) (READ).
- **Tier = item type** `t1m`/`t2m`/`t3m`/`t4m`/`t5m`.
  - The `type2` column (`t1me`..`t5me`, "exc") is used only by cube recipes. All `*me` types have `Equiv1 = map`.
- **ItemTypes chain:** `t3m` → `t2m` → `t1m` → `map`. `t4m` and `t5m` have no parent (not even `map`).
- `calc1` holds the tooltip "Tier:" number. For T1–T3 it equals the tier. For unique T5 maps it is the sub-tier: Zhar 1, Hellcaves 2, Fallen Gardens 2, Imperial 3, Void 3, Ureh 3, Djinn 2, Na-Krul 1.
  - The ubers use 666.
  - Code never reads it for gameplay; no reader was found.

**Area/level id = the base's `len` formula.**
- The handler evaluates the Misc `len` column with D2Common #10428, the item-formula evaluator, at 0x102D48E1 (READ).
- That value is a constant level id per base.
- So neither the area nor the tier is stored on the individual item. Both come from the base.

**What is stored on the item:**

| Stored | What it holds |
|---|---|
| the stat list | the mods |
| stat 185 `uber_difficulty` | uber maps only |
| stat 360 `corrupted` | outcome id |
| stat 361 `corruptor` | the 1..3000 corruption roll (live MPQ), then 3001 (T1–T3) or 1001 (T4) once an outcome applies; see §4 |
| `heroic` | set by a Standard of Heroes |
| stat 493 `map_glob_skirmish_mode` | fortified |



**Monster level** (DATA + READ):
- Base: Levels `MonLvl3Ex` (Hell expansion) of that level. T1 = 87, T2 = 88, T3 = 89, T4 = 92, T5 = 87–89.
- Plus `map_glob_arealevel`, added by the PD monster-level hook 0x10268D00 for levels 137–201.
- Elite bonuses are in `drops.md`: champion +2, unique/minion +3.

| code | name | type | calc1 | level | mlvl | MonDen(H) | MonUMin/Max(H) | drops |
|---|---|---|---|---|---|---|---|---|
| ubtm | Uber Trist Map | ubr | 666 | 185 PD2 Pandemonium Finale | 83 | 0 | 3/3 | yes |
| dcma | Uber Map | ubr | 666 | 137 UberDiabloLvl | 90 | 0 | 0/0 | yes |
| t11 | Kurast Map | t1m | 1 | 143 Kurast | 87 | 2062 | 40/60 | yes |
| t12 | Arcane Map | t2m | 2 | 142 Arcane | 88 | 1980 | 40/60 | yes |
| t13 | Fortress Map | t3m | 3 | 149 Fortress Map | 89 | 1980 | 40/60 | yes |
| t21 | Lava Map | t1m | 1 | 145 Lava | 87 | 1980 | 40/60 | yes |
| t22 | Jungle Map | t1m | 1 | 148 Jungle Map | 87 | 1980 | 40/60 | yes |
| t23 | Siege Map | t3m | 3 | 139 Siege Map | 89 | 1980 | 40/60 | yes |
| t24 | Tomb Map | t1m | 1 | 151 Tomb Map | 87 | 2860 | 40/60 | yes |
| t25 | Sewer Map | t2m | 2 | 141 Sewers | 88 | 2145 | 40/60 | yes |
| t31 | River Of Blood Map | t3m | 3 | 144 River Of Blood | 89 | 1980 | 40/60 | yes |
| t32 | Throne Map | t2m | 2 | 150 Throne Map | 88 | 1815 | 40/60 | yes |
| t33 | Spider Map | t1m | 1 | 158 Spider Map | 87 | 2200 | 40/60 | yes |
| t34 | Ice Map | t1m | 1 | 146 Ice Map | 87 | 1980 | 40/60 | yes |
| t35 | Graveyard Map | t3m | 3 | 154 Graveyard Map | 89 | 1050 | 40/60 | yes |
| t36 | Caldeum Map | t1m | 1 | 155 Caldeum Map | 87 | 1980 | 40/60 | yes |
| t37 | Pandemonium Map | t2m | 2 | 156 Pandemonium Map | 88 | 1350 | 40/60 | yes |
| t41 | Monastery Map | ubr | – | 152 Monastery Map | 92 | 2640 | 50/70 | no |
| t39 | Desert Map | t3m | 3 | 147 Desert Map | 89 | 1900 | 40/60 | yes |
| rtma | Rathma Map | ubr | 666 | 161 Necropolis Jungle | 92 | 1600 | 40/60 | no |
| t38 | Frozen Forest Map | t2m | 2 | 160 Frozen Forest Map | 88 | 1650 | 40/60 | yes |
| t42 | Torment Map | t4m | – | 164 Realm of Terror | 92 | 2475 | 40/60 | yes |
| t14 | Library Map | t2m | 2 | 167 Library Map | 88 | 2310 | 40/60 | yes |
| uba | Uber Ancients Map | ubr | – | 168 Uber Ancients | 90 | 0 | 0/0 | yes |
| t26 | Westmarch Map | t1m | 1 | 169 Westmarch Map | 87 | 2062 | 40/60 | yes |
| t15 | Crypts Map | t3m | 3 | 170 Crypts Map | 89 | 1980 | 40/60 | yes |
| t43 | Sanctuary of Sin Map | t4m | – | 171 Sanctuary Of Sin Map | 92 | 2475 | 40/60 | yes |
| t16 | Ruined Cistern Map | t2m | 2 | 174 Ruined Cistern Map | 88 | 1980 | 40/60 | yes |
| t3a | Ashes Map | t2m | 2 | 175 Ashen Plains Map | 88 | 1950 | 40/60 | yes |
| t51 | Zhar Map | t5m | 1 | 176 Zhar Library | 87 | 1980 | 40/60 | no |
| t52 | Hellcaves Map | t5m | 2 | 181 Hellcaves | 88 | 1980 | 40/60 | no |
| t53 | Fallen Gardens Map | t5m | 2 | 183 Fallen Gardens | 88 | 1650 | 40/60 | no |
| t44 | KanemithMap | t4m | – | 186 Kanemith Outside | 92 | 2475 | 40/60 | yes |
| t28 | Skovos Stronghold Map | t3m | 3 | 190 Skovos Stronghold Map | 89 | 2250 | 40/60 | yes |
| luca | Lucion Map | ubr | 666 | 188 Lucion Arena | 89 | 1980 | 40/60 | no |
| t27 | Demon Road Map | t3m | 3 | 191 Demon Road Map | 89 | 2000 | 40/60 | yes |
| t54 | Imperial Palace Map | t5m | 3 | 192 Imperial Palace Map | 89 | 1750 | 40/60 | no |
| t17 | Halls of Torture Map | t2m | 2 | 193 Halls of Torture Map | 88 | 2200 | 40/60 | yes |
| t55 | Outer Void Map | t5m | 3 | 180 Outer Void Map | 89 | 2000 | 40/60 | no |
| t56 | Ureh City Map | t5m | 3 | 195 Ureh City Map | 89 | 1500 | 40/60 | yes |
| t57 | Djinns Domain Map | t5m | 2 | 197 Djinns Domain Map | 88 | 1400 | 40/55 | yes |
| t58 | Na-Krul's Abyss Map | t5m | 1 | 200 Na-Krul's Abyss Map | 87 | 2250 | 0/0 | yes |
| t3b | Kyovoshad Map | t3m | 3 | 201 Kyovoshad Map | 89 | 1850 | 40/60 | yes |

"drops" = `spawnable`.
- **Unique maps** (t51–t56) are unspawnable except Ureh/Djinn/Na-Krul. They come from the unique-map special drop (`drops.md`) and from corruption (§4).
- **Dropped maps are always white.** PD forces quality 2 for ItemTypes `map` (0x102C8A67; `drops.md`, READ). Magic and rare come only from the cube.

### 1.2 Opening a map (PD 0x102D4820, READ)
The handler is called as `(ecx=game, edx=player, item, 0, x, y, …)`.

**Gates, in order.** The map fails with message 0x13 if any of these is not met:
1. `game+0x1DF4` (the game's map/event type) must be 0, or 0x28 (then only levels 136/185 may be opened). So **one map per game**.
2. Levels 152, 153 and 255 are refused outright (0x102D0FE0). 152/153 is the Monastery (t41, unspawnable).
3. **Hell only**: difficulty 0 and 1 are refused.
4. Character level ≥ 80, unless the level is in exempt set 0x104E352C = {136,137,161,162,163,168,185,188,189} (ubers).
5. Hell quest flag 39 (Rite of Passage) must be complete (#10174 on the Hell quest record).
6. The player must stand in **Harrogath (109)**. PvP maps (157/159/166) are opened from the Rogue Encampment (1). The level must be a town (0x102CEA40).

**Uber arenas** (set 0x104E3568 = {137,161,168,188}):
- Must be used within 150 units of (5074,5114), the blood fountain (0x102CFAF0).
- `uber_difficulty` (stat 185) is copied to `game+0x2629`.
- 0x102DB680 moves every eligible player near the fountain into the arena.

**Normal maps.** The item's full stat list is split by ItemStatCost `Divide` (0x102D4E70):

| Divide | Destination | List |
|---|---|---|
| < 1000 | monster stat `Divide` | monster list, game+0x1DF8 (count +0x21F8) |
| 1024 | monster list with the stat id unchanged | only `map_defense` 369 |
| 1000–1999 | player stat `Divide`−1000 | player list, game+0x21FC (count +0x25FA) |
| 2000–2999 | global key `Divide`−2000 | global list, game+0x2630 {u16 layer, u16 key, i32 value}, count +0x26F0 |
| 3000 | physical-as-extra | kept aside and appended to the end of the monster list |
| 0 | only stat 500 `map_force_event` | sets the "force a random event" argument of the portal call |

After the split:
- The first 24 stats are sent to every client in packet 0x55 (tooltip/overlay).
- The zone builder 0x102DC520 runs (§3).
- The portal is created (0x102DB2F0; its object class is Misc `missiletype`, 584/585/622, or 60 if blank).
- Multi-level maps also rebuild their extra levels, e.g. Zhar 176 → 177–179 and 197 → 198/199.

---------------------------------------------------------------------------------------------------------------------

## 2. Map mods

### 2.1 Where they come from (DATA)
There is no PD-specific affix table. Map mods are ordinary **MagicPrefix/MagicSuffix rows** with `itype` in t1m..t4m:
- 242 rows in all, rows 671–977 of each table.
- The Properties are `map-*`, and the stats are ItemStatCost 186–508.

Each family exists twice:
- **Rare-allowed rows** (`rare` = 1). Frequency 1–4 (prefixes 1–2, suffixes 2–4).
- **Magic-only rows** (`rare` blank). Frequency 6–18. Values are about 1.5× stronger, and density/MF/XP is about 2×.

Each row lists two tiers, e.g. `t1m,t2m` / `t2m,t3m` / `t3m,t4m`. Because of the ItemTypes chain (§1.1):
- **A t3m map can roll every tier's rows**: t1-, t2- and t3-grade versions of a family, in equal weight.
- A t2m map rolls t1- and t2-grade rows.
- t4m rolls only the `…,t4m` (t3-grade) rows.
- t5m (unique) maps roll nothing from the tables. Their mods are fixed in UniqueItems.

Result: rare t3 maps get only about one third of their affixes at t3 strength. Model run, `PD2Maps.rollMap`, 20,000 rare Fortress (t3m) maps:
- t1-grade 38,538
- t2-grade 37,874
- t3-grade 38,550
- multi-tier rows 5,038

Unique maps and corruption outcomes also use `map-*` properties outside the affix tables (§4, §5).

### 2.2 How many mods (READ + VERIFIED)

| Quality | Rule | Status |
|---|---|---|
| white | 0 mods. All dropped maps are white. | READ |
| magic | stock picker D2Game 0x6FC344D0 via PD wrapper **0x102D7170**. For ItemTypes `map` the wrapper sets "force" on **both** the prefix call and the suffix call, so a magic map always gets **1 prefix + 1 suffix**. Stock items: prefix 50 %, suffix 50 %, at least one. | force flag **VERIFIED** (mapaffix.c 30000/30000) |
| rare | stock loop D2Game 0x6FC35240 with PD count **0x102D57C0**, patched in at 0x6FC352BF. Loop limits are max 3 prefixes and 3 suffixes. | count **VERIFIED** (mapaffix.c 30000/30000) |
| unique (T5) | fixed UniqueItems mods | DATA |
| corrupted | adds 1 outcome (§4) | DATA+READ |

The same wrapper also forces both affixes for other magic items:
- charms (type 13) at ilvl ≥ 90
- rings, amulets and jewels at ilvl ≥ 85
- everything else at ilvl ≥ 65

**Rare affix count:**
```
count = 4                                   if jewel (type 58)
      = 6                                   if ilvl >= 85
      = 5 + r % 2                           if ilvl >= 65
      = 4 + r % 3                           if ilvl >= 45
      = 3 + r % 4                           otherwise          (r = PD rand 0x102C5D10 on the item seed)
stock (replaced): [3,4,4,5,5,5,6,6][rand & 7]
```

**Maps are dropped at ilvl = monster level (87–92), so they roll 6 affix attempts.**
- Cube-made rares use the cube's output ilvl. That path was not traced.
- Each round of the rare loop picks prefix or suffix with `rand & 1`.
- A failed pick closes that side and the round is retried.
- The weighted pick cannot fail while a candidate exists (§2.3), and every map tier has more than 3 prefix and 3 suffix groups. So **a rare map at ilvl ≥ 85 always gets 6 affixes: 3 prefixes and 3 suffixes**. The model gives 6.00 on 20,000 rolls each of rare t11/t13/t42 at ilvl 88; `items.js` agrees.



### 2.3 Roll rules of the stock picker D2Game 0x6FC344D0 (READ)
Filters, applied in order:
1. `spawnable`.
2. Frequency > 0.
3. `itype` must match (with Equiv) and no `etype` may match (0x6FC2B3C0).
4. The `rare` column must be set when the item is rare (quality 6, 8 or 9).
5. The affix's `group` must not already be on the item. The check covers the 3 prefix and 3 suffix slots (0x6FC331E0).
6. The affix `level` must be ≤ alvl, and `maxlevel` (when not blank) ≥ alvl. All map affixes are level 1 with no max, so item level never matters for maps.

**Choice:**
```
weight = frequency;  r = rand % (total + 1);  walk the candidates subtracting weights; take the first with r < 0
```
- When `r == total` the walk ends without a hit, and `edi` still points at the last candidate, which is taken (D2Game 0x6FC3481E–0x6FC34853). So the pick never fails while a candidate exists, and the last candidate's weight is effectively `w + 1`. This is VERIFIED natively in `items.md` §2.4 (`harness/itemaffix_probe.js`; the "picks nothing" mutation gives 542 mismatches).



**Values:** `min + rand(max − min + 1)`, or `min` when max ≤ min.
- The second stat of a property (func 3/8) reuses the same rolled value. Examples: `map-play-mag-gold%` gives MF = GF, and `map-mon-att-pierce` gives AR% = pierce %.

**Group namespace:** prefix and suffix groups share one namespace. These families therefore exclude each other:
- 157: Splashing ↔ of Sturdiness/Stability/Vigor
- 158: Strong/Mighty/Formidable ↔ of Crimson
- 164: Sparking/Shocking/Thunderous ↔ of Warding/Guarding/Preservation
- 165: Chilling/Boreal/Glacial ↔ of Cracking/Crumbling/Disintegration
- 166: Smoking/Molten/Volcanic ↔ of Hindrance/Aversion/Permeability

(DATA + READ)

### 2.3a Affix families (DATA; from MPQ MagicPrefix/MagicSuffix; 'rare' rows are the only ones rare maps can roll)

| side | group | names | rare-allowed rows (itypes, frequency) values | magic-only rows values |
|---|---|---|---|---|
| prefix | 150 | Champion's | (t1m,t2m, f2) glob-monsterrarity 30..40<br>(t2m,t3m, f2) glob-monsterrarity 40..50<br>(t3m,t4m, f2) glob-monsterrarity 50..60 | (t1m,t2m, f12) glob-monsterrarity 45..60<br>(t2m,t3m, f12) glob-monsterrarity 60..75<br>(t3m,t4m, f12) glob-monsterrarity 75..90 |
| prefix | 151 | Flaming/Ember/Smoldering | (t1m,t2m, f2) mon-extra-fire 5..10; glob-density 13..18; play-addxp 3..4<br>(t2m,t3m, f2) mon-extra-fire 10..15; glob-density 18..23; play-addxp 4..5<br>(t3m,t4m, f2) mon-extra-fire 15..20; glob-density 23..25; play-addxp 5..6 | (t1m,t2m, f12) mon-extra-fire 8..15; glob-density 26..36; play-addxp 6..8<br>(t2m,t3m, f12) mon-extra-fire 15..23; glob-density 36..46; play-addxp 8..10<br>(t3m,t4m, f12) mon-extra-fire 23..30; glob-density 46..50; play-addxp 10..12 |
| prefix | 152 | Hibernal/Snowflake/Shivering | (t1m,t2m, f2) mon-extra-cold 5..10; glob-density 13..15; play-mag-gold% 15..20<br>(t2m,t3m, f2) mon-extra-cold 10..15; glob-density 15..18; play-mag-gold% 25..30<br>(t3m,t4m, f2) mon-extra-cold 15..20; glob-density 18..20; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-extra-cold 8..15; glob-density 26..30; play-mag-gold% 30..40<br>(t2m,t3m, f12) mon-extra-cold 15..23; glob-density 30..36; play-mag-gold% 50..60<br>(t3m,t4m, f12) mon-extra-cold 23..30; glob-density 36..40; play-mag-gold% 70..80 |
| prefix | 153 | Static/Glowing/Arcing | (t1m,t2m, f2) mon-extra-ltng 5..10; glob-density 13..20; play-addxp 3..4<br>(t2m,t3m, f2) mon-extra-ltng 10..15; glob-density 18..23; play-addxp 4..5<br>(t3m,t4m, f2) mon-extra-ltng 15..20; glob-density 23..25; play-addxp 5..6 | (t1m,t2m, f12) mon-extra-ltng 8..15; glob-density 26..40; play-addxp 6..8<br>(t2m,t3m, f12) mon-extra-ltng 15..23; glob-density 36..46; play-addxp 8..10<br>(t3m,t4m, f12) mon-extra-ltng 23..30; glob-density 46..50; play-addxp 10..12 |
| prefix | 154 | Toxic/Pestilent/Septic | (t1m,t2m, f2) mon-extra-pois 5..10; glob-density 15..20; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-extra-pois 10..15; glob-density 20..25; play-mag-gold% 15..20<br>(t3m,t4m, f2) mon-extra-pois 15..20; glob-density 25..30; play-mag-gold% 25..30 | (t1m,t2m, f12) mon-extra-pois 8..15; glob-density 30..40; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-extra-pois 15..23; glob-density 40..50; play-mag-gold% 30..40<br>(t3m,t4m, f12) mon-extra-pois 23..30; glob-density 50..60; play-mag-gold% 50..60 |
| prefix | 156 | Silver/Shining/Opulent | (t1m,t2m, f2) mon-att-pierce 30..50; glob-density 8..10; play-mag-gold% 15..20<br>(t2m,t3m, f2) mon-att-pierce 50..70; glob-density 15..18; play-mag-gold% 30..35<br>(t3m,t4m, f2) mon-att-pierce 70..90; glob-density 18..20; play-mag-gold% 45..50 | (t1m,t2m, f12) mon-att-pierce 45..75; glob-density 16..20; play-mag-gold% 30..40<br>(t2m,t3m, f12) mon-att-pierce 75..105; glob-density 30..36; play-mag-gold% 60..70<br>(t3m,t4m, f12) mon-att-pierce 105..135; glob-density 36..40; play-mag-gold% 90..100 |
| prefix | 305 | Fast/Speedy/Ludicrous | (t1m,t2m, f2) mon-att-cast-speed 40..60; glob-density 15..20; play-addxp 3..4<br>(t2m,t3m, f2) mon-att-cast-speed 60..80; glob-density 20..25; play-addxp 4..5<br>(t3m,t4m, f2) mon-att-cast-speed 80..100; glob-density 25..30; play-addxp 5..6 | (t1m,t2m, f12) mon-att-cast-speed 60..90; glob-density 30..40; play-addxp 6..8<br>(t2m,t3m, f12) mon-att-cast-speed 90..120; glob-density 40..50; play-addxp 8..10<br>(t3m,t4m, f12) mon-att-cast-speed 120..150; glob-density 50..60; play-addxp 10..12 |
| prefix | 311 | Large/Huge/Colossal | (t1m,t2m, f1) mon-hp% 5..10; glob-density 15..20; play-addxp 6..8<br>(t2m,t3m, f1) mon-hp% 10..15; glob-density 20..25; play-addxp 8..10<br>(t3m,t4m, f1) mon-hp% 15..20; glob-density 25..30; play-addxp 10..12 | (t1m,t2m, f6) mon-hp% 10..20; glob-density 30..40; play-addxp 12..16<br>(t2m,t3m, f6) mon-hp% 20..30; glob-density 40..50; play-addxp 16..20<br>(t3m,t4m, f6) mon-hp% 30..40; glob-density 50..60; play-addxp 20..24 |
| prefix | 158 | Strong/Mighty/Formidable | (t1m,t2m, f2) mon-ed% 10..15; play-addxp 4..5; play-mag-gold% 10..15<br>(t2m,t3m, f2) mon-ed% 15..20; play-addxp 5..6; play-mag-gold% 25..30<br>(t3m,t4m, f2) mon-ed% 20..25; play-addxp 6..7; play-mag-gold% 40..45 | (t1m,t2m, f12) mon-ed% 15..23; play-addxp 8..10; play-mag-gold% 20..30<br>(t2m,t3m, f12) mon-ed% 23..30; play-addxp 10..12; play-mag-gold% 50..60<br>(t3m,t4m, f12) mon-ed% 30..38; play-addxp 12..14; play-mag-gold% 80..90 |
| prefix | 157 | Splashing | (t1m,t2m, f2) mon-splash[358] 100..1; glob-density 8..13; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-splash[358] 100..1; glob-density 18..23; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-splash[358] 100..1; glob-density 23..25; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-splash[358] 150..2; glob-density 16..26; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-splash[358] 150..2; glob-density 36..46; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-splash[358] 150..2; glob-density 46..50; play-mag-gold% 70..80 |
| prefix | 312 | Bloody/Gory/Sanguinary | (t1m,t2m, f2) mon-openwounds 5..10; play-addxp 4..5; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-openwounds 10..15; play-addxp 5..6; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-openwounds 15..20; play-addxp 6..7; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-openwounds 8..15; play-addxp 8..10; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-openwounds 15..23; play-addxp 10..12; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-openwounds 23..30; play-addxp 12..14; play-mag-gold% 70..80 |
| prefix | 315 | Unhealthy/Sickly/Diseased | (t1m,t2m, f2) play-regen -40..-30; play-addxp 3..4; play-mag-gold% 5..10<br>(t2m,t3m, f2) play-regen -60..-50; play-addxp 4..5; play-mag-gold% 20..25<br>(t3m,t4m, f2) play-regen -80..-70; play-addxp 5..6; play-mag-gold% 35..40 | (t1m,t2m, f12) play-regen -60..-45; play-addxp 6..8; play-mag-gold% 10..20<br>(t2m,t3m, f12) play-regen -90..-75; play-addxp 8..10; play-mag-gold% 40..50<br>(t3m,t4m, f12) play-regen -120..-105; play-addxp 10..12; play-mag-gold% 70..80 |
| prefix | 163 | Crushing/Smashing/Obliterating | (t1m,t2m, f2) mon-crush 15..25; glob-density 8..10; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-crush 25..35; glob-density 15..18; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-crush 35..45; glob-density 18..20; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-crush 23..38; glob-density 16..20; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-crush 38..53; glob-density 30..36; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-crush 53..68; glob-density 36..40; play-mag-gold% 70..80 |
| prefix | 164 | Sparking/Shocking/Thunderous | (t1m,t2m, f2) mon-phys-as-extra-ltng 60..80; glob-density 13..15; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-phys-as-extra-ltng 70..90; glob-density 15..18; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-phys-as-extra-ltng 80..100; glob-density 18..20; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-phys-as-extra-ltng 90..120; glob-density 26..30; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-phys-as-extra-ltng 105..135; glob-density 30..36; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-phys-as-extra-ltng 120..150; glob-density 36..40; play-mag-gold% 70..80 |
| prefix | 165 | Chilling/Boreal/Glacial | (t1m,t2m, f2) mon-phys-as-extra-cold[25] 60..80; glob-density 13..15; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-phys-as-extra-cold[25] 70..90; glob-density 15..18; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-phys-as-extra-cold[25] 80..100; glob-density 18..20; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-phys-as-extra-cold[25] 90..120; glob-density 26..30; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-phys-as-extra-cold[25] 105..135; glob-density 30..36; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-phys-as-extra-cold[25] 120..150; glob-density 36..40; play-mag-gold% 70..80 |
| prefix | 166 | Smoking/Molten/Volcanic | (t1m,t2m, f2) mon-phys-as-extra-fire 60..80; glob-density 13..15; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-phys-as-extra-fire 70..90; glob-density 15..18; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-phys-as-extra-fire 80..100; glob-density 18..20; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-phys-as-extra-fire 90..120; glob-density 26..30; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-phys-as-extra-fire 105..135; glob-density 30..36; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-phys-as-extra-fire 120..150; glob-density 36..40; play-mag-gold% 70..80 |
| prefix | 167 | Poisonous/Envenomed/Plagued | (t1m,t2m, f2) mon-phys-as-extra-pois[125] 123..163; glob-density 13..15; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-phys-as-extra-pois[125] 143..184; glob-density 15..18; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-phys-as-extra-pois[125] 163..205; glob-density 18..20; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-phys-as-extra-pois[125] 185..245; glob-density 26..30; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-phys-as-extra-pois[125] 215..276; glob-density 30..36; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-phys-as-extra-pois[125] 245..308; glob-density 36..40; play-mag-gold% 70..80 |
| prefix | 168 | Runic/Taboo/Occult | (t1m,t2m, f2) mon-phys-as-extra-mag 8..10; glob-density 13..15; play-mag-gold% 5..10<br>(t2m,t3m, f2) mon-phys-as-extra-mag 10..12; glob-density 15..18; play-mag-gold% 20..25<br>(t3m,t4m, f2) mon-phys-as-extra-mag 12..14; glob-density 18..20; play-mag-gold% 35..40 | (t1m,t2m, f12) mon-phys-as-extra-mag 12..15; glob-density 26..30; play-mag-gold% 10..20<br>(t2m,t3m, f12) mon-phys-as-extra-mag 15..18; glob-density 30..36; play-mag-gold% 40..50<br>(t3m,t4m, f12) mon-phys-as-extra-mag 18..21; glob-density 36..40; play-mag-gold% 70..80 |
| prefix | 169 | Stygian | (t1m,t2m,t3m, f1) glob-add-mon-doll 690; glob-density 25..30; play-mag-gold% 40..45 | – |
| prefix | 169 | Lustful | (t1m,t2m,t3m, f1) glob-add-mon-succ 885; glob-density 25..30; play-mag-gold% 45..50 | – |
| prefix | 169 | Vampiric | (t1m,t2m,t3m, f1) glob-add-mon-vamp 1170; glob-density 18..20; play-mag-gold% 15..20 | – |
| prefix | 169 | Bovine | (t1m,t2m,t3m, f1) glob-add-mon-cow 391; glob-density 25..30; play-mag-gold% 15..20 | – |
| prefix | 169 | Reanimated | (t1m,t2m,t3m, f1) glob-add-mon-horde 698; glob-density 25..30; play-mag-gold% 35..40 | – |
| prefix | 169 | Ghastly | (t1m,t2m,t3m, f1) glob-add-mon-ghost 1111; glob-density 18..20; play-mag-gold% 15..20 | – |
| prefix | 169 | Souless | (t1m,t2m,t3m, f1) glob-add-mon-souls 918; glob-density 25..30; play-mag-gold% 50..60 | – |
| prefix | 169 | Shamanic | (t1m,t2m,t3m, f1) glob-add-mon-fetish 785; glob-density 20..25; play-mag-gold% 25..30 | – |
| prefix | 169 | Duplicated | (t1m,t2m, f1) glob-extra-boss 1 | – |
| prefix | 169 | Rampant | (t1m,t2m,t3m, f1) glob-add-mon-shriek 1169; glob-density 25..30; play-mag-gold% 20..30 | – |
| suffix | 157 | of Sturdiness/of Stability/of Vigor | (t1m,t2m, f4) mon-ac% 50..100; play-addxp 3..4; play-mag-gold% 10..15<br>(t2m,t3m, f4) mon-ac% 100..150; play-addxp 4..5; play-mag-gold% 25..30<br>(t3m,t4m, f4) mon-ac% 150..200; play-addxp 5..6; play-mag-gold% 40..45 | (t1m,t2m, f18) mon-ac% 75..150; play-addxp 6..8; play-mag-gold% 20..30<br>(t2m,t3m, f18) mon-ac% 150..225; play-addxp 8..10; play-mag-gold% 50..60<br>(t3m,t4m, f18) mon-ac% 225..300; play-addxp 10..12; play-mag-gold% 80..90 |
| suffix | 158 | of Crimson | (t1m,t2m, f4) mon-abs-fire% 4..6; glob-density 13..18; play-addxp 3..4<br>(t2m,t3m, f4) mon-abs-fire% 6..8; glob-density 18..23; play-addxp 4..5<br>(t3m,t4m, f4) mon-abs-fire% 8..10; glob-density 23..25; play-addxp 5..6 | (t1m,t2m, f18) mon-abs-fire% 6..9; glob-density 26..36; play-addxp 6..8<br>(t2m,t3m, f18) mon-abs-fire% 9..12; glob-density 36..46; play-addxp 8..10<br>(t3m,t4m, f18) mon-abs-fire% 12..15; glob-density 46..50; play-addxp 10..12 |
| suffix | 159 | of Tangerine | (t1m,t2m, f4) mon-abs-ltng% 4..6; glob-density 13..18; play-addxp 3..4<br>(t2m,t3m, f4) mon-abs-ltng% 6..8; glob-density 18..23; play-addxp 4..5<br>(t3m,t4m, f4) mon-abs-ltng% 8..10; glob-density 20..25; play-addxp 5..6 | (t1m,t2m, f18) mon-abs-ltng% 6..9; glob-density 26..36; play-addxp 6..8<br>(t2m,t3m, f18) mon-abs-ltng% 9..12; glob-density 36..46; play-addxp 8..10<br>(t3m,t4m, f18) mon-abs-ltng% 12..15; glob-density 40..50; play-addxp 10..12 |
| suffix | 160 | of Opal | (t1m,t2m, f4) mon-abs-mag% 4..6; glob-density 13..18; play-addxp 3..4<br>(t2m,t3m, f4) mon-abs-mag% 6..8; glob-density 18..23; play-addxp 4..5<br>(t3m,t4m, f4) mon-abs-mag% 8..10; glob-density 20..25; play-addxp 5..6 | (t1m,t2m, f18) mon-abs-mag% 6..9; glob-density 26..36; play-addxp 6..8<br>(t2m,t3m, f18) mon-abs-mag% 9..12; glob-density 36..46; play-addxp 8..10<br>(t3m,t4m, f18) mon-abs-mag% 12..15; glob-density 40..50; play-addxp 10..12 |
| suffix | 161 | of Azure | (t1m,t2m, f4) mon-abs-cold% 4..6; glob-density 13..18; play-addxp 3..4<br>(t2m,t3m, f4) mon-abs-cold% 6..8; glob-density 18..23; play-addxp 4..5<br>(t3m,t4m, f4) mon-abs-cold% 8..10; glob-density 18..25; play-addxp 5..6 | (t1m,t2m, f18) mon-abs-cold% 6..9; glob-density 26..36; play-addxp 6..8<br>(t2m,t3m, f18) mon-abs-cold% 9..12; glob-density 36..46; play-addxp 8..10<br>(t3m,t4m, f18) mon-abs-cold% 12..15; glob-density 36..50; play-addxp 10..12 |
| suffix | 306 | of Protection/of Security/of Safety | (t1m,t2m, f4) mon-red-dmg 100..200; glob-density 13..15; play-mag-gold% 15..20<br>(t2m,t3m, f4) mon-red-dmg 300..400; glob-density 15..18; play-mag-gold% 30..35<br>(t3m,t4m, f4) mon-red-dmg 500..600; glob-density 18..23; play-mag-gold% 45..50 | (t1m,t2m, f18) mon-red-dmg 150..300; glob-density 26..30; play-mag-gold% 30..40<br>(t2m,t3m, f18) mon-red-dmg 450..600; glob-density 30..36; play-mag-gold% 60..70<br>(t3m,t4m, f18) mon-red-dmg 750..900; glob-density 36..46; play-mag-gold% 90..100 |
| suffix | 307 | of Swiftness/of Quickness/of Velocity | (t1m,t2m, f4) mon-velocity% 20..30; glob-density 15..20; play-addxp 3..4<br>(t2m,t3m, f4) mon-velocity% 30..40; glob-density 20..25; play-addxp 4..5<br>(t3m,t4m, f4) mon-velocity% 40..50; glob-density 25..30; play-addxp 5..6 | (t1m,t2m, f18) mon-velocity% 30..45; glob-density 30..40; play-addxp 6..8<br>(t2m,t3m, f18) mon-velocity% 45..60; glob-density 40..50; play-addxp 8..10<br>(t3m,t4m, f18) mon-velocity% 60..75; glob-density 50..60; play-addxp 10..12 |
| suffix | 308 | of Regeneration/of Renewal/of Rebirth | (t1m,t2m, f4) mon-regen 1024..1560; play-addxp 3..4; play-mag-gold% 20..25<br>(t2m,t3m, f4) mon-regen 1560..2048; play-addxp 4..5; play-mag-gold% 35..40<br>(t3m,t4m, f4) mon-regen 2048..2560; play-addxp 5..6; play-mag-gold% 50..55 | (t1m,t2m, f18) mon-regen 1536..2340; play-addxp 6..8; play-mag-gold% 40..50<br>(t2m,t3m, f18) mon-regen 2340..3072; play-addxp 8..10; play-mag-gold% 70..80<br>(t3m,t4m, f18) mon-regen 3072..3840; play-addxp 10..12; play-mag-gold% 100..110 |
| suffix | 309 | of the Leech/of the Parasite/of the Bloodsucker | (t1m,t2m, f4) mon-lifesteal-hp% 4..8; glob-density 13..15; play-addxp 5..6<br>(t2m,t3m, f4) mon-lifesteal-hp% 8..12; glob-density 15..20; play-addxp 6..7<br>(t3m,t4m, f4) mon-lifesteal-hp% 12..16; glob-density 18..20; play-addxp 7..8 | (t1m,t2m, f18) mon-lifesteal-hp% 8..16; glob-density 26..30; play-addxp 10..12<br>(t2m,t3m, f18) mon-lifesteal-hp% 16..24; glob-density 30..40; play-addxp 12..14<br>(t3m,t4m, f18) mon-lifesteal-hp% 24..32; glob-density 36..40; play-addxp 14..16 |
| suffix | 310 | of Resilience/of Hardiness/of Fortitude | (t1m,t2m, f4) mon-balance1 40..60; glob-density 15..20; play-mag-gold% 10..15<br>(t2m,t3m, f4) mon-balance1 60..80; glob-density 20..25; play-mag-gold% 25..30<br>(t3m,t4m, f4) mon-balance1 80..100; glob-density 25..30; play-mag-gold% 40..45 | (t1m,t2m, f18) mon-balance1 60..90; glob-density 30..40; play-mag-gold% 20..30<br>(t2m,t3m, f18) mon-balance1 90..120; glob-density 40..50; play-mag-gold% 50..60<br>(t3m,t4m, f18) mon-balance1 120..150; glob-density 50..60; play-mag-gold% 80..90 |
| suffix | 316 | of Health/of Vitality/of Vim | (t1m,t2m, f2) mon-hp% 5..10; glob-density 15..20; play-addxp 6..8<br>(t2m,t3m, f2) mon-hp% 10..15; glob-density 20..25; play-addxp 8..10<br>(t3m,t4m, f2) mon-hp% 15..20; glob-density 25..30; play-addxp 10..12 | (t1m,t2m, f9) mon-hp% 10..20; glob-density 30..40; play-addxp 12..16<br>(t2m,t3m, f9) mon-hp% 20..30; glob-density 40..50; play-addxp 16..20<br>(t3m,t4m, f9) mon-hp% 30..40; glob-density 50..60; play-addxp 20..24 |
| suffix | 313 | of Imbalance/of Inequality/of Uneveness | (t1m,t2m, f4) play-balance1 -20..-10; glob-density 15..20; play-addxp 3..4<br>(t2m,t3m, f4) play-balance1 -30..-20; glob-density 20..25; play-addxp 4..5<br>(t3m,t4m, f4) play-balance1 -40..-30; glob-density 25..30; play-addxp 5..6 | (t1m,t2m, f18) play-balance1 -30..-15; glob-density 30..40; play-addxp 6..8<br>(t2m,t3m, f18) play-balance1 -45..-30; glob-density 40..50; play-addxp 8..10<br>(t3m,t4m, f18) play-balance1 -60..-45; glob-density 50..60; play-addxp 10..12 |
| suffix | 164 | of Warding/of Guarding/of Preservation | (t1m,t2m, f4) mon-curseresist-hp% 20..40; play-addxp 5..6; play-mag-gold% 20..25<br>(t2m,t3m, f4) mon-curseresist-hp% 40..60; play-addxp 6..7; play-mag-gold% 35..40<br>(t3m, f4) mon-curseresist-hp% 60..80; play-addxp 7..8; play-mag-gold% 50..55 | (t1m,t2m, f18) mon-curseresist-hp% 30..60; play-addxp 10..12; play-mag-gold% 40..50<br>(t2m,t3m, f18) mon-curseresist-hp% 60..90; play-addxp 12..14; play-mag-gold% 70..80<br>(t3m, f18) mon-curseresist-hp% 90..120; play-addxp 14..16; play-mag-gold% 100..110 |
| suffix | 166 | of Hindrance/of Aversion/of Permeability | (t1m,t2m, f4) play-res-all -15..-10; play-addxp 4..5; play-mag-gold% 15..20<br>(t2m,t3m, f4) play-res-all -20..-15; play-addxp 5..6; play-mag-gold% 30..35<br>(t3m,t4m, f4) play-res-all -25..-20; play-addxp 6..7; play-mag-gold% 50..55 | (t1m,t2m, f18) play-res-all -23..-15; play-addxp 8..10; play-mag-gold% 30..40<br>(t2m,t3m, f18) play-res-all -30..-23; play-addxp 10..12; play-mag-gold% 60..70<br>(t3m,t4m, f18) play-res-all -38..-30; play-addxp 12..14; play-mag-gold% 100..110 |
| suffix | 165 | of Cracking/of Crumbling/of Disintegration | (t1m,t2m, f4) play-ac% -30..-20; glob-density 13..18; play-mag-gold% 15..20<br>(t2m,t3m, f4) play-ac% -40..-30; glob-density 15..18; play-mag-gold% 30..35<br>(t3m,t4m, f4) play-ac% -50..-40; glob-density 18..20; play-mag-gold% 45..50 | (t1m,t2m, f18) play-ac% -45..-30; glob-density 26..36; play-mag-gold% 30..40<br>(t2m,t3m, f18) play-ac% -60..-45; glob-density 30..36; play-mag-gold% 60..70<br>(t3m,t4m, f18) play-ac% -75..-60; glob-density 36..40; play-mag-gold% 90..100 |
| suffix | 314 | of Fumbling/of Bumbling/of Blundering | (t1m,t2m, f4) play-block -20..-10; glob-density 13..18; play-mag-gold% 15..20<br>(t2m,t3m, f4) play-block -30..-20; glob-density 18..23; play-mag-gold% 30..35<br>(t3m,t4m, f4) play-block -40..-30; glob-density 23..25; play-mag-gold% 40..50 | (t1m,t2m, f18) play-block -30..-15; glob-density 26..36; play-mag-gold% 30..40<br>(t2m,t3m, f18) play-block -45..-30; glob-density 36..46; play-mag-gold% 60..70<br>(t3m,t4m, f18) play-block -60..-45; glob-density 46..50; play-mag-gold% 80..100 |
| suffix | 317 | of Darkness | – | (t1m,t2m NOT SPAWNABLE, f0) play-lightradius -100..-80; glob-density 5..8; play-mag-gold% 15..20<br>(t2m,t3m NOT SPAWNABLE, f0) play-lightradius -140..-100; glob-density 8..10; play-mag-gold% 25..30<br>(t3m,t4m NOT SPAWNABLE, f0) play-lightradius -180..-140; glob-density 10..13; play-mag-gold% 35..40 |
| suffix | 318 | of the Jeweler | – | (t1m,t2m NOT SPAWNABLE, f5) mon-dropjewelry 1..2<br>(t2m,t3m NOT SPAWNABLE, f5) mon-dropjewelry 2..3<br>(t3m NOT SPAWNABLE, f5) mon-dropjewelry 3..4<br>(t1m,t2m NOT SPAWNABLE, f5) mon-dropjewelry 2..3<br>(t2m,t3m NOT SPAWNABLE, f5) mon-dropjewelry 4..5<br>(t3m NOT SPAWNABLE, f5) mon-dropjewelry 6..7 |
| suffix | 318 | of the Smith | (t1m,t2m, f9) mon-dropweapons 3..4<br>(t2m,t3m, f9) mon-dropweapons 4..5<br>(t3m, f9) mon-dropweapons 5..6 | (t1m,t2m, f18) mon-dropweapons 4..5<br>(t2m,t3m, f18) mon-dropweapons 6..7<br>(t3m, f18) mon-dropweapons 8..10 |
| suffix | 318 | of the Armorer | (t1m,t2m, f9) mon-droparmor 2..3<br>(t2m,t3m, f9) mon-droparmor 4..5<br>(t3m, f9) mon-droparmor 6..7 | (t1m,t2m, f18) mon-droparmor 4..5<br>(t2m,t3m, f18) mon-droparmor 6..7<br>(t3m, f18) mon-droparmor 8..10 |
| suffix | 318 | of the Crafter | (t1m,t2m, f5) mon-dropcrafting 2..3<br>(t2m,t3m, f5) mon-dropcrafting 3..4<br>(t3m, f5) mon-dropcrafting 4..5 | (t1m,t2m, f18) mon-dropcrafting 3..4<br>(t2m,t3m, f18) mon-dropcrafting 4..5<br>(t3m, f18) mon-dropcrafting 6..7 |
| suffix | 320 | of the Adventurer | (t1m,t2m,t3m,t4m NOT SPAWNABLE, f0) glob-arealevel 1 | – |

### 2.4 Every map stat: target, effect, handler (DATA routing + READ handlers)
Handler abbreviations:
- **A** = applier PD 0x102DBA50 (§2.5). It is applied to monsters that spawn in levels 137–201 whose MonStats byte +0x4C is 0 (0x102DBFE0), and to players entering a map level.
- **Z** = zone builder 0x102DC520 (§3).
- **D** = death/drop code (`drops.md`).

| map stat (id) | property / typical source | target | effect | handler | notes |
|---|---|---|---|---|---|
| map_play_magicbonus 370 / goldbonus 371 | `map-play-mag-gold%` (same value twice) | player 80 / 79 | +MF / +GF | A | |
| map_play_addexperience 373 | `map-play-addxp` | player 85 | +% experience (stock stat) | A | |
| map_play_ac% 410 | Cracking… | player 16 | −% defense | A | |
| map_play_fastergethitrate 411 | Imbalance… | player 99 | −FHR | A | |
| map_play_toblock 412 | Fumbling… | player 20 | −block | A | |
| map_play_hpregen 413 | Unhealthy… | player 74 | negative life regen ("Drain Life") | A | |
| map_play_{fire,light,cold,poison}resist 428–431 | Hindrance… (`res-all`) | player 39/41/43/45 | −resist | A | |
| map_play_max*resist 418–421 | corruption `res-all-max` | player 40/42/44/46 | −max resist | A | |
| map_play_fasterattackrate 451 / fastercastrate 452 / velocitypercent 457 | corruption `swing-cast`, `speed-all` | player 93 / 105 / 67 | ± IAS/FCR/FRW | A | |
| map_play_maxhp/maxmana_percent 454/455 | (no source found) | player 76/77 | | A | |
| map_play_damageresist 456 | corruption | player 36 | −DR% | A | |
| map_play_lightradius 467 | of Darkness (unspawnable) | player 89 | −light radius | A | |
| map_mon_passive_*_mastery 388–391 | Flaming/Hibernal/Static/Toxic | monster 329–332 | +% elemental damage | A | cold/poison apply to all their attacks |
| map_mon_fasterattackrate/castrate 392/393 | Fast/Speedy/Ludicrous | monster 93/105 | | A | |
| map_mon_tohit 394 + map_mon_pierce 406 | Silver/Shining/Opulent (same value) | monster 119 + 156 | +AR%, pierce chance | A | map_mon_att 277 also → 119 (SetStat overwrite if both) |
| map_mon_ac% 395 | Sturdiness… | monster 16 | +defense% | A | |
| map_mon_absorb*_percent 396–399 | of Crimson / Tangerine / Opal / Azure | monster 148 / 146 / 144 / 142 | **cold / magic / light / fire** absorb | A | |
| map_mon_normal_damage_reduction 400 | Protection… | monster 34 | flat physical DR | A | |
| map_mon_velocitypercent 401 | Swiftness… | monster 67 | +FRW | A | |
| map_mon_hpregen 402 | Regeneration… | monster 74 | life regen (<<8 units) | A | |
| map_mon_lifedrainmindam 403 | Leech… (+405 same value) | monster 60 | flat "life drain" per hit | A | also gives +max life |
| map_mon_fastergethitrate 404 | Resilience… | monster 99 | +FHR | A | |
| map_mon_maxhp_percent 405 | Large/Huge/Colossal, of Health…, Leech… | monster **stat 7 directly** | life += life/100·v, clamp 0x7FFFFFFF | A | writes marker stat 432 = 1; see §2.5 |
| map_mon_openwounds 407 / crushingblow 408 | Bloody…, Crushing… | monster 135/136 | OW/CB on players | A | |
| map_mon_curse_resistance 409 | Warding… | monster 109 | curse duration reduction | A | |
| map_mon_passive_*_pierce 414–417 | (no affix; `pierce-all` property) | monster 333–336 | −player resist | A | |
| map_mon_ed% 426 | Strong… | monster 25 | +% physical damage | A | |
| map_mon_splash 427 | Splashing (skill 358 proc_SplashDamage, 100 %/lvl 1; magic 150 %/lvl 2) | monster 359 (skill-event, layer = skill<<6 \| level) | melee splash | A | §2.5 |
| map_mon_phys_as_extra_{ltng,cold,fire,pois,mag} 432–436 | Sparking…Plagued, Runic… | monster 50/51, 54/55/56, 48/49, 57/58/59, 52/53 | v % of stats 21/22 | A | §2.5 |
| map_mon_deadlystrike 449 | corruption | monster 141 | PD crit/DS | A | |
| map_mon_cannotbefrozen 450 | corruption `nofreeze-hp%`, Fallen Gardens | monster 153 | | A | |
| map_mon_skillondeath 453 | (no source) | monster 197 | skill-event | A | |
| map_mon_drop{jewelry,weapons,armor,crafting,charms,jewels} 494–497/502/506 | of the Jeweler (unspawnable), Smith, Armorer, Crafter; unique maps | monster (same id) | extra TC roll `pdRand%100 < v` | A + D | `drops.md` |
| map_defense 369 | (Divide 1024) | monster 369 | none found ("Corrupt" string) | A | |
| map_glob_density 372 | most affixes | key 0 | MonDen × (1+v/100) | Z | |
| map_glob_arealevel 374 | of the Adventurer (unspawnable), corruption, unique maps | key 1 | +monster level | 0x10268D00 | also written to the region by Z (dead field) |
| map_glob_monsterrarity 375 | Champion's, corruption | key 2 | MonUMin/MonUMax × (1+v/100) | Z | byte, wraps at 256 |
| map_glob_add_mon_* 437–442, 470, 471, 499 | Stygian, Lustful, Vampiric, Bovine, Reanimated, Ghastly, Souless, Shamanic, Rampant | key 3 | adds MonStats class v to the level's spawn list | Z | |
| map_glob_skirmish_mode 493 | Fortify Map (cube) | key 4 | fortified (§3) | Z + D | |
| map_glob_extra_boss 498 | Duplicated (t1m,t2m) | key 5 | map-boss preset spawned twice | PD 0x102DE287 | |
| map_glob_dropcorrupted 503 | Warlord of Blood | key 6 | drops corrupted `rand()%100 < v` | D 0x102C8D0B | |
| map_glob_boss_dropskillers 186 | Zhar | key 7 | "boss drops a skill charm" | **none** | no reader |
| map_glob_boss_dropcorruptedunique 187 | Warlord | key 8 | | **none** | no reader |
| map_glob_boss_dropubermats 211 / droppuzzlebox 212 / dropfacet 505 | Void / Imperial / Fallen Gardens | key 9 (all three) | | **none** | no reader; three stats share one key |
| map_glob_sundermonsters 508 | (no affix source) | key 10 | sunder | A | §2.5 |
| map_glob_dropsocketed 272 | Ureh | key 11 | sockets on drops | D 0x102C8D3C | |
| map_glob_dropbonus 273 | Djinn | key 12 | extra full drop `pdRand%100 < v` | Z → monster 273 | |
| map_glob_dropethereal 274 | Na-Krul | key 13 | forced ethereal `rand()%100 < v` | D 0x102C8BE1 | |
| treacherous 276 | (zone events) | key 14 | +200 % life + stat 276 | Z | |
| map_glob_bossdropstreasure 275 | Djinn | key 15 | "treasure explosion" | **none** | no reader |
| map_force_event 500 | cube (map + iwss) | Divide 0 | forces a random map event | open handler | |

Keys read anywhere in PD (grep of 0x102DBA30/0x102DBA00 and of `+0x2630..0x26F0`): 1, 5, 6, 10, 11, 13 directly, and 0, 1, 2, 3, 4, 12, 14 in Z. **Keys 7, 8, 9 and 15 have no reader in these DLLs** (READ). This covers skill charm, corrupted unique, uber mats/puzzle box/facet, and treasure explosion.

### 2.5 The applier PD 0x102DBA50 (monster branch VERIFIED)
`harness/mapapply.c` runs the real applier with the real D2Common stat-list code, SetStat #10188 → 0x6FD8A280 with sorted insert 0x6FD88640. It uses random monster lists built like the map-open builder and compares every final `(stat<<16|layer, value)` and the life write with the model in `maps.js` (`applyMonsterList`).
- Result: **20000/20000 match**.
- Mutation "maxhp% as a plain stat": 17848/20000.
- Mutation "no layer write": 15066/20000.

Only stubbed calls are involved. None of the executed D2Common bytes are touched by PD2's runtime patches (checked against `pd2_patch_records.json`).

```
new statlist S (state 198 'map'; skipped if the unit already has state 198)
for i, (param, stat, v) in monster list:
  if 432 <= stat <= 436:                       # physical as extra element
     mn = ftoui(float(v)/100 * float(stat21)); mx = ftoui(float(v)/100 * float(stat22))
     later: SetStat(S, maxStat, mx); if cold/poison: SetStat(S, lenStat, param or 25/125)
     stat = minStat; v = mn; param = 0
  if stat == 76 and unit is not a player:      # +% monster life
     life7 = min(life7 + (life7/100)*v, 0x7FFFFFFF); SetUnitStat(unit, 7, life7); SetStat(S, 432, 1); continue
  SetStat(S, stat, v ? v : 1, layer 0)         # SET, not add: two map stats with the same target overwrite
  if S.count > i: S.entry[i].layer = param     # <-- the layer is written to index i of the SORTED list
```

**Other branches** (READ, not run):
- PvP levels 157/159/166 set 443/481 = −4, 491 = 1, and 482 or 492 = 25.
- **Sunder (key 10).** It applies only when the key entry's *value* equals the level id and its *layer* is nonzero.
  - Those entries are written by PD's random-zone code 0x102C2469 as {layer 10, key 10, value = levelId}.
  - A map item's `map_glob_sundermonsters` becomes {layer 0, key 10, value 1}.

---------------------------------------------------------------------------------------------------------------------

## 3. Density, champion/unique packs, monster level, XP and drops

Zone builder 0x102DC520 (READ). It runs once per opened map with `apply` = 1, and for the extra levels of multi-level maps with `apply` = 0 (then no monster-list entries are appended). It edits the stock MonRegion `game+0xF0[level]` (fields filled by D2Game 0x6FCEABEC from Levels.txt):

| key | field | rule |
|---|---|---|
| 0 density | MonDen `+0x2B8` | `ftoui(float(MonDen) * (1 + v/100))` |
| 1 area level | `+0x2DC`/`+0x2E0` += v | never read by D2Game. The level actually used comes from PD 0x10268D00 = D2Common #10894 level + key 1 |
| 2 rarity | MonUMin/MonUMax bytes `+0x2BC/+0x2BD` | `trunc(x * (1 + v/100))` stored as a byte (wraps above 255) |
| 3 add monster | spawn slot at `+0x14 + 0x34·n` | class = v, counts +0x10/+0x11/+0x12; n capped at 12 |
| 4 fortified | MonDen, UMin, UMax × **v/100** (no +1) | fortify uses v = 50, so **−50 % density and −50 % champion/unique packs** |
| 12 drop bonus | monster stat 273 = v | |
| 14 treacherous | monster stat 76 = 200, 276 = v | |

Fortify (key 4) also appends these to every map monster's list (with `apply`):
- +100 % life (76)
- +40 % damage (25)
- +40 % fire/light/cold/poison mastery (329–332)
- +20 % magic mastery (357)
- 493 = v

**Item stats are summed before routing.** All density rolls on a map add into one `map_glob_density` value, so density is `MonDen·(1+Σ/100)`.

**Stock spawn** (D2Game 0x6FC67A61, not patched by PD, READ):
- MonDen is clamped to **10000**.
- Each 3×3-subtile cell spawns a pack when `rand(100000) <= MonDen`.
- So the cap is reached at +400 % on a 2000-density map.
- Pack type (normal, champion or unique) is chosen by 0x6FCFF080. It compares the level's special-pack counter `+0x2C8` with MonUMin/MonUMax. PD2's map levels use 40/60, much higher than stock. The exact per-branch probabilities were not decoded.

**Worked example** (model, `PD2Maps.zone`):
- Throne (t2m, level 150, MonDen 1815) with a rolled total of +86 % density gives MonDen `trunc(1815·1.86)` = 3375. That is 3.4 % of cells.
- Champion's (magic, t3-grade) 75–90 % turns MonUMin/Max 40/60 into 70/114 at +75 %.

**XP / MF / quantity / quality.** The map's player list gives stat 85 `item_addexperience` (+% XP), 80 (MF) and 79 (GF), applied like item stats (READ; routing DATA). Drop quantity and quality effects are in `drops.md`:
- `map_mon_drop*` extra TC rolls
- key 12 extra full drops
- fortify ×2 drop multiplier
- the extra full drop for every monster in T4 map levels
- corrupted/socketed/ethereal drops

There is no separate "extra rare" mod in the tables.

---------------------------------------------------------------------------------------------------------------------

## 4. Map currency (cube; DATA + READ)
Everything here uses the live MPQ `CubeMain.txt`. The full cube engine, the refusal rules and the corruption tables for every item type are in `cube.md` (§1, §2); this section keeps only what touches maps. `maps.js` does not duplicate the tables: `PD2Maps.corruptionTable(code, quality)` and `corruptionOutcome(code, quality, roll)` call `cube.js` (`PD2Cube.corrupt`).

**BLOCK rows.** Rows with **op 30** are "BLOCK" rows. When one matches, PD's cube 0x102BDCB0 stops and nothing happens. They stop white maps from being fortified, heroed, event-forced or corrupted, and they also block the stack-duplication rows.

**Uncorrupted only.** Every map re-roll row carries `op 18 361 = 0`: stat 361 (`corruptor`) must be 0. A corrupted map therefore cannot be re-rolled, scoured or corrupted again. The same holds for a map that is left with a phase-1 roll and no outcome (T4, unique; see below; `cube.md` §2.4).

| Action | Inputs | Output |
|---|---|---|
| Upgrade magic → rare | T1–T3 magic + `runs` + `ggm4` + `upma` (or `urma`) | same base, new rare roll (6 affixes, §2.2) |
| Reroll rare | rare + `runs` + `ggm4` + `rera` (or `rrra`) | new rare roll |
| Imbue | white + `jewg` + `imma` (magic), or white + `imra` + `jewg` + `runs` (rare); or `irma` / `irra` | magic / rare |
| Reroll magic | magic + `imma` or `irma` | new magic roll (1 prefix + 1 suffix) |
| Scour | magic/rare + `scou` | white |
| Upgrade tier | 3 × T1 (or T2) + `upmp` | a new **white** T2 (T3) of random base |
| Reroll base | 3 × T1/T2/T3 of the same quality | new random base of that tier and quality |
| T3 → dungeon | T3 (any quality) + `scrb` | rare **t4m** with fixed extra mods: +20–30 FRW, 80–100 AR%/pierce, splash, 10–20 % life + cannot be frozen, +40–50 % XP |
| 3 dungeons → dungeon | 3 rare t4m | new rare t4m with the same fixed extras |
| Fortify | non-white map + `fort` | +`map_glob_skirmish_mode` 50 (§3) |
| Standard of Heroes | non-white map + `std` | `heroic` + 20 MF, 20 GF, 20 density, 10 XP |
| Forced event | non-white map + `iwss` | `map_force_event` 63 |
| **Corrupt** | non-white map + `wss` | two passes, below |

### 4.1 Corruption (DATA + READ; from `cube.md` §2.1 and §2.5)
One click runs two passes:
1. **Phase 1**, row 341 `map + wss` (`op 18 361 = 0`): stat 361 = a stock property roll, uniform on **1..3000**. The shard is handed back.
2. PD re-runs the transmute at once. **Phase 2**: the first row in file order whose input matches the map and whose `op 16` test holds wins. The test is the stock one, **`stat 361 ≤ value`** (D2Game jump table 0x6FC90B34 → 0x6FC9060F; it fails when stat > value). The winning row sets stat 360 = outcome code (78–88), stat 361 = 3001 (T1–T3) or 1001 (T4), and its mods; the shard is used up.

So P(row k) = (value_k − value_{k−1}) / 3000.

**T1–T3 maps (magic or rare):** 11 outcomes of 9 % each, then the 8 unique maps at 1 % together. There is no gap: the last unique row ends at 3000.

| corruptor | p | outcome (T1 / T2 / T3 values) |
|---|---|---|
| 1–270 | 9 % | monster deadly strike 4–8 / 8–12 / 12–16, density, XP |
| 271–540 | 9 % | monsters cannot be frozen + 10–15 / 15–20 / 20–25 % life, density, XP |
| 541–810 | 9 % | players −5..−15 / −15..−25 / −25..−35 IAS and FCR, density, rarity |
| 811–1080 | 9 % | players −5..−10 / −10..−15 / −15..−20 % DR, density, rarity |
| 1081–1350 | 9 % | players +20–30 / 30–40 / 40–50 IAS, FCR and FRW, rarity |
| 1351–1620 | 9 % | +50 / 75 / 100 MF and GF, rarity |
| 1621–1890 | 9 % | density +60–80 / 80–100 / 100–120 |
| 1891–2160 | 9 % | +1 area level, density |
| 2161–2430 | 9 % | rarity +30–40 / 40–50 / 50–60 |
| 2431–2700 | 9 % | −2..−3 / −3..−4 / −4..−5 max resists, density, rarity |
| 2701–2970 | 9 % | +1 % jewelry drop mod |
| 2971–3000 | 1 % | becomes a unique map: Zhar, Fallen Gardens, Hellcaves (Warlord), Ureh, Djinn's Domain, Na-Krul's Abyss 0.13 % each; Outer Void, Imperial Palace 0.10 % each |

(T1 rows 347–365; T2 and T3 blocks follow with the tier values.)

**T4 maps (dungeons: Torment, Sanctuary of Sin, Kanemith, Mesa), rows 2077–2086.**
- The T4 rows run 100, 200, …, 1000 against the same 1..3000 phase-1 roll.
- So each of the 10 outcomes is **3.33 %**, with T3-grade values except deadly strike 15–20 and cannot be frozen + 25–35 % life.
- **2/3 of attempts do nothing.** The shard comes back and the map keeps 361 = X, which locks it against every later orb.

**Unique maps (t5me):** row 341 matches but no phase-2 row exists. The only effect is that 361 is set, which locks the map.

**White maps:** refused by BLOCK row 339.

**Other routes to a corrupted item.** The corruption engine PD 0x102BB5F0 is a different path, used by `map_glob_dropcorrupted` drops (key 6, PD 0x102C8D30) and a few other callers. It pre-corrupts a drop with a normal outcome of that item's own table (`cube.md` §2.6). It does not change the map cube odds above.

The "Random rare / down tier / up tier" rows are disabled.

**Uber maps:** cube `dcma`/`rtma`/`luca` alone to cycle `uber_difficulty` 0 → 1 → 2 → 0 (op 18 on stat 185). Uber Ancients map = `ubaa` + `ubab` + `ubac`.

---------------------------------------------------------------------------------------------------------------------

## 5. Map bosses and uber maps (READ + DATA)
Map bosses are presets in the level files. PD's preset spawner 0x102DE248 spawns the preset again when the class is a map boss (0x102C7C10) or class 1044, and key 5 (`Duplicated`) is set. The boss sets below were extracted from PD's static initializers with `sets.py`; the method was validated on the T4 level set used in `drops.md`.

| set | tier | MonStats (hcIdx) | Hell TC |
|---|---|---|---|
| 0x104E3130 | T1 | IceBoss 861, torajanBoss 879, TombBoss 870, act2hireTraitorBoss 893, megademonboss 746, westmarchMapBoss 996, unravelerboss 750, spiderboss 915 | Map Boss T1 |
| 0x104E2FD4 | T2 | lernaeanhydra2 1026, ArcaneBoss 800, AshenBoss 1025, archerBoss 894, SewerBoss 884, LibraryBoss 997, CanyonBoss 963, ThroneBoss 882, **TortureHallsBoss 1149** | Map Boss T2 (**TortureHalls: Map Boss T1**) |
| 0x104E31B8 | T3 | CowBoss 826, leoricMapBoss 998, SiegeBoss 883, **BastionBoss 809**, DemonRoadBoss 1121, SkovosBoss 1122, baalminionboss 755, MarketBoss 964 | Map Boss T3 (**Bastion: Map Boss T2**) |
| 0x104E31C0 | T4 | fingermageboss 747, siegebeastMapBoss 1000, KanemithBoss 1103 | UberAncients |
| 0x104E31A0 | T4/special | willowispboss 743, RadamentBoss 1104, SharpToothBoss 1105 | UberAncients |
| 0x104E3010 | T5 | Zhar + 3 minibosses, Warlord, Rakanoth, Iskatu, Imperial ×3, Ureh ×2 | Map Boss T3 |

- Bosses are MonStats `boss` with fixed Hell life: T1 2800–5500, T2 3800–6500, T3 6000–7000, T4 10000–16500, and Ureh boss 15000.
- They are scaled only by players and by the map's `+% life` (§2.5).
- Crushing blow on a map boss uses divisor 30 (`uber_review.md`).
- Their drops use the TCs above plus the map-boss special rolls in `drops.md` (unique map 1/3000).
- The boss-drop flags on unique maps (keys 7, 8, 9, 15) are **not read** by these DLLs.
- Uber maps: see `uber_review.md`. `uber_difficulty` is stored in `game+0x2629` when levels 137/161/188 open. It drives AI delay and bonus drops there.

---------------------------------------------------------------------------------------------------------------------

## 6. Unverified or open

- The cube output ilvl for re-rolled maps, which decides 5 vs 6 rare affixes when below 85.
- Pack-type probabilities in 0x6FCFF080.
- XP effect of fortify.
- Whether the realm server reads keys 7/8/9/15.
