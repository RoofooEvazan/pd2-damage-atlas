# Critical strike, deadly strike and crushing blow (PD2 = 1.13c + ProjectDiablo.dll)

Everything below comes from the game code (D2Game/D2Common 1.13c, ProjectDiablo.dll) and PD2 `data.zip`.
It extends `damage.md` §1b and §4. JS: `adv/re/damage.js` (`critOutcome`, `critMultiplier`, `crushingBlowChance`,
`crushingBlow`, `crushingBlowDivisor`). Native check: `harness/damage.c` + `harness/damage_check.js`.

**Status key**
- **VERIFIED**: the real code ran natively in the harness and matched the JS on every case. The last run was 9 modes × 20,000 cases, 0 mismatches.
- **READ**: read from the disassembly but not executed.
- **DATA**: read from the PD2 .txt files.
- **MODEL**: our own simulation built from READ pieces.

## 0. Stat IDs

| id | name | who sets it | how the server reads it |
|---|---|---|---|
| 337 | `passive_critical_strike` | Amazon Critical Strike (`dm12`, 5→75; lvl 20 = 63, lvl 30 = 68). No passiveitype, so it is stored at layer 0 | #10973 GetUnitStat(u, 337, 0) |
| 344 | `passive_mastery_melee_crit` | Sword, Mace, One-Hand and Spear Mastery, Claw Mastery (`dm56`; lvl 20 = 29, Claw = 25). Stored at layer = passiveitype | PD 0x102D30B0 (weapon-matched layers only) |
| 347 | `passive_mastery_throw_crit` | Throwing Mastery (`thro`) | same, and only when the weapon is throwable **and** the skill is a throw skill (PD 0x102727D0) |
| 258 | `item_crit_chance` | items | GetUnitStat(u, 258, 0) |
| 256 / 257 | `item_crit_multiplier` / `item_ds_multiplier` | items; Javelin and Spear Mastery puts 256 on layer `spea` | layer 0 + 0x102D30B0 |
| 141 / 210 | deadly strike / max deadly strike | items, Blessed Aim, 250 (DS per level, via op) | layer 0 |
| 136 | `item_crushingblow` | items; Two-Hand Mastery `dm56` on layer `2han` (lvl 20 = 29) | layer 0 + 0x102D30B0 (+ Smite calc3) |
| 268 | `item_crushingblow_efficiency` | items; Two-Hand Mastery `min(edln,25)` on layer `2han` (10 + (lvl−1), reaches 25 at lvl 16) | layer 0 + 0x102D30B0 (+ Smite calc4/100) |

**How passive layers work** (READ): D2Common #10056 at 0x6FDA2480 builds a skill's passivestat list and calls SetStat(list, stat, value, **layer = passiveitype**) at 0x6FDA25DF..0x6FDA25FB.
- GetUnitStat(u, stat, 0) (#10973 0x6FD88B70 looks up `(stat<<16)|layer` exactly) therefore never sees a mastery's stats.
- Mastery stats reach the attack code only through **PD 0x102D30B0** (`typeParam`). It sums the stat's entries whose layer is an item type the wielded weapon (#10061) matches. The item-type test uses Weapons.txt `type` **and** `type2`.
- Special rules in 0x102D30B0 for layer 114 `2han` and layer 116 `1han`:
  - **`2han`** counts only when the other hand is **empty**. The `2han` types are all two-handed melee weapons through `type2`: 2H swords, 2H axes, mauls, polearms, spears, staves, scythes and the 2H Phase Blade.
  - **`1han`** on a 1han weapon always counts, with or without a shield or second weapon.
  - **`1han`** on a 2han-type weapon counts only when the other hand is occupied. For a Barbarian or Assassin (#10747) this is the branch that lets **One-Hand Mastery apply to a two-handed sword held one-handed** (with a shield or a second weapon).

## 1. Critical strike and deadly strike (Q1)

### Pipeline (VERIFIED, damage.md §1b; stat reads confirmed READ here)
Melee fill D2Game 0x6FCFD450 → crit block 0x6FCFD52C (PD patch) → 0x102ED930 → 0x1026F8C0 → **PD 0x10270E00** → roll **PD 0x10270D20**. Missiles: 0x6FC5A730 → 0x102EEBD0 → **0x10270C50**, which runs 0x10270E00 on the missile's owner with the missile's skill.
```
c  = typeParam(344, or 347 when throwing)   // PD 0x102727D0 mode 2; 0 without a weapon
   + GetStat(337,0) + GetStat(258,0)          // Critical Strike + item crit: plain SUM
if c > 0 and rand(100) < min(c, 75):  mult = 200 + GetStat(256,0) + typeParam(256)       // crit
elif GetStat(141,0) > 0 and rand(100) < min(s141, 75 + s210):  mult = 150 + s257 + typeParam(257)   // DS
phys = trunc(phys * mult / 100)               // physical damage only, before resist
```
- **Combining the sources:** they are added into one number, capped at 75, and rolled once. There is no 1−(1−a)(1−b) combination and no scaling ("normalization") other than the 75 cap.
  - Example: Critical Strike 25 (66) + 10 item crit = 76, which is capped to 75.
  - Example: Claw Mastery 20 (25) + 10 item crit = 35.
- **Crit and DS never stack.** DS gets its own second roll only when the crit roll fails. So P(crit) = min(c,75)/100 and P(DS) = (1 − P(crit)) · min(d, 75 + s210)/100.
  - Expected multiplier = 1 + Pc·(1 + s256/100) + Pd·(0.5 + s257/100).
  - Example: 29% crit, 0 DS gives ×1.29. 29% crit with 30% DS gives ×1.3965. 73% crit with 20% DS gives ×1.757.
- **Where it applies:** every hit that goes through the melee damage fill with a weapon. That includes Attack and every melee skill, and each Whirlwind hit (WW calls 0x6FCFDDE0 per hit), which rolls on its own.
  - An unarmed melee hit gets no crit and no DS (READ, damage.md).
  - Missiles use the owner's stats. Masteries count through the owner's weapon type; 347 counts only for throw skills with a throwable weapon.
  - An ownerless missile with stat 141 gets ×1.5 with **no roll** (0x10270C8D, READ).
- **Character screen:** the stock screen shows no crit.
- **BH Advanced Stats panel** (0x1007C040, READ): crit = mastery(252 Claw) + mastery(135 Throwing) + mastery(128 One-Hand) + GetStat(337) + GetStat(258), capped at 75.
  - It **omits Sword/Mace/Spear Mastery crit** (the server counts them).
  - It counts Throwing Mastery for melee.
  - Its mastery check is BH's own, not 0x102D30B0.
  - Our `panel()` copies BH on purpose. The combat model now uses the server formula instead (see §4).

## 2. Crushing blow (Q2)

### Event registration: one roll per hit (VERIFIED for the node logic, READ for registration)
- Stat 136 has `fCallback`. D2Game's stat callback **0x6FCF9470** registers one item-event node for each non-zero `(stat 136, layer)` on the attacker: `domeleedamage` and `domissiledamage` → func 16.
  - The node's argument is `(136<<16)|layer`.
  - Duplicates are refused through 0x6FC570E0. PD wraps that check at 0x6FCF94F9 → 0x102C1120, which only special-cases stat 359.
- An item-CB character with Two-Hand Mastery therefore has two nodes, `(136,0)` and `(136,114)`. The dispatcher 0x6FC57030 calls **PD 0x102AF610** once per node.
- PD 0x102AF610 +0x6F5..+0x75C:
```
tp = typeParam(136) + (current skill == Smite(97) ? calc3 : 0)
v  = GetStat(attacker, 136, 0)
if (v < 1 && tp > 0) chance = tp              // no layer-0 CB: whichever node fires, rolls tp
else if (node.layer != 0) return              // layered node when items also give CB: does nothing
else chance = v + tp
if pdRand(att) % 100 >= chance: return
```
- **Result:** exactly one roll per hit with chance = items + weapon-matched mastery (+ Smite). Nothing is doubled.
  - This was checked natively with random node layers (`crushingBlowChance`, harness mode 7, 20,000 cases, 0 mismatches). A mutation that lets layered nodes roll gives 2,332 mismatches.
  - Only with two different layered CB sources and no item CB would each node roll. PD2 data has only one layered CB source (Two-Hand Mastery), so this cannot happen now (READ + DATA).

### Amount (VERIFIED, 20,000 native cases)
```
div = defender player/merc ? 10
    : prime evil ? (on PD map-boss list ? 30 : 70 + 10·(100 − life%))
    : 8                                          // normal, champion, unique, superunique, Andariel, Duriel
    (+ monster player-count HP bonus/50: +1.4 per extra player, 3p → 10.8, 8p → 17.8)
div ×= 1.5 for missiles (domissiledamage)
eff = 1 + (GetStat(268,0) + typeParam(268))/100 (+ Smite calc4/100)
cb  = life_current(<<8) × eff / div                     // double
if stat36 ≥ 100: nothing                               // pierce 425 ignored
life = trunc(life − (cb − trunc(trunc(cb)·stat36/100)))  // raw stat 36: negative values NOT halved, no −100 floor
```
Every item on the Q2 bug checklist was checked, and none holds:
- **Efficiency as a flat value?** No. It is a percent that multiplies the share linearly (+25 → ×1.25). Two-Hand Mastery caps its own part at 25; the stat itself has no cap.
- **Applied twice?** No. It is read once, per proc.
- **Applied after resist?** No. It is applied before resist, and resist then cuts `trunc(cb)`.
- **Applied to the whole life?** No. It scales the share of *current* life.
- **Integer shift or division error?** No. Life is in <<8 units on both sides and the division by 100 is there.
- **Two-hander bonus, or a Whirlwind or PD hook that doubles CB?** No. The only other PD patches in the execute path (0x6FCFE233/288/2AF) do not fire events.
- **Hit-roll gate:** a proc needs the hit flag (damage+0 bit 0x20). WW's hit roll is PD 0x10270EB0 (via 0x6FC48D76 → 0x102EEDA0).
- **Timing:** CB is applied before the hit's own damage, which is then clamped to the remaining life.

**Quirks, not bugs:**
1. **CB reads raw stat 36.**
   - Negative physical resist counts in full for CB, while normal damage counts it half (PD 0x1026EA70).
   - Amplify Damage 20 (−30) makes CB ×1.30 but physical damage only ×1.15.
   - There is no −100 floor for CB.
2. **Efficiency ≤ −100** would divide by zero, or give a negative CB that *adds* life, since there is no clamp to max life. No PD2 item has negative 268 (DATA), so this is theoretical.
3. **Player count** raises the divisor for normal monsters: 1/8 in single player, 1/17.8 at /players 8.

### Two-Hand Mastery (DATA + READ)
| stat | calc | lvl 1 / 10 / 16 / 20 / 30 |
|---|---|---|
| 342 AR% | ln12 30+10/lvl | 30 / 120 / 180 / 220 / 320 |
| 343 dmg% | ln34 45+15/lvl | 45 / 180 / 270 / 330 / 480 |
| 136 CB | dm56 0→35 | 5 / 23 / 28 / 29 / 31 |
| 268 CB efficiency | min(edln,25), ELen 10 +1/lvl | 10 / 19 / 25 / 25 / 25 |
| 478 splash radius | min(20+lvl,40) | – |

All of these are on layer `2han`. They count only with a two-handed melee weapon and the other hand empty.

### Whirlwind hit schedule (READ; the event timing is MODEL)
- **Do-func:** srvdofunc 76 = stock D2Game **0x6FC48BE0**. It runs on the WW sequence's event frames: seqnum 10 has 8 records (D2Common 0x6FDEBE20), with events on records 3 and 7. The sequence rate is s = stat68 (100 − WSM) + EIAS − 30, clamped 15..175.

> **Correction (audit):** the event-timing part of this bullet is superseded by `whirlwind.md` (VERIFIED, `harness/ww.c`): Whirlwind has no `UseAttackRate`, so its sequence rate is fixed at 256 whatever WSM/IAS, and after the first event (frame 3) the do-func runs **every frame** because 0x6FD02310(p = 3) rewinds the sequence. Only the gate (5 / 6 / 5 frames) sets the hit rate.

- **Hit gate:** PD replaces the stock gate at 0x6FC48CE5 with **PD 0x102BED80**.
  - A hit is allowed when `frame ≥ next`, and then `next = frame + Skills.Param3` (WW Param3 = **5 frames**).
  - Dual wield: 2 hits per tick and `next = frame + 6`, or 5 on PvP maps.
  - Stock 0x6FC46E40 used a 4–16-frame delay from the weapon's own speed (#10592). PD2 removed that, so weapon speed no longer sets the delay.
- **Target:** each hit takes one target, via PD 0x102CB2C0 at 0x6FC48D22. It picks the nearest enemy in range that was not the last target; if the last target is the only one, it is hit again. Each hit then goes through 0x6FCFDDE0: crit/DS, then events, so one CB roll per hit.
- **Rate (superseded; see `whirlwind.md`, VERIFIED natively):**
  - WW hits **exactly 5 times per second** with one weapon (8.33 with two weapons, 10 with two weapons on PvP maps), at any IAS or WSM.
    - WW has no `UseAttackRate`, so its sequence rate is fixed at 256.
    - After the first event (frame 3), the do-func runs every frame, because 0x6FD02310(p=3) rewinds the sequence to record 3.
    - The PD2 gate alone sets the rate.
  - The earlier MODEL figures here (4.4 / 5.0 / 3.1 hits/s by sequence rate) were wrong.
  - With a pack, the hits alternate between the **two nearest** enemies. They are not spread round-robin.

### Worked example (Hell map, one player)
Barbarian level 90, Two-Hand Mastery 20, Whirlwind 20, two-handed sword (Colossus Blade), other hand empty. No other CB gear, so crit/DS come only from gear. Monster with 50% physical resist.
- CB chance **29%**. Efficiency **+25%**, so the share is ×1.25.
- **Per proc** on a normal, champion or unique monster: 1.25/8 = 15.625% of current life, halved by the 50% resist to **7.81% of current life**.
  - Map-boss-list prime evil: 1.25/30 → 2.08%.
  - Prime evil at full life: 1/70 → 0.89%.
  - At /players 3: 5.79%.
- **Expected per WW hit:** 0.29 × 7.81% = **2.27% of current life**. The effect compounds: life × (1 − 0.0227)^hits.
- **Per second** at 5 hits/s (the real rate at any IAS; see `whirlwind.md`): **10.8%** of current life.
  - Against a 60,000-life monster that is about 6,500 life in the first second.
  - The amount falls as the monster's life falls, so CB's share of the kill is largest on high-life targets.
- **Crit/DS** (gear only, e.g. 0 crit and 30% DS) multiplies only the weapon roll: ×(1 + 0.30·0.5) = ×1.15.
  - With Sword Mastery 20 as well (29% crit on the 2H sword) and 30% DS: ×1.3965.
- **Why it feels stronger than the sheet:** the sheet shows the weapon roll with the +330% mastery damage. It does not show CB, which is a share of the monster's life. Against map monsters with tens of thousands of life, CB alone removes about 10%/s of the remaining life. This is working as coded, not an efficiency bug.

## 3. PD2 code bugs found
None in the efficiency math or the CB chance. The observations to report are all by design or theoretical:
- CB ignores the "negative resist counts half" rule (quirk 1).
- There is no guard for efficiency ≤ −100 (quirk 2).
- The ownerless-missile DS always applies ×1.5 with no roll.
- The BH in-game panel under-reports crit for Sword/Mace/Spear Mastery users, and shows Throwing Mastery crit for melee.

## 4. Bugs in our page model, fixed
1. **`engine.js`: masteries on `type2` never matched.** The weapon type test used only Weapons.txt `type`.
   - Every `2han` (Two-Hand Mastery) and `1han` (One-Hand Mastery) passive showed "weapon mismatch".
   - The character-screen AR and damage therefore **left out Two-Hand Mastery** (−220% AR, −330% damage at level 20) and One-Hand Mastery.
   - Fix: `itemIsType` (type + type2) and `passiveLayerMatches`, a copy of PD 0x102D30B0 including the hand rules.
2. **`engine.js`: layered passives added at layer 0.** Passives were added to the totals at layer 0 whatever the weapon.
   - As a result T(136), T(268) and T(256) contained Two-Hand Mastery CB and efficiency, and Javelin and Spear Mastery's crit multiplier, even with a shield or the wrong weapon.
   - Once fix 1 was in, the panel would have double-counted them through `masteryStat`.
   - Fix: `C.T0(id)` (the layer-0 value) and `C.TP(id)` (the weapon-matched layered sum). `panel()` now uses T0, and BH's masteryStat test uses type + type2.
3. **`combat.js`: `attackOn` took crit from the BH panel copy.** That meant only 3 masteries, Throwing Mastery counted for melee, and crit for unarmed hits.
   - Fix: chance = TP(344) + T0(337) + T0(258), capped at 75. Multipliers are T0 + TP. No crit or DS without a weapon.
4. **`combat.js`: CB used the wrong chance and efficiency.** It used `P.cb.chance` and raw T(268).
   - Fix: chance = T0(136) + TP(136), efficiency = T0(268) + TP(268). They are now exposed as `cb.eff` and `crit`.
5. **`adv-template.html`:** the DPS note showed the crit multiplier as 200 + T(256). It now shows the attack's own chance and multipliers. The CB card mentions the Two-Hand Mastery hand rule.

**Still not modelled:**
- ~~The Whirlwind rate~~ Now modelled: `combat.js` uses the fixed PD2 gate (5 hits/s one weapon, 8.33 dual wield, IAS irrelevant; `whirlwind.md`).
- The player-count CB divisor: `attackOn` assumes one player.
- Smite's CB terms.
