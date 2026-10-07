# PD2 chance to hit, block and avoid: the full pipeline

PD2 is Diablo II 1.13c (`D2Game.dll`, `D2Common.dll`) plus `ProjectDiablo.dll` (PD). All of this comes from the game code.

- Code: `adv/re/hit.js` (`window.PD2Hit` / `module.exports`).
- Test: `node adv/re/test_hit.js`.
- Native harness: `harness/hitpd.c` + `harness/hitpd.py`.

**Status key**
- **VERIFIED**: PD2's own code ran natively in `harness/hitpd.c` and matched the model on every case. Only its D2Common/D2Game imports were stubbed, through PD's lazy-import slots.
  - 30,000 random hit-roll cases, 0 mismatches. This covers AR, defense, the chance, the hit/miss result, the post-hit call and the RNG state after the roll.
  - 8,000 block/avoid cases, 0 mismatches. This covers the result flag and the RNG state.
  - `hit_vectors.json` (2,534 of those cases) is replayed through `hit.js` by `test_hit.js`, 0 mismatches.
- **READ**: read from the disassembly only.

`tdiv` is C integer division, which truncates toward zero. "stat N" is the unit's total stat (D2Common #10973 / #10910; both give the same total).

---

## 1. Who calls the roll

### PD2 routes every normal hit check through its own code

**Melee resolver PD 0x10270EB0** replaces stock `0x6FCFE5A0`.
- PD2 builds patch records at runtime (`0x100F9839…`, `{3, rva, 0x102EEDA0, …}`). They redirect **31 of the 34** stock `call 0x6FCFE5A0` sites through wrapper 0x102EEDA0, which turns the register arguments into stdcall.
  - Callers include normal attack (srvdofunc 1), Jab, Frenzy, Fend/Zeal/Fury, Whirlwind, Double Swing, Dragon Talon/Tail, Leap, the monster charge/leap/frenzy skills, Smite and more.
- The three stock sites it leaves alone:
  - `0x6FC6F872`: srvstfunc 16. No skill in PD2's Skills.txt uses it.
  - `0x6FC97788` and `0x6FCC1185`: no references were found to the code that contains them.
- So the stock roll `0x6FCFDE90` is not reachable in practice (READ).
- PD skill functions call 0x10270EB0 directly, passing D2Common #10653 (skill ToHit) as the to-hit argument:
  - Joust (srvdofunc 162)
  - Poison Dagger (164)
  - Tiger Strike / Fists of Fire / Cobra Strike / Claws of Thunder / Blades of Ice / Royal Strike (170). These pass #10653 + stat 325 `progressive_tohit`.
  - Vengeance (174)

**Missile collision PD 0x102723E0** replaces stock `0x6FC5EDE0`. PD redirects all 12 of its callers through wrapper 0x102F1010.
- It rolls only when the Missiles.txt `ToHit` column (+0x18B) is set. That is 108 of the 1,057 missiles: arrows, bolts, javelins, throwing weapons, and some monster/trap missiles.
- Every other missile hits without an AR roll.
- The roll is `0x10271740(owner, target, 0x10268220(missile, owner), isMissile = 1)`.
- **0x10268220** returns the owner's `passive_mastery_throw_th` (stat 345, layered by weapon type through 0x102D30B0) + #10653(owner, missile skill = MissileData+0x0A, level = +0x0C).
  - Stock passed the missile's own stat 19 instead (READ).

**Area callback PD 0x1026F8F0** is used by:
- Leap/Leap Attack (153)
- proc_SplashDamage, Golem Splash and Skeleton Splash (163)
- the Blade Creeper AI (MonAI 102, PD 0x102B9820)

It rolls only if the per-call flag is set. Splash sets it to 0, so **splash always hits**. Leap Attack sets it only for skill 143. Blade Creeper sets it to 1.

Its to-hit argument is `stat119(attacker) + skill ToHit`, but the roll adds stat 119 again. **Leap Attack and Blade Creeper therefore count your AR% twice** (READ).

**Blade Shield** (srvdofunc 54): stock callback 0x6FCB3040 went through 0x6FCFE5A0, which includes the range check and block. PD replaces it at `0x6FCB35BC → 0x1026FFE0`, which calls the roll only: **no block, no range check** (READ).

**Smite** (srvdofunc 150, stock 0x6FCB9000, player branch) ORs the hit bit into the resolver result. So the AR roll never makes Smite miss (READ).

### Resolver 0x10270EB0(defender, attacker, game, toHit, range)
(Corrected in the audit: wrapper 0x102EEDA0 pushes stock eax = defender first and edi = attacker second. The resolver calls the roll as 0x10271740(attacker, defender, toHit, 0).)
1. Only players or monsters, on either side.
2. PD 0x102ECE30: the can-attack check (stock 0x6FD00A60).
3. Melee range check PD 0x10270FF0. Range = arg + 2, plus unit-size terms (READ, not detailed here).
4. **There is no "running player is auto-hit" branch.**
   - Stock 0x6FCFE5A0 set the hit without a roll when the defender was a player in mode 3 (run).
   - In PD2, running players are rolled against like anyone else (READ).
5. Hit roll 0x10271740(attacker, defender, toHit, 0) (**VERIFIED**).
6. Then block/avoid 0x1026FB60(1, attacker, defender, game, 0) (**VERIFIED**).
   - Result flags: block → 0x201 (hit cleared); weapon block 0x10 → |0x8000; dodge/avoid/evade → hit cleared.
   - If the hit stands and the defender lacks state 54, `|4`.

## 2. The hit roll PD 0x10271740(attacker, defender, skillToHit, isMissile) (VERIFIED)
1. dLvl = stat12(defender). aLvl = stat12(attacker).
2. **Def** = stat33 `armorclass_vs_hth` (or stat32 `armorclass_vs_missile` when isMissile) + D2Common #10672(defender).
   - #10672: stat31 + dex/4, × (100 + stat16 + stat171)%, then stat182 (stock function, VERIFIED earlier).
3. **Player attacker** (unit type 0):
   - AR = D2Common #10621 = stat19 + 5·dex − 35 + CharStats ToHitFactor (stock, VERIFIED earlier).
   - **0x10271620(attacker, defender, &AR, &Def)**, same rules as stock 0x6FCFB1F0:
     - stat115 `item_ignoretargetac`: Def = 0 if the defender is a monster with MonsterData, not superunique or unique (flags & 0xA), not a MonStats `boss` (#10064) and not a merc (#11104).
       - The two calls are lazy-import thunks: PD 0x10278580 → [0x104E8B50] = D2Common **#10064**(0, unit), which tests MonStats byte +0xC & gdwBitMasks[6] (0x40, the `boss` column). PD 0x102784F0 → [0x104E7950] = D2Common **#11104**(unit), the hireling class test (PD hook adds 1056). There is **no level test** (see the Complete factor list below).
       - Stock also passed a monster that had no MonsterData.
     - stat116 `item_fractionaltargetac` v > 0:
       - Halved (toward zero) against players, act bosses, superuniques (flag 2) and mercs.
       - v < 1 → 0, cap 100.
       - Def −= Def·v/100, using 64-bit division (__alldiv 0x10302E20). Stock took integer shortcuts for huge values; the results are identical in normal ranges.
     - stat123 `item_demon_tohit`: added to AR if #10255(defender) (MonStats demon). stat124 `item_undead_tohit`: added if #10239 (undead).
   - pct = **mastery** (not for missiles; see below) + stat119 `item_tohit_percent` + skillToHit + Σ stat179 `attack_vs_montype` values whose layer matches the defender's MonType.
     - The montype term only applies if the defender is a monster with MonStats MonType ≠ 0.
     - Matching goes through stock 0x6FC21620, the MonType equiv bit matrix.
   - **Mastery PD 0x102727D0(unit, weapon, 0, mode 0)**:
     - Only when the attacker has a weapon (D2Game 0x6FC572C0).
     - Reads stat **345** `passive_mastery_throw_th` when the weapon's ItemTypes.Throwable is set **and** the used skill's `itypea1` type is throwable. Otherwise it reads stat **342** `passive_mastery_melee_th`.
     - The value comes from 0x102D30B0, which sums the stat layers whose item type matches the equipped weapon, with a special case for 1han/2han grips.
     - Stock called #10804 (stat 342 only).
4. **Monster / merc / summon attacker**: AR = stat19 + 5·stat2 + skillToHit (flat); pct = stat119. Step 3's target adjustments are skipped (same as stock).
5. **% application PD 0x102CE840**:
   - bonus = (double)AR · pct / 100, truncated toward zero.
   - Above 2147483647 it becomes 0x7FFFFFFF; below −2^31, cvttsd2si gives 0x80000000.
   - AR = int32(AR + bonus).
   - In normal ranges this equals stock's `AR + tdiv(AR·pct, 100)`.
6. **Chance PD 0x10271470(Def, AR, dLvl, aLvl)**. All operations are 32-bit int, and the wraparound is modelled.
   - If Def < 0: AR −= Def, Def = 0. Then if AR < 0: Def −= AR, AR = 0.
   - S = AR + Def. ratio = tdiv(111·AR, S), or 100 when S = 0.
   - L = aLvl + dLvl. If L = 0 → 5.
   - c = tdiv(2·ratio·aLvl, L). If c < 5 → 5. If c ≤ 95 → c.
   - Above 95: ex = ratio − tdiv(95·L, 2·aLvl); if S ≠ 0, ex = tdiv(tdiv(S·ex, 111)·90, S); c = 95 + tdiv(2·ex·aLvl, L). Capped at 100.
7. **Roll**: r = pdRand(attacker) (PD 0x102C5D10). Hit if chance > r mod 100 (unsigned).
   - After a hit, PD 0x102EFAF0 calls stock 0x6FCFCE70. For a player attacker with stat117 `item_preventheal`, that applies state 52 `preventheal` to the monster, except classes 704–709 (the ubers).
   - Returns 1 on a hit, 0 on a miss.

### PD2 RNG PD 0x102C5D10 (VERIFIED; not the stock generator)
- Seed: lo = unit+0x20, hi = unit+0x24 (items use their own seed).
- p = lo·0x6AC690C5 (64-bit). new lo = hi + low32(p).
- new hi = low32(p ≫ 20) + carry, with carry = 1 when hi ≠ 0 and low32(p) > 0x7FFFFFFF − hi (unsigned). Returns new lo.
- Stock: new = lo·A + hi, keeping both halves of the 64-bit result.
- The returned value uses the same formula as stock. Only the next state differs.
- `block_regen.md` called this "same LCG"; that is corrected here.

## 3. Block and avoid after a hit: PD 0x1026FB60(flag, attacker, defender, game, isMissile) (VERIFIED)
All rolls use the **defender's** seed.

1. **Shield block** (only if flag ≠ 0).
   - b = D2Common #10212(defender, game+0x70 expansion). This is the character-screen value (VERIFIED earlier).
     - Player: (toblock + BlockFactor) × (dex − 15) / (2·clvl), **capped at 75**.
     - Monster: toblock.
   - If b < 1, go to the avoid chain.
   - Against monster class 1112 (`wraithMapMod`): b = tdiv(b, 3).
   - Blocked if pdRand % 100 < b → result 1.
   - **There is no moving/running penalty.** Stock divided by 3 for a running player.
   - There is no 90 cap. The "90 / PvP 75 / 80" caps in `defense.js` are **resistance** caps, not block caps.
2. **Avoid chain PD 0x1026FCF0**:
   - **Lockout**: a player defender whose playerdata+0x1C0 > game frame gets no avoid of any kind. This is set for 4 frames after any successful dodge/avoid/evade/weapon block.
   - **PvP**: defender is a player on level 157, 159 or 166 (PD 0x102CEAE0). Each chance below is halved (tdiv 2).
   - **Weapon block**:
     - w = stock 0x6FCFA540: the largest stat 348 `passive_weaponblock` whose param matches the item in body location 4/5, or the param-0 value.
     - If w > 0: w = min(w, 75); w ÷ 3 against wraithMapMod.
     - Only with weapon class 5 (2hs) or 13 (ht2).
     - Success if roll < w → 0x10. This comes before evade, even when moving.
   - **Moving** (player mode 2 walk / 3 run; monster mode 2 walk / 15 run): evade = stat340.
     - **If ≤ 0, the chain ends: no dodge or avoid either.**
     - Success if roll < evade → 8.
     - **On failure, PD2 goes on to dodge/avoid.** Stock gave a moving player evade only.
   - **Dodge / avoid**: stat338 `passive_dodge` (melee) or stat339 `passive_avoid` (missile). If ≤ 0 → 0. Success if roll < v → 4 (dodge) or 2 (avoid).
   - There are no caps other than weapon block's 75.

## 4. Monster numbers (READ)
- **Spawn level (0x6FCCFDB0)**:
  - Normal: MonStats Level.
  - NM/Hell: non-noRatio, non-boss monsters use Levels.txt MonLvl{2,3}Ex (area level). This includes PD2 map areas; their Levels rows are in `hitcalc-data.json` `areas`.
  - Mercenaries use difficulty 0.
- **MonUMod handlers**:
  - Table D2Game 0x6FD2E550. MonUMod constants are from MonUMod.txt; CDB = DifficultyLevels `ChampionDamageBonus` (record +0x34) = 90 / 75 / 66.
  - Uniques and superuniques (0x6FC448A0 with arg 1) and their minions (arg 0) get umods 1–4. Umod 4 `leveladd` (0x6FC41E80) is **level +3**, exp ×5.
  - Champion (umod 16, 0x6FC42DA0, arg 1): level −1 after leveladd, so **champions are +2**.
    - stat119 += tdiv(CDB·75, 100) = **67 / 56 / 49** (MonUMod constant 10).
    - damage% += CDB·100/100.
  - `strong` (5, 0x6FC42BC0):
    - unique: stat119 += tdiv(CDB·100, 100) = 90 / 75 / 66
    - minion: stat119 += tdiv(CDB·50, 100) = 45 / 37 / 33
  - `berserk` (39): stat119 += tdiv(CDB·300, 100) = 270 / 225 / 198.
  - `fanatic` (37): stat16 = −70, so **defense −70%**.
  - `stoneskin` (28, arg 1 only): stat31 × 2.
- **AR at attack time** (0x6FC97240): recomputed from the **current** level (the one after umods) with the attack's TH column (A1/A2/S1). Uses the "L-" columns in ladder / non-open games.
  - In NM/Hell with p ≥ 2 players: AR += tdiv(AR·f, 128), f = [0,0,8,16,24,32,40,48,56][p] or 8p − 16.
- **Defense**: scaled once at spawn (#11089) from the spawn level. The umods change the level afterwards, and nothing recomputes AC. This comes from the order of the code; the moninit path was not traced in full.

## 5. Differences from stock 1.13c (summary)
| Step | Stock | PD2 | Status |
|---|---|---|---|
| ratio | 100·AR/(AR+Def) | **111**·AR/(AR+Def) | VERIFIED |
| cap | 5..95 | 5..**100** (excess above 95 counts at 90/111) | VERIFIED |
| running player | auto-hit, no roll | **always rolled** | READ |
| AR% | int, 64-bit fallback | double, clamp 0x7FFFFFFF (same values normally) | VERIFIED |
| mastery | #10804 stat 342 | stat 342 **or 345** (throwable weapon + throw skill) | VERIFIED (selection) |
| missile to-hit arg | missile stat 19 | skill ToHit at impact + **throw mastery 345** | READ |
| RNG next state | 64-bit LCG | own variant (same output formula) | VERIFIED |
| block when moving | ÷3 while running | **no penalty**; ÷3 only vs wraithMapMod | VERIFIED |
| avoid order | moving: evade only; else weapon block → dodge/avoid | weapon block → evade → dodge/avoid; moving without evade → nothing | VERIFIED |
| weapon block | two claws only, no cap | cap 75, 2hs or ht2, PvP ÷2 | VERIFIED |
| Blade Shield | resolver (range, block) | roll only, no block | READ |
| Leap Attack, Blade Creeper | resolver | AR% counted **twice** | READ |

## 6. Harness notes
- `harness/hitpd.c` loads `PD.img` at 0x10000000 and pre-fills these lazy-import slots with stubs that read the case: 0x104E7100 / 0x104E814C (stats), 0x104E7D44 (#10672), 0x104E8784 (#10621), 0x104E8274 (weapon), 0x104E8B50 (#10064), 0x104E7950 (#11104), 0x104E7EA0 (#10255), 0x104E7524 (#10239), 0x104E8A24 (#10702), 0x104E8228 (#10511), 0x104E8968 (#11088), 0x104E781C (#10108), 0x104E4A38 (montype matrix), 0x104E5158 (post-hit), 0x104E7384 (#10212), 0x104E50E4 (weapon block), 0x104E877C (#10431).
- It also sets the CRT memset import 0x103233A4 and a fake sgptDataTables at [0x104E34B8] (MonStats +0x1C MonType, Skills +0x18 itypea1).
- It stubs the layered mastery reader 0x102D30B0 and the event call 0x102D0E50, and redirects the `call 0x10271470` at 0x102718DF through a recorder that captures (Def, AR, dLvl, aLvl).
- Not run natively:
  - the callers (resolver range check, missile path, area callbacks)
  - the layered stat readers
  - the MonUMod handlers
  - monster level/AR scaling (the scaler itself is VERIFIED in FINDINGS.md)

---

## 7. Complete factor list (audit)

Every input to a **player's** chance to hit a **monster**, in the order the game applies it. Machine-readable version: `adv/re/hit_factors.json` (31 entries). Calculator support: `hit.js` (new options are listed in §7.7).

### 7.1 Answer: does Ignore Target Defense depend on level?
**No.** PD 0x10271620 reads no level stat. The two calls in the ITD branch are:
- PD 0x10278580(0, mon): lazy thunk for D2Common **#10064**. It tests the MonStats `boss` bit (+0xC bit 6).
- PD 0x102784F0(mon): lazy thunk for D2Common **#11104**, the hireling class test.

They resolve through PD import tables 0x103C7868 / 0x103C65F8 and `0x102F10F0(module 1 = D2COMMON.dll, −ordinal)`.

ITD sets Def = 0 when all of these hold:
- the attacker is a player
- stat 115 ≠ 0 (any value)
- the defender has unit type 1 and MonsterData
- `MonsterData+0x16 & 0xA` = 0, meaning not unique (0x8) and not superunique (0x2); champions (0x4) and unique minions (0x10) **are affected**
- the MonStats `boss` flag is not set
- the defender is not a hireling class

This is **VERIFIED natively**: 106 eligible ITD cases in `hit_vectors.json` all gave Def = 0. 28 of them had monster level > attacker level, and 61 were champions or minions.

The MonStats `boss` flag covers more than act bosses: 115 rows in PD2. These include Andariel … Baal, Radament, Summoner, Blood Raven, Griswold, Nihlathak, Izual, the ubers, Dclone, Rathma/Mendeln, map bosses (e.g. `ArcaneBoss`, `CowBoss`, `Lucion`, `diabloMap`…) and Invader classes.

Level matters only through the chance formula. With Def = 0 the ratio is 111, so the chance is 100% while the monster level is at most about 1.18 × clvl:

| Your level | Last monster level at 100% | Chance at +10 / +20 levels |
|---|---|---|
| 50 | 59 | 99 / 92 |
| 70 | 83 | 100 / 96 |
| 90 | 107 | 100 / 98 |

`H.itdChance(clvl, mlvl)` returns these values.

### 7.2 Your attack rating
1. **Base AR** = stat19 + 5·Dex − 35 + CharStats ToHitFactor (#10621; VERIFIED).
   - ToHitFactor: Ama 5, Sor −15, Nec −10, Pal 20, Bar 20, Dru 5, Asn 15.
   - Stat 19 includes 224 `item_tohit_perlevel` (op 2 param 1: v·clvl >> 1), 278 by-time, and Grim Ward (flat 120 + 10/lvl, party).
2. **+AR vs demon / undead** (123/124, plus 245/246 per level and 298/299 by time). Added flat **before** the %, only vs monsters with the MonStats demon or l/hUndead bit (VERIFIED).
3. **Percent** = mastery + stat119 + skillToHit + Σ stat179 (VERIFIED). Applied once as trunc(AR·pct/100.0).
   - stat119 includes 225 per level (v·clvl >> 1) and 279 by time.
   - Auras: Blessed Aim 60 + 15/lvl, Fanaticism 40 + 5/lvl, Concentration 50 + 10/lvl (party).
   - Paladin's own: Holy Fire/Freeze/Shock/Sanctuary 50 + 5/lvl (the `toht` passivestat).
   - Passives: Penetrate 35 + 10/lvl, Warmth 20 + 10/lvl.
   - Enchant 20 + 5/lvl and Cold Enchant 20 + 10/lvl go to the target.
   - Werewolf/Werebear: Shape Shifting 50 + 12/lvl.
   - Taunt, Dim Vision and Inner Sight lower the **monster's** AR only.
   - `hit.js` `SOURCES` has every formula.
4. **Skill ToHit** = D2Common #10653: ToHit + (lvl−1)·LevToHit, or ToHitCalc (Valkyrie, Stun, UberTalicBash).
   - Normal attack: 0.
   - Tiger Strike family: + stat 325 `progressive_tohit`.
5. **Mastery** (melee only): stat 342, or 345 when the weapon is Throwable and the skill's itypea1 is throwable.
   - Layers match the weapon item type (PD 0x102D30B0).
   - Two-Hand Mastery (`2han`) counts only with the other hand empty.
   - A 2-handed sword wielded one-handed (other hand occupied, #10747) counts as `1han`.
   - Values: Sword/Mace 28 + 10/lvl, One-Hand 35 + 12, Two-Hand 30 + 10, Spear 30 + 10, Claw 30 + 10, Throwing 30 + 10.
6. **Missiles**: melee mastery is not used. The to-hit argument is layered stat 345 + #10653 of the missile's skill and level (PD 0x10268220).
   - The attacker for the roll is the missile's owner, so the owner's stats, level and seed are used.

### 7.3 Target defense
1. **Spawn armor class**: #11089 at the spawn level (VERIFIED scaler). It is not recomputed when unique (+3) or champion (+2) levels are added.
2. **Stone Skin**: stat31 × 2, for the unique/superunique only (umod 28, handler 0x6FC41BE0).
3. **−Defense per hit** (stat 120): see §7.5. It changes base stat 31.
4. **Flat armorclass on the monster**: Attract −(120 + 10/lvl + 5·Curse Mastery blvl) and Inner Sight −(40 + tiered 25/45/60/80/100).
5. **Percent** = stat16 + stat171, summed with no floor (#10672). Sources:
   - Fanatic champion: stat16 = −70.
   - Map mod `map_mon_ac%`: → stat16 (+X).
   - Monster Shout: +100 + 10/lvl.
   - Overseer Whip: +50.
   - Your Conviction: −dm(lvl, 10, 75).
   - Weaken: −(10 + 2/lvl).
   - Battle Cry: −min(15 + lvl − 1, 30).
   - Cloak of Shadows: −min(15 + lvl − 1, 95).
   - Amplify Damage, Decrepify and Lower Resist do **not** change defense in PD2.
6. **stat 182** armor override: BaalMinionBossBerserk −100 sets defense to 0 while active. The player's Berserk (−25) is irrelevant here.
7. + stat 33 (melee) or stat 32 (missile). Monsters normally have none: no MonProp or MonUMod gives it.
8. **ITD** (§7.1), then **−% target defense** (stat 116):
   - Items plus Blessed Aim's passive (passivestate `passive_blessedaim`, −1% per hard point).
   - Halved vs players, MonStats bosses, superuniques and mercs; **plain uniques and champions are not halved**.
   - v < 1 → 0; cap 100.
9. **Negative defense**: if the result is < 0, the chance function adds |Def| to AR (VERIFIED).
10. **Map mods** (READ, new): PD2 routes map-item stats through the ItemStatCost `Divide` column (record +0xC) (PD 0x102D4E70).
    - < 1000: monster stat id, in the list at game+0x1DF8.
    - 1000 + x: player stat.
    - 2000 + x: global key, in the game table at +0x2630.
    - 3000: phys-as-extra.
    - The monster list is applied at spawn as a state-198 (`map`) statlist (PD 0x102DBFE0 → 0x102DBA50). This happens only in Levels Id 137–201 and only for MonStats Align = 0 monsters.
    - Relevant here: `map_mon_ac%` → 16, `map_mon_tohit` / `map_mon_att` → 119 (monster AR only), `map_glob_arealevel` → key 1 (monster level), `map_mon_curse_resistance` → 109 (curse **duration** only; monster cap 75% in PD 0x102BFE20).
    - Stat 395's own op 13 never fires on monsters: D2Common's op-13 update check (0x6FD89E5F) requires an item owner.
11. **Aura Enchanted** uniques (umod 30, table 0x6FD2E4A0, not patched) get Might, Holy Fire, Blessed Aim, Holy Freeze, Conviction, Fanaticism, or Holy Shock (monster level ≥ 20). The level is mlvl/6, /5, /7 or /8, clamped 1..99. Class 704 gets Conviction 20. None of these raise the monster's defense.

### 7.4 Levels
- aLvl = attacker stat 12 (clvl; for missiles, the owner's). dLvl = monster stat 12.
- Monster level (spawn, 0x6FCCFDB0):
  - Normal difficulty, noRatio monsters and MonStats bosses: the MonStats Level column.
  - NM/Hell otherwise: Levels MonLvlEx.
  - **PD2 replaces that lookup** (call 0x6FCCFF0B → PD 0x102EEE40 → 0x10268D00). In map areas (Id 137–201) it adds the map's `map_glob_arealevel`. Otherwise, if the area is in the active **desecrated** group, the level is 85. That group is only chosen in Hell and rotates on a time basis (PD 0x1026C3C0).
  - Then +3 for uniques, superuniques and their minions (umod 4), or +2 for champions (+3 then −1).
- No level condition exists anywhere else in the roll: not in the resolver, ITD, −%def, mastery, or the post-hit code.

### 7.5 After the hit: stat 120 `item_damagetargetac` (READ, not patched by PD2)
D2Game 0x6FCFA890 is called from the damage function 0x6FCFD450. That function is reached from the melee damage store 0x6FCFDDE0 and from other damage paths.

It runs only if:
- the attacker is a player, or has alignment 2 through the state-105 statlist (#10830); mercenaries and summons do not qualify
- the defender is a monster
- the damage record has the hit flag set and none of 0x8380

What it does: `base31 = max(0, total31 + stat120)`. It reads through #10973 (full stats) and writes through #10887 (base).
- Flat stat-31 changes active at that moment are baked into the base again on every hit (Attract, Inner Sight: full' = 2·full − base + v).
- PD2 patch records inside 0x6FCFD450 only replace the crit/deadly-strike block (0x6FCFD52C–0x6FCFD5B4) and the call at 0x6FCFD746.

### 7.6 Special attacks (unchanged, READ)
- Every melee hit rolls on its own (Frenzy, Zeal, Fury, Whirlwind, Double Swing, kicks). There is no auto-hit for running or stunned targets.
- Smite, splash, and the 949 missiles without Missiles.txt `ToHit` never roll. The latter include Guided Arrow, Lightning Strike/Fury bolts, throwing stars and all spells.
- Blade Shield rolls but cannot be blocked.
- Leap Attack and Blade Creeper count stat 119 twice.
- The PD2 hover tooltip (PD 0x10271940 / 0x10271540) is only an estimate:
  - It uses the MonStats Level column, without area, map or champion levels.
  - It adds +AR vs demon/undead after the %.
  - It does not exclude mercs from ITD.
  - In NM/Hell with a setting off, it scales defense by 10/12.

### 7.7 Calculator inputs added to `hit.js` (backward compatible)
- `playerAttack`:
  - `alwaysHit` (non-rolling attacks)
  - `blessedAimBlvl` (added to stat 116 before halving)
  - `mon.ac` / `mon.flatAC` / `mon.defPct` / `mon.overridePct`, as an alternative to `mon.def`
  - a note when defense is negative
- `monsterStats` options:
  - `mapLevelBonus`, `desecrated`, `levelOverride`
  - `mapDefPct` (map_mon_ac%)
  - `defPct` (curses, auras, Shout: stat 171)
  - `flatAC` (Attract / Inner Sight)
  - `overridePct` (stat 182)
  - `mapARPct` (monster side)
- `SOURCES`: skill-level formulas for every AR% / defense source above. `dm()`, `itdChance()`.
- `test_hit.js` still replays the native vectors (0 mismatches) and checks the new factors.

**Not modelled / open**:
- Whether Blessed Aim's passive −%def needs the aura to be active.
- The exact desecrated area groups (a runtime table).
- Realm servers may run different server code; this covers the client install, where single-player maps run.
