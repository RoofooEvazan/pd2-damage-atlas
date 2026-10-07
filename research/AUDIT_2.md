# Audit 2: independent re-check of 13 write-ups

Auditor: a separate agent that did not write any of the audited work. Date: 26 Sep 2026.

**Method**
- Every harness was rebuilt from its `.c` source with the command in the write-up (or in the harness header), then run. Mutation runs were also repeated.
- JS models were run for the worked examples.
- Spot-checks used `dis/*.asm`, the memory images and `excel_mpq/` (the live MPQ tables). The zip was read with `T()` only to compare.
- No JS, harness or `site/` file was edited. Rebuilding replaced the harness binaries with builds from the same sources.

**Verdicts at a glance**

| write-up | verdict |
|---|---|
| pierce_ow.md | solid |
| whirlwind.md | solid |
| procs_cooldowns.md | solid |
| shrines_movement.md | minor issues |
| monster_ai.md | solid |
| experience.md | solid (label nit) |
| mercenaries.md | minor issues |
| maps.md | **problems** |
| items.md | solid |
| cube.md | solid |
| vendors.md | solid |
| data_sources.md | minor issues |
| bh_panel_review.md | minor issues |

---

## 1. Per write-up

### pierce_ow.md — solid
| check | result |
|---|---|
| `pierce_ow.py` (binary rebuilt from `pierce_ow.c`) | roll_pd 9,899 / roll_stock 10,101 / step 5,000 / rand 3,000 / OW 11,062: **all 0 mismatches**. Thresholds `pd 0→0 67→1 87→5 95→10`, LF cap 6, stock 67→4 |
| `pierce_ow.py mutate` | every group fails (145 / 2,746 / 669 / 2,117 / 1,611) |
| spot: Pierce skill values | MPQ Skills.txt Pierce EMin 20, EMinLev 2/2/2/1/1. That gives lvl 20 = 58, lvl 22 = 62, lvl 27 = 67, lvl 47 = 87, lvl 55 = 95. Confirmed |
| spot: player draw sequence | The harness rand model (matched natively on 3,000 cases) gives 66, 86, 72, 39, 45, 94, 66, 56, 33, 83 from {0, 666}, as written |
| spot: OW example | 3716 per proc = levelScale(90) 2691 + 25 + 5·200. 3716·25/256 = 363 life/s. Consistent with `damage.md` / `bh_panel_review.md` |
| wording | "11,062 random procs" is really 11,062 proc **attempts**, of which 5,545 are actual procs. Note added |

### whirlwind.md — solid
| check | result |
|---|---|
| `ww.c` + `ww.py` | A rate 20,000 / B timeline 6,000 / C picker 20,000: **0 mismatches** |
| mutations | wwrate 12,346; dual5 1,498; first1 6,000; records37 6,000; gate_every_event 5,906; radius 9,909; nolast 945. All exactly as claimed |
| `ww_check.js` | timeline 6,000 / picker 20,000, 0 mismatches |
| worked examples (`whirlwind.js`) | M = 5/10/25/50/100 gives 1/2/5/10/20 hits (dual 2/4/8/18/34) and END 11/16/31/56/106. The M = 50 schedule is frames 3…48, stop 52, END 56. The forced-rate table (s = 15…175) and the radius table (rangeadder 1/2/3/4 → 3/5/6/8) are reproduced. CB 10.8 %/s = 1−(1−0.0227)^5 |
| spot: data | MPQ: Whirlwind UseAttackRate blank, Param3 = 5, velocitypercent `min(ln56, 65)`. Colossus Blade speed 5, rangeadder 3. Phase Blade −30. Barbarian WalkVelocity 6. Confirmed |

### procs_cooldowns.md — solid
| check | result |
|---|---|
| `procs.c` (11 modes × 40,000) | all modes **0 mismatches**; every mutated mode fails ("ALL OK") |
| `procs_rng.c` → `procs.js` | The write-up names no JS checker, so I wrote a throwaway one in the scratchpad. `pdRand`, `d2Rand` and `rand(100)` match all 20,000 native seeds (0 mismatches) |
| spot: cooldowns | MPQ Skills.txt `delay`: Joust `max(104 − lvl·5/2, 38) − reductions`, Gust `max(163 − (lvl·5 + gustreduction), 13)` → 41 and 13 frames. Matches `procs.js DELAY` |
| spot: replenish | 1 + ⌊2500/33⌋ = 76 frames, as written |

### shrines_movement.md — minor issues
| check | result |
|---|---|
| `shrine.c` + `shrine.py` | kind 0: 120,000 / kind 1: 40,000 / kind 2: 40,000, **0 mismatches**. Mutate: 5,995 / 2,905 / 17,055 (as claimed) |
| "native Monte Carlo 3×100k within ±0.2" | **not present** in the shipped `shrine.py` / `shrine.c`, so it cannot be re-run. `mf_misc.js shrineTypeChances` reproduces the §1.2 table |
| spot: PD2 stamina shrine | PD 0x102C7B60: `SetStat(list, 0x43 = 67, 0x23 = 35, …)`, then a tail-jump to the original SetStat (28 = 1000). Confirmed |
| spot: exploding/poison shrine count | D2Game 0x6FC8C598–0x6FC8C5E9: `arg0 + rand(arg1 − arg0)`, i.e. 5–9. Confirmed |
| **issue**: Experience shrine example (§1.5) | "941 → 1411" leaves out ExpRatio (`mf_misc.expGain` defaults to 1024). The real values for a level-80 player are **455 → 682**. Correction added |

### monster_ai.md — solid
| check | result |
|---|---|
| `aipick.c` + `aipick.py` | 50,000 cases, **0 mismatches** (the RNG state is compared too) |
| mutations | dist 9,974; aidel 1,175; zlvl 605; idle 363. The write-up has 9,977 / 1,134 / 570 / 363. The drivers do not fix the random seed, so these counts change from run to run; each mutation still fails |
| `check_js.js` | 40,000 cases, 0 mismatches |
| spot: distance 0x6FCD08F0 | `(min + 2·max) >> 1` = max + ⌊min/2⌋. Confirmed |

### experience.md — solid
| check | result |
|---|---|
| `node exp_check.js 50000` | 50,000 cases, **0 mismatches**: 36,572 kill records, 9,372 splits, 12,801 merc awards (as claimed) |
| mutations at 20k | party89 3,793; range 453; merc86 2,747; gate 146; death 451; champ 976; players 2,000; zero1 96; double 19. **Identical** to the table |
| worked example | `gainXp(51195, clvl, 85)` = 40,183 / 23,338 / 12,798 / 5,399 / 3,049 / 38. Matches §1.6 |
| spot: Q4 Uber cap | `uberdiablonew` Level 110, Exp(H) 9000 × MonLvl 110 XP(H) 160,000 / 100 = 14,400,000. Capped share at clvl 95 → 106,110 (the 64-bit ExpRatio path `(x >> 10)·15`). Confirmed |
| label nit | "`pd2data.mpq` == `data.zip`" is not true in general, though it is for every table this page uses. Correction added |

### mercenaries.md — minor issues
| check | result |
|---|---|
| `merc.py` (builds `merc.c`) | revive 2,100 / hire 20,000 / xp 20,000 / leech 20,000 / skill 39,938: **0 mismatches**. Mutations as in the table |
| `mercpd.py` | equip 19,680 / class items 20,000: **0**. Mutations 348 / 57 / 61 / 310 |
| data source | Both drivers read `data.zip`. hireling.bin and skills.bin are identical to the MPQ copies. `mercpd.py` uses zip Weapons/Armor/**Misc**.txt; Misc differs from live only in maps, ears and jewelry sockets, none of which are merc equipment. Results stand |
| spot: hire/revive | MPQ Hireling A2 Hell Level 75, Gold 15,000 → 48,750 at level 90. Revive L50 18,750, L75 42,180, cap 50,000 from L82. Confirmed |
| **issue** | §7 names stock 0x6FCFEA00 as the kill handler. PD2 uses PD 0x102CA4B0 (`experience.md`, VERIFIED). The rules agree. Correction added |

### maps.md — problems
| check | result |
|---|---|
| `mapapply.c` | 20,000/20,000 match. Mutations: 17,848 and 15,066 matches (as claimed, written as match counts) |
| `mapaffix.c` | rare count 30,000/30,000, force flag 30,000/30,000. Mutation 29,145 / 25,027 |
| spot: T1 set | PD 0x10126E30 builds 0x104E30B4 from 0x1037FE10/0x1037FE40 = {143, 145, 146, 148, 151, 155, 158, 169}. That equals the MPQ `t1m` maps (t11, t21, t22, t24, t26, t33, t34, t36) and TC "Map Tier 1". Confirmed |
| **problem 1**: §2.3 "r == total picks nothing" + §2.2 "5.8–5.9 affixes" | Wrong. D2Game 0x6FC3481E–0x6FC34853: when the walk ends, `edi` still holds the last candidate, and it is taken. `items.md` proves this natively: probe, and 542 mismatches for the "picks nothing" mutation. `items.js` gives 6.00 affixes on 20,000 ilvl-88 rare t11/t13 maps. `maps.js pick()` carries the wrong rule. Corrections added |
| **problem 2**: §4 corruption and §1.1 "1..1000" | Taken from the stale zip. The live MPQ row 341 rolls **1..3000**, and the thresholds are 270…2970, then the unique maps up to 3000. Phase 2 is the stock op-16 test, `stat ≤ value` (the D2Game jump table 0x6FC90B34 → 0x6FC9060F fails when stat > value), not "roll < value". The T4 rows stop at 1000 → 3.33 % each with 2/3 duds, not 10 %. Corrections added |

### items.md — solid
| check | result |
|---|---|
| `itemaffix_driver.js 30000` | magic / rare / value 10,000 each, **0 mismatches** |
| mutations | noforce 3,449; nocap 542; alvl 3,686; weight 3,405; range 6,208 (all as claimed) |
| `itemfull_driver.js` / `itemstaff_driver.js` / `itemauto_driver.js` | 20,000/20,000 each. Staff mutations req 8,495 and lvl 14,475 mismatches (as claimed) |
| `itemaffix_probe.js` | circlet W = 3411: r = W returns the last candidate, r = W+1 the first |
| worked examples (`test_items.js`, 10^6) | amulet 14.705 / 2.111 / 44.08 / 20.93 %; diadem 29.77 / 4.26 %; jewel 5.93 %; GC 4.93 / 4.55 / 9.05 / 9.01 %. All within CI. Circlet ilvl 60 "+1/+2 class" 34.50 % against the written 34.3 % (a small Monte-Carlo drift) |
| spot: draw | disassembly as above; confirmed |
| spot: "of Anima" | MPQ MagicSuffix itype4 = `amu` (a typo for `amul`). Confirmed |

### cube.md — solid
| check | result |
|---|---|
| `cube/cube.c` | 40,000/40,000. Mutations 32,415 / 39,247 / 34,085 matches (as claimed). `engine_vectors.txt` is byte-identical after the rerun |
| `cube/craft.c` | 40,000/40,000. Mutations 39,291 / 28,037 matches (as claimed) |
| `node test_cube.js` | engine vectors 20,000/20,000; distribution sums checked |
| spot: corruption rows | MPQ lines 341 (`map + wss`, corruptnum 1..3000), 343–345 disabled, 347… thresholds 270…2970, uniques up to 3000, T4 lines 2077–2086 at 100…1000. Confirmed. Zip: 1..1000, thresholds 90…990 |
| spot: op 16 | stock D2Game jump table: op 16 → 0x6FC9060F, which fails when stat > value. So the test is ≤, as in §1.4 |
| worked examples (`cube.js corrupt`) | bow: brick 25 %, sockets 7.5 / 7.0 / 6.0 / **4.5 %**. T1 map: 11 × 9 %, uniques 0.10–0.133 %. Confirmed |
| nit | "zip copy (2,300 rows)" should be 2,302. Note added |

### vendors.md — solid
| check | result |
|---|---|
| `gamble_check.py 30000` | 420,000 slots, **0 mismatches**. Mutate (2,000): 1,957 mismatches (as claimed) |
| `gamble_check_js.js` (after a 20,000-window native run) | 280,000 slots, 0 mismatches |
| `price_check.js 40000` | **0 mismatches**. Mutations superior 1,002, cap 3,567, eth 842. The write-up says 1,003 / 3,568 / 843, off by one each (cosmetic) |
| worked examples (`examples.js`) | Every price in §1.7 and §3.3 is reproduced (Circlet 109,818; Coronet 162,976 → 146,679 at −10 %; Anya items; Shako 61,471 / 30,735 / 7,683; superior 37,500) |
| spot: data | MPQ DifficultyLevels GambleUnique 50 / Set 100 / Rare 10000 / Uber 90 / Ultra 33. Ring uniques' rarity sum is 60 (SoJ 1). ci0 has ubercode ci1 and no ultracode. Confirmed |

### data_sources.md — minor issues
| check | result |
|---|---|
| native checks | none claimed (correct: no VERIFIED labels) |
| spot: tiers | MPQ `t1m` = t11, t21, t22, t24, t26, t33, t34, t36; zip `t1m` = t12, t15, t17, t22, t23, t25, t26, t36; 22 type/calc1 differences; t57/t58/t3b only in the MPQ. Confirmed |
| spot: PD T1 set | confirmed (see maps.md) |
| spot: CubeMain | MPQ 2,341 rows, zip 2,302 |
| **issue** | It says "maps.md §corruption already uses the MPQ" and that maps.md used the MPQ. §4 of maps.md is zip-based. Corrections added. Also, its "rebuild commands … I did not run these" is now out of date: `drops-data.json`, `monsters.json`, `engine-data.json` and `site/hitcalc-data.json` were all rewritten at 00:12 on 26 Sep, and the drops data carries the live tiers |

### bh_panel_review.md — minor issues
| check | result |
|---|---|
| native checks | none of its own. It uses "TESTED" (not VERIFIED) and leans on other harnesses. No harness-less VERIFIED claims |
| spot: open wounds 1.4 | BH (2691+25)·25 >> 8 + 200 = **465**; game 3716·25/256 = **363**/s. Confirmed |
| issues | §5 says the XP % line was not checked. `experience.md` §6 has since checked it (note added) |

---

## 2. Cross-document conflicts (settled)

| conflict | ruling | evidence |
|---|---|---|
| maps.md §4 corruption (zip, 1..1000) vs cube.md §2.5 (MPQ, 1..3000) | **cube.md is right.** Live: roll 1..3000 for all maps. T1–T3: 11 × 9 % + 8 unique maps = 1 %. T4: 10 × 3.33 %, 66.7 % no effect. The comparison is stock op 16 `≤` | MPQ CubeMain lines 341, 347–365, 2077–2086; D2Game 0x6FC9060F |
| maps.md "weighted pick picks nothing" vs items.md "never fails" | **items.md is right** | D2Game 0x6FC3481E–0x6FC34853; `itemaffix_probe.js`; mutation `nocap` 542 mismatches |
| crit_cb.md old Whirlwind model vs whirlwind.md | **whirlwind.md is right.** crit_cb's rate bullets were already marked superseded, but its "Do-func" bullet still gave the records-3-and-7 / IAS model | ww.c A/B/C rerun, 0 mismatches; mutation `records37` fails 6,000/6,000 |
| drops.md on zip map tiers vs data_sources.md | **data_sources.md is right** about the tiers and about `t3b`. TreasureClassEx is identical in both copies, and the "Map Tier N" TCs list live-tier codes. So only the item→tier labels and the missing `t3b` (Map Tier 3 = 9 maps live, 8 in zip) were affected. `drops-data.json` has since been rebuilt with live data; drops.md's text was not updated | MPQ TreasureClassEx; drops-data.json contents |

**Other contradictions found**
- `dmg_B_pipeline.md` (event 7 table row) says on-attack 195 fires on every melee attempt, hit or miss. `procs_cooldowns.md` mode 11 (VERIFIED) shows it fires only on hits. Note added to dmg_B_pipeline.md.
- `shrines_movement.md` Experience-shrine example vs `experience.md`: the ExpRatio was left out. Note added.
- `mercenaries.md` §7 (stock kill handler) vs `experience.md` §1.4 (PD 0x102CA4B0). Note added.
- `bh_panel_review.md` §5 is out of date against `experience.md` §6. Note added.
- `maps.js pick()` still returns null on r == total. It disagrees with `items.js` and gives maps a 5.8–5.9 affix mean. JS was not touched.
- `AGENT_CONTEXT.md` still says `T()` "picks the right one". It reads `data.zip` by default (`data_sources.md` issue 2). The file is outside `adv/re/` and was not edited.

**VERIFIED without a backing harness**
- `shrines_movement.md` §1.2 / §4: "native Monte Carlo 3×100k within ±0.2 points". There is no such mode in the shipped `shrine.c`/`shrine.py`.
- Everything else labelled VERIFIED in the 13 write-ups was re-run and matched.

---

## 3. Corrections added (all formatted `> **Correction (audit):** …`)

1. `maps.md` §1.1 (stored stats): stat 361 roll is 1..3000 live, not 1..1000.
2. `maps.md` §2.2: rare maps always get 6 affixes; the 5.8–5.9 figure comes from the wrong `maps.js pick()`.
3. `maps.md` §2.3: r == total takes the last candidate; the pick does not fail.
4. `maps.md` §4: corruption table is the stale zip version. Live: 1..3000, op 16 ≤, T1–T3 9 % each + 1 % unique maps, T4 3.33 % each with 2/3 duds; see cube.md §2.5.
5. `crit_cb.md` "Do-func" bullet: event timing superseded by whirlwind.md (rate 256, do-func every frame).
6. `drops.md` §2: `t3b` is valid live; Map Tier 3 has 9 maps; drops-data.json is now rebuilt from live data.
7. `data_sources.md` §4.1: maps.md §corruption did not use the MPQ.
8. `data_sources.md` §5: same, plus drops-data.json has since been rebuilt.
9. `shrines_movement.md` §1.5: Experience shrine example is 455 → 682, not 941 → 1411.
10. `experience.md` status key: mpq ≠ zip in general; the tables used here are identical.
11. `mercenaries.md` §7: kill handler is PD 0x102CA4B0.
12. `bh_panel_review.md` §5: XP % line now covered by experience.md §6.
13. `dmg_B_pipeline.md` (on-event table): 195 fires only on melee hits.
14. `pierce_ow.md` §2.2: 11,062 are attempts; 5,545 actual procs.
15. `cube.md` header: zip CubeMain has 2,302 rows.

No status labels needed changing.

---

## 4. What should be redone

1. **`maps.js`**: fix `pick()` so that r == total returns the last candidate, as `items.js` does. Then regenerate the rare-map affix mean (expect 6.00).
2. **`maps.js corruptionOutcome` / `CORRUPT`**: these are documented as taking "a corruptor roll 1..1000" (maps.js line 14), i.e. the zip table. Rebuild them from `cube.js` / MPQ CubeMain (1..3000, op 16 ≤, T4 duds), or have the site use `cube.js corrupt()`.
3. **`maps.md` §4**: rewrite from `cube.md` §2.5, beyond the correction note.
4. **`drops.md` header and `drops-data.json` `generated` string**: the data is now live, but both still say data.zip. Re-run `test_drops.js` expectations for Map Tier 3 (9 maps).
5. **`shrine.py`**: add the Monte Carlo mode that §1.2 cites, or drop the claim.
6. **`aipick.py`**: fix the random seed so that the mutation counts are reproducible.
7. **`AGENT_CONTEXT.md`**: correct the "T() picks the right one" line (it reads the zip unless `source='mpq'`).
8. **`crit_cb.md` / `engine.js`**: "Still not modelled: the Whirlwind rate" in crit_cb §3 may be stale (bh_panel_review says the page now uses the fixed rate). Confirm and update.
