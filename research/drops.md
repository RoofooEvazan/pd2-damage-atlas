# Item drops (PD2 = 1.13c D2Game/D2Common + ProjectDiablo.dll)

Engine: `drops.js` (browser `window.PD2Drops`, Node `module.exports`), data: `drops-data.json` (built by
`extract_drops.py` from PD2 `data.zip` excel + the map simulator species list), tests: `test_drops.js`,
harnesses: `harness/tcq.c` + `tcq_driver.js`, `harness/upq.c` + `upq_driver.js`.

Status key: **VERIFIED** = the real game code ran natively in the harness and matched drops.js on every case.
**READ** = from the disassembly, not executed. **DATA** = straight from the txt tables. **APPROX** = simplified.

## Harness results

| harness | real code run | cases | result |
|---|---|---|---|
| `tcq` | TC wrapper 0x6FC32D60, D2Common #10634 0x6FDA3420 (TC lookup + upgrade), TC routine 0x6FC32380, quality roll 0x6FC2FC40 (+ MF getter 0x6FC2F2F0, dim. 0x6FC2E130), gold find 0x6FC2F260, RNG 0x6FC21180/0x6FC211D0, `_ftol` 0x6FD15724 | 20,000 random (TC, ilvl, monster level, noRatio/boss bits, difficulty, game players, party, monster_playercount, MF, GF, seed, gold) on the **full PD2 TC table** laid out in the real 0x2C/0x1C record format | **0 mismatches** (51,810 items: item id, quality, flags, final gold, final seed). Same result with x87 PC=53 (Windows) and PC=64 |
| `upq` | unique picker 0x6FC2F370 (with the PD2 byte patch at 0x6FC2F5CE), set picker 0x6FC33C20 | 20,000–30,000 random (base code, ilvl, seed, game type/ladder, drop flags, per-game "already dropped" bits) on the full UniqueItems/SetItems tables in the real 0x14C/0x1B8 layout | **0 mismatches** (pick, fail, final item seed) |

Stubs (tcq): item record/type/ItemRatio row lookups, stat getters/setters, is-gold, max gold, MonStats lookup
0x6FC21220, game player count 0x6FC579A0, party count 0x6FCBC1F0, item creation 0x6FC31880 (records id/quality/flags).
Stubs (upq): item record/version/level/seed pointer, set-index/flag setters, unique-apply 0x6FC2EDC0.

## Pipeline for one kill

PD2 replaces the stock monster death handler (D2Game table 0x6FD1A498: 0x6FC96270 → PD **0x102C11F0**). PD's handler
does its extras first and ends by calling the stock handler (lazy import → 0x6FC96270), whose drop call
(0x6FC962A7 → 0x6FC95900) PD hooks to 0x102EDAF0 → 0x102D5B80 → stock **0x6FC95680** (the main drop).

### 1. Monster → treasure class (READ; upgrade VERIFIED)
- 0x6FC95680: difficulty d = game+0x6D. SuperUnique (0x6FC43220 ≠ −1): SuperUniques TC(d) (or MonStats TC3 when the
  record is missing). Else MonsterData+0x16: 0x4 champion → MonStats **TreasureClass2**, 0x8 unique → **TreasureClass3**,
  otherwise (normal, minion 0x10) **TreasureClass1**. Words at MonStats+0x86/0x88/0x8A (+8·d). TreasureClass4 (quest)
  only for quest monsters (TCQuestId) – not used by map monsters.
- Item level / monster level: `ilvl = max(1, stat 12)` (0x6FC2E5D0; PD's copy 0x10268DE0 identical).
- Map monster level = area level + elite bonus. MonUMod 4 `leveladd` +3 (0x6FC41E80), MonUMod 16 `champion` −1
  (0x6FC42E1D) → **champion +2, unique/minion +3** (berserk +3 as in the simulator). PD map mod `map_glob_arealevel`
  (global id 1) raises the level's monster level (0x102DC666). (READ)
- **TC upgrade** 0x6FC32D60: only when game is expansion, **d > 0** and the MonStats flag byte has neither `noRatio`
  (bit 2) nor `boss` (bit 6) (gdwBitMasks); level = monster stat 12. #10634 0x6FDA3420 then walks **forward** from the
  TC while the next record has the same `group` (≠0) and `level ≤ mlvl`. Sub-TCs are looked up with level 0 (no upgrade).
  - **Finding:** `Map H2H t1/t2/t3` (and Cast/Miss/Wraith/Quill/Swarm/Cow) share one group with **all three at level 85**,
    so every normal map monster (mlvl ≥ 85) is upgraded to the **t3** TC regardless of the map tier; t3 → `Map Good t3` →
    `T3 Map Drop` (maps T1/T2/T3 at 32/33/33). Champion/unique map TCs (`Map Champ tN`, `Map Unique tN`) have no group and
    keep their tier.

### 2. TC resolution — D2Game 0x6FC32380 (VERIFIED)
Records (D2Common): TC 0x2C bytes {+0 group, +2 level, +4 nEntries, +8 classic total, +0xC expansion total, +0x10 picks,
+0x14 NoDrop, +0x1A..+0x24 words magic, rare, set, unique, (+2 unused), +0x28 entries}; entry 0x1C {+0 classic cum,
+4 expansion cum, +8 id, +0xA mul/unique idx, +0xC flags (1 unique, 2 set, 4 TC, 0x10 expansion-only), +0xE.. 6 words}.
- Loader (READ): TC 0 is a dummy (0x6FDA47D0 creates it first; #10634 rejects id 0); then the auto TCs, then
  TreasureClassEx rows in file order, **single pass** (0x6FDAA564 creates the row, 0x6FDAA5F7 parses entries), so a
  reference to a later row is dropped: `MapStat DropCrafting` loses `Treasure Fallen Rune NM`, `InvaderBarbarian1h` loses
  `Invader2hSword`; unknown names (`t3b`, `Map Uitem`, `Act 5 (N) Weap C` …) are dropped too. Entry name resolution:
  item code (≤4 chars) → TC name → unique name (flag 0x11) → set name (0x12); `,mul=` → +0xA.
- Auto TCs (0x6FDA47D0, READ): for every ItemTypes row with `TreasureClass`=1 and L = 3,6,…,96: `<code><L>`, picks 1,
  group 0, level L−3; items (not quest, spawnable, of that type, not `tpot` unless the type is `tpot`) with
  L−3 < level ≤ L; prob = ItemTypes `Rarity` of the item's type (min 1).

> **Correction (audit):** `t3b` (Kyovashad Map, t3m) is a valid item in the live MPQ `Misc.txt`; it was "unknown" only in the stale `data.zip`. `Map Tier 3` therefore has 9 entries live, not 8. The `drops-data.json` now on disk (rebuilt 26 Sep 00:12) already contains `t3b`, t57/t58 and the live map tiers (e.g. t11 = type 106 t1m), although its `generated` string still says "data.zip". The header of this page (built from `data.zip`) is stale for Misc/Levels; see `data_sources.md`.

- Walk: a stack of ≤64 {tc, remaining picks, 6 quality words}. Positive picks: `range = total + NoDrop'`,
  `roll = rand(range)` (dropping monster's seed); `roll < NoDrop'` → no drop; else entry = binary search on the
  cumulative starts (exact search in `findEntry`). Negative picks: the k-th pick takes `roll = k` (probabilities are counts).
  A TC entry pushes the child (replacing the slot when the parent has no picks left); quality words become
  `parent == 0 ? child : max(parent, child)` (so the **max along the chain**).
- **NoDrop with players** (0x6FC324B5): party = party members alive in the killer's level (1..8, ≤1 → 1);
  `n = party + tdiv(gamePlayers − party, 2)`, `n = min(n, monster stat 100 monster_playercount)`; gamePlayers =
  max(real players, `/players` in SP game types 1–3) (0x6FC579A0). If n > 1:
  `r = nd/(nd+total)`, `NoDrop' = trunc(total·(1−x)/x)`, x = 1 − r^n (x87 doubles, repeated multiply).
  Online full party: n = players. Single player with /players P: n = 1 + ⌊(P−1)/2⌋ (P7 = P8 = 4).
  Example `Map H2H t3` (NoDrop 95, total 903): n=1 → 95, n=2 → 8, n≥3 → 0.
- **At most 6 items per TC call** (max defaults to 6); gold counts. Entry `mul` (gold): `gold = gold·mul >> 8`.
  Gold find 0x6FC2F260 (`max(0, tdiv(g·(100+gf),100))`) runs after the cap check (so not for a 6th item).
- Per item: forced unique/set entries (flags 1/2) skip the roll (none in PD2 data); else the quality roll.

### 3. Quality roll — D2Game 0x6FC2FC40 (VERIFIED)
- Early returns without RNG: ItemTypes `Normal` → normal; Items `unique` flag → unique; ItemTypes `Magic` and Items
  `quest` → unique.
- ItemRatio row (#10560): highest Version ≤ 100 with Uber = (weap/armo and code = ubercode/ultracode) and
  Class Specific = ItemTypes `Class` set → the Version 1 rows.
- For unique, set, rare (only if ItemTypes `Rare`), then magic:
  `c = (Ratio − tdiv(ilvl−qlvl, Div))·128`; with MF ≠ 0: `c = tdiv(c·100, 100+effMF)` (effMF: unique 250, set 500,
  rare 600 diminishing, magic none: `mf ≤ 10 ? mf : tdiv(mf·f, mf+f)`); `c = max(c, Min)`; `c −= c·tc/1024`
  (tc = the chain's word); success if `c ≤ 0 or rand(c) < 128`. MF ≤ −100 skips all four.
- **Correction to mf_misc.md:** for ItemTypes with `Magic`=1 (rings, amulets, charms, jewels…) a failed rare check
  returns **magic** directly (0x6FC2FED2) – the magic check and its RNG call are skipped.
- Superior: `(HiQ − tdiv(Δ, HiQDiv))·128`; normal vs low: `(Normal − tdiv(Δ, NormalDiv))·128` (c ≤ 0 → normal).
- MF = stat 80 of the killer (+ its owner). PD map mod `map_play_magicbonus` (Divide 1080 → player stat 80) and
  `map_play_goldbonus` (→ stat 79) simply add to MF/GF.

### 4. Item creation (READ; unique/set pick VERIFIED)
PD wraps item creation (0x6FC32A0C → 0x102EE5C0 → 0x102C89D0), then stock 0x6FC31490 → 0x6FC31070 / 0x6FC30A20.
- PD: ItemTypes `map` (row 105) → quality normal (0x102C8A54).
- Stock adjustments (0x6FC30AB6): type `Magic` → at least magic (quest item → unique); type without `Rare` and quality
  rare → magic; Items `unique` → unique; type `Normal` → normal.
- **Unique** (0x6FC2F370, VERIFIED): UniqueItems in table order with `enabled`, code match, `lvl ≤ ilvl`, `ladder`
  only in ladder/realm games (game+0x6A/+0x74); weight = `rarity` — **PD2 byte patch 0x6FC2F5CE removes the min-1 clamp**
  (rarity 0 uniques get weight 0, but a lone candidate is still picked when the total is 0: `rand(0)` = 0);
  `roll = rand(total)` on the item seed, pick the last candidate with start ≤ roll. **Once per game:** a pick whose bit
  is set in game+0x1B24 fails; 0x6FC2EDC0 sets the bit (no PD2 unique has `nolimit`). Failed unique → rare with ×3
  durability (0x6FC30D64), or magic when the type cannot be rare. `newGame()` holds this state; share it over a map run.
- **Set** (0x6FC33C20, VERIFIED; PD wrapper 0x102D7210 only handles TC-forced indices): `lvl ≤ ilvl`, code match,
  set 29 (Cow King's) only with drop flag 1, weight `rarity` (0 → 1), no per-game limit. Failed set → magic (×2 dur).
- Rare/magic/superior always succeed (affixes are not generated by drops.js).
- Gold: `amount = ilvl + rand(5·ilvl)` (0x6FC31085, item seed), then mul and GF as above.
- Ethereal (0x6FC2EBB0): weapon/armor with durability, not low quality, not quest: `rand(100) < 5`; PD NOPs the set
  exclusion (0x6FC2EBF7) so sets can be ethereal. PD `map_glob_dropethereal` (global 13): C `rand()%100 < v` sets drop
  flag 4 = forced ethereal (0x102C8BDA).
- Sockets (0x6FC2EC90, normal/superior expansion items): max = min(Items `gemsockets`, ItemTypes MaxSock1/25/40 by
  ilvl ≤25/≤40/>40, difficulty cap 3/4/6), `rand(100) < 33` → 1..max.
- PD map-only post steps (0x102C8D04..0x102C8D69): `map_glob_dropcorrupted` (global 6) corrupts, `map_glob_dropsocketed`
  (global 11) adds 1–2 or 2–4 sockets (0x102D3050). APPROX: which bases qualify, socket range per base.

### 5. PD2 extras in the death handler 0x102C11F0 (READ)
Map level = level id 137–201 (0x102CE890; note the simulator's level 203 is outside this range in this binary; pass
`inMap: true` to override). Order: specials, extra full drops, map-stat drops, then the main drop. `pdRand` = PD's
LCG variant 0x102C5D10 on the monster seed.
- **Specials** (0x102C5480): TC `Unique` = U, `Set` = S (the TC rows are data only); U' = ⌊U·T[p]/127⌋ with
  T = [·,127,125,123,121,119,117,115,113] for p = 1..8 players (0x10376CA0); drop if `pdRand % (U'·max(S,1)) < mult`
  (mult 2 in skirmish for non-bosses). Hell: puzzlebox `lbox` 1/168,800, puzzlepiece `lpp` 1/67,500, `rkey`/`rtp`/`rid`
  1/5,000,000, `llmr` 1/25,000,000, `lsvl` 1/7,500,000, Demonic Cube `imrn` 1/1,000,000; T1/T2/T3 map levels:
  Sigils `ubaa`/`ubab`/`ubac` 1/5,440 (sets 0x104E30B4/31E4/30A4); map levels: unique map 1/56,250 (bosses 1/3,000),
  map chosen by `pdRand & 7` (0x102D2EA0, APPROX order). Not modelled: Rathma jawbone / `lucd` (unknown gates),
  map-boss essences (`dcbl`/`scrb`), wss events.
- **Extra full drops** (each = the whole main drop again): skirmish (monster stat 493), **every monster in T4 map levels**
  (set 0x104E3018 = 152,153,164,165,171,172,186,187), levels 197–199, and `map_glob_dropbonus`: global id 12 becomes
  monster stat 273 on map monsters (zone builder 0x102DC982), extra drop if `pdRand%100 < v`.
- **Map-stat drops**: monster stats 494/495/496/497/502/506 (`map_mon_drop*`, applied to map monsters) → TCs
  `MapStat DropJewelry/Weapons/Armor/Crafting/Charms/Jewels` (0x102D0FC0 index + base of `Gold`), chance
  `pdRand%100 < v·mult`, via 0x6FC32380 without upgrade.
- Map mod lists: map item stats split by ItemStatCost `Divide` (0x102D4E70): 1024/<1000 → monster stats, 1xxx → player
  stat xxx, 2xxx → global id xxx (game+0x2632 list, 0x102DBA30), 3xxx special.

## API

```js
PD2Drops.setData(await (await fetch('drops-data.json')).json());
const game = PD2Drops.newGame();                    // one per map run (unique-once-per-game state)
const drops = PD2Drops.monsterDrops({
  monId: 'mummyIce', role: 'unique',                // roles of the simulator: ordinary|companion|minion|champion|berserk|unique
  difficulty: 2, areaLevel: 87, /* mlvl override */ players: 1, partyNear /* default players */, singlePlayer: false,
  mf: 300, gf: 100, levelId: 146, tier: 'T1', inMap /* default: 137<=levelId<=201 */,
  mapMods: { dropbonus, dropjewelry, dropweapons, droparmor, dropcrafting, dropcharms, dropjewels,
             dropethereal, dropsocketed, dropcorrupted, magicbonus, goldbonus, arealevel, skirmish },
  game, seed /* or rng: PD2Drops.makeRng(lo, hi) */, specials: true, tc /* override */, isMapBoss });
// -> [{code, name, baseName, type, category, quality, uniqueName?, setName?, ilvl, ethereal, sockets, gold?, corrupted?, source}]
PD2Drops.tcDrop('Map Good t3', { ilvl: 89, mf: 0, players: 1, seed: 1 });
PD2Drops.makeRng(seed)                              // game LCG: lo=seed, hi=666; rand(n), pdRand(), step()
```
`quality`: low|normal|superior|magic|rare|set|unique for equipment, jewelry, charms and jewels; rune|gem|potion|gold|misc
for the rest (`category` gives weapon|armor|jewelry|charm|jewel|rune|gem|potion|gold|map|special|misc).
Seeds passed as numbers are scrambled (`mix32`) because the LCG multiplier is divisible by 3·5·31 and consecutive small
seeds would correlate; the game's own monster seeds are well mixed. Item-internal randomness (gold amount, unique pick,
ethereal, sockets) uses a second stream derived from the monster seed (the game uses the item's own seed).

Simulator integration: for position i, `monId = current.species[positions[i][2]].id`, `role = current.roles[i]`,
`{level_id, tier, area_level} = chosen()`; call `monsterDrops({monId, role, areaLevel: area_level, levelId: level_id,
tier, players, mf, gf, mapMods, game, seed: runSeed ^ i})`.

## Test output (test_drops.js, 20,000 kills, Royal Crypts T3, mlvl 89/92, 1 player)
| case | items/kill | unique | set | rare | magic | rune (per kill) |
|---|---|---|---|---|---|---|
| normal, MF 0 | 0.91 | 0.0008 | 0.0001 | 0.0034 | 0.035 | 0.0034 |
| normal, MF 300 | 0.91 | 0.0016 | 0.0010 | 0.0098 | 0.092 | 0.0034 |
| unique, MF 0 | 5.00 (4 potions) | 0.0084 | 0.0089 | 0.067 | 0.88 | 0.0064 |
| unique, MF 300 | 5.00 | 0.0128 | 0.0227 | 0.164 | 0.77 | 0.0064 |

## Known simplifications
- Affixes, durability, defense/damage, item seeds and item-level-dependent prefixes are not generated.
- The main-drop and extra-drop RNG order follows the code, but the item-internal stream is not the game's item seed.
- PD `dropsocketed` / `dropcorrupted` eligibility and ranges, unique-map selection order, Rathma/Lucion/boss essence
  drops, quest TCs, SuperUnique TCs for map bosses (pass `tc`), and PD's random stat 437 on some uniques (0x102C8C2F)
  are approximated or omitted.
- `specials` apply to every kill (the code has no map gate for them apart from Sigils and unique maps).
