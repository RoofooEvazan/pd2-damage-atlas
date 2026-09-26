# Player → monster damage (server side, PD2 = 1.13c + ProjectDiablo.dll)

JS: `adv/re/damage.js`. Native test: `harness/damage.c` + `harness/damage_check.js`
(`gcc -m32 -nostdlib -static -O1 -fno-pie -no-pie -ffreestanding -fno-stack-protector -o damage damage.c && node damage_check.js 20000`).

**Status key**
- **VERIFIED**: the real code (stock D2Game, or PD2's own function in `PD.img`) was run natively and matched `damage.js` on every case. The last run was 9 modes × 20,000 random cases with **0 mismatches**.
- **READ**: read from the disassembly but not executed.

**Units:** the server keeps life and damage in **1/256 of a point** (`<<8`), and `trunc` is C division. Stat numbers are ItemStatCost IDs (`s25` = stat 25).

## 0. Pipeline and where PD2 hooks it

| step | stock address | PD2 change |
|---|---|---|
| Melee/skill hit: fill damage struct | D2Game `0x6FCFD450` (from `0x6FCFDDE0`) | crit block `0x6FCFD52C..0x6FCFD5B4` → `0x102ED930` → `0x1026F8C0` → `0x10270E00` |
| Physical roll | `0x6FCFC530` → roll `0x6FCFBED0` | mastery call `0x6FCFC6A0` → `0x102727D0` (charscreen.md) |
| Elemental roll (item stats) | `0x6FCFCD80` → `0x6FCFBED0` | – |
| Resist/DR/absorb loop | `0x6FCFC0B0`, type table `0x6FD22AB0` (12 × 0x2C) | per-type call `0x6FCFC4E9` → `0x102ED620` → **`0x1026F410`**; table entry 0 pierce := 425, entry 4 pierce := 358 (`0x102BE9C0`); PvP% `0x6FCFAC10` → `0x1026D1D0` (returns 100 for player→monster, READ); cold/poison length pre-step `0x6FCFA7B0` → `0x102713C0` (not read) |
| Execute: events, clamp, leech, life | `0x6FCFE13C` | leech call `0x6FCFE218` → `0x102EE040` → **`0x102700F0`**; heal SetStat in `0x6FCFB570` → **`0x10268600`** |
| On-hit events (itemevent funcs table `0x6FD277A8`) | event 5 `domeleedamage` / 6 `domissiledamage` fired at `0x6FCFE1DD` / `0x6FCFE1C6` | table overwritten by `0x102BEA10`: func 15 open wounds → **`0x102AF060`**, func 16 crushing blow → **`0x102AF610`** |
| Missile hit | `0x6FC5AE10`; damage from missile stats `0x6FC5A4E0` | DS block `0x6FC5A730` → `0x102EEBD0` → **`0x10270C50`** (owner's crit/DS) |

Order within one hit (`0x6FCFE13C`, READ):
1. Fill: physical roll, then crit/DS, then elemental, then leech %.
2. Resist/DR/absorb for each damage type.
3. On-hit events. This is where crushing blow and open wounds act.
4. Physical damage is clamped to the monster's *current* life (`0x6FCFE1FB`).
5. Leech.
6. Total damage is subtracted from life.
7. Cold, freeze, poison and burn are applied.

## 1. Physical damage

### 1a. Roll: D2Game 0x6FCFC530 + 0x6FCFBED0 (VERIFIED, 20k cases, real RNG)
```js
[mn,mx] = weapon ? (grip==2 ? [s23,s24] : [s21,s22]) : [s21>0?s21:1, s22>1?s22:2]
mn = (mn<<8) + (s111<<8);  mx = (mx<<8) + (s111<<8)      // item_normaldamage: attacker TOTAL, BEFORE the %
if (mn < 1) mn = 256;  if (mx <= mn) mx = mn + 256
pct = enDmgPct + s25 + (weapon ? trunc(str*StrBonus/100) + trunc(dex*DexBonus/100) + mastery : str)
pct = max(pct, -90)
min' = mn + pct100(mn, s18+pct);  max' = mx + pct100(mx, s17+pct)
phys = base + min' + rand(max'-min')          // rand only if max' > min'
phys = src==128 ? phys : trunc(phys*src/128)   // Skills.txt SrcDam
```
- **Roll range:** `rand(n)` is in [0, n), so the top value is max' − 1/256. A weapon never quite reaches its listed maximum.
- `pct100(v,p)` = trunc(v·p/100), except when v > 0x100000 it is trunc(v/100)·p. `damage.js` has the exact version.
- **enDmgPct** (damage+0x0C) holds:
  - the skill's ED% (D2Common #10786 at the skill do-func, e.g. `0x6FC64F9D`);
  - +s121 against demons (#10255, inline), +s122 against undead (`0x6FCFA8C0`, #10239), and +s180 `damage_vs_montype` (`0x6FCFAD70`).
  - `0x6FCFA890` applies stat 120 `item_damagetargetac` to the target's defense; it is not a damage term.
  - These are added only when the attacker is a player or its alignment (#10830) is 2, and the target is a monster.
- **base** (damage+0x08) is the skill's own flat physical damage. It is added after the percent, so ED does not multiply it.
- **mastery**: PD `0x102727D0` mode 1. That is stat 343 `passive_mastery_melee_dmg`, or 346 when throwing, filtered by weapon type (see charscreen.md). It only counts with a weapon.
- **ED on the weapon** is already inside s21–s24 (the op-13 rule in charscreen.md). Off-weapon ED sits in s17 (max) and s18 (min).
- **Difference from the character screen:** the server adds **s111 from all sources before the percent**. The screen adds weapon-only s111 after the percent.

### 1b. Critical strike / deadly strike: PD 0x10270D20 + 0x10270E00 (VERIFIED, 20k cases, real rand)
Stock 1.13c rolled mastery crit, stat 337 and stat 141 as three separate chances, and any success doubled the damage. **PD2 replaces this:**
```js
critChance = masteryCrit(PD 0x102727D0 mode 2: stat 344, or 347 when throwing) + s337 + s258   // item_crit_chance
if (critChance > 0 && rand(100) < min(critChance, 75))  → CRIT, mult = 200 + s256 + typeParam(256)
else if (s141 > 0 && rand(100) < min(s141, 75 + s210))   → DS,   mult = 150 + s257 + typeParam(257)
phys = trunc(phys * mult / 100)   // double precision; sets result flag 0x2000
```
- **Only one of the two can happen.**
  - Crit is rolled first.
  - DS gets a second, independent roll only if crit fails. When crit chance is 0 or less, DS uses the first roll.
  - The chances therefore combine as P(any) = c + (1−c)·d.
- **Multipliers:**
  - Crit is ×2.00 plus stat 256 `item_crit_multiplier`.
  - DS is ×1.50 plus stat 257 `item_ds_multiplier`.
  - `typeParam(x)` = PD `0x102D30B0`: the sum of the stat's entries whose parameter is an item type the weapon matches (same mechanism as the mastery stats).
- **Caps:**
  - Crit chance is capped at 75.
  - DS is capped at 75 + stat 210 `item_maxdeadlystrike`.
  - Stat 250 (DS per level) reaches 141 through its op.
- **Physical only:** crit/DS multiplies physical damage only, before resistances. Leech therefore sees the multiplied value.
- **Melee needs a weapon** (READ). PD only reaches the hook when `arg5==0` and D2Game `0x6FC572C0` returns a weapon. Every other path jumps into the NOPed block, so an unarmed melee hit gets no crit and no DS.
  - Missiles: `0x10270C50` applies the same rules with the owner's stats and the missile's skill.
  - An ownerless missile gets a stock-like DS of ×1.5 from the missile's own stat 141.
- **Expected multiplier:** `critMultiplier()` = 1 + c·(1 + s256/100) + (1−c)·d·(0.5 + s257/100).

### 1c. Physical resistance, DR and pierce
These are covered in §2: the physical entry of the type table goes through the same code as the elements. What is specific to physical damage:
- **Resistance:** monster stat 36 `damageresist`.
- **Pierce:** PD2 gives the physical entry stat **425 `passive_phys_pierce`**; stock had none.
- **Flat reduction:** stat 34 `normal_damage_reduction` (<<8) is subtracted before the resist.
  - It is scaled by damage+0x54 / 1024 when that field is set (missiles copy stat 327 there).
  - PD caps it at 25 only when the target is in a PvP map (levels 157, 159, 166).
- **No absorb** for physical damage.
- **Sanctuary:** attacker state 47 (`sanctuary`) sets physical resist to 0 against undead that are not prime evil (PD getter `0x1026F7E7`, READ).

## 2. Resistances (all damage types): PD 0x1026F410 / 0x1026F680 / 0x1026EA70 / 0x1026F820 (VERIFIED, 20k cases)
Type table `0x6FD22AB0`, with entries as PD2 configures them:

| # | field | res | maxres | pierce | absorb % / flat | DR stat |
|---|---|---|---|---|---|---|
| 0 | phys +0x08 | 36 | – | **425** | – | 34 |
| 1 | fire +0x10 | 39 | 40 | 333 | 142/143 | 35 |
| 2 | ltng +0x1C | 41 | 42 | 334 | 144/145 | 35 |
| 3 | cold +0x24 | 43 | 44 | 335 | 148/149 | 35 |
| 4 | magic +0x20 | 37 | 38 | **358** | 146/147 | 35 |
| 5–6 | cold/freeze length | 43 | 44 | 335 | – | – |
| 7 | poison length +0x2C | 110 | – | 336 | – | – |
| 8 | poison +0x28 | 45 | 46 | 336 | – | – |

```js
res = monster stat[resStat]                       // total: includes Conviction, Lower Resist, Amplify...
if (res < 100 || !monster) {                      // pierce is IGNORED vs immune (res>=100) monsters
    res -= attacker stat[pierceStat]
    if (res < 0 && ownerIsPlayer(attacker)) res = trunc(res/2)     // PD 0x1026EA70: negative res counts HALF
}
if (res < 1) res = max(res, -100)                 // floor -100, AFTER halving (i.e. needs raw -200)
// monsters: no upper cap; players/mercs capped (not covered here)
v = max(v - DR, 0)                                // DR only on phys (34) and elemental (35); skipped with bypass
if (v > 0 && res != 0) v = trunc(v * (100 - min(res,100)) / 100)   // double precision; res>=100 → 0
v -= trunc(v * min(absorb%,40) / 100);  v -= min(v, absorbFlat<<8)  // monster absorb, if any
```
- **When the halving applies:** it runs for every table type (all of them have a pierce ID in PD2), even with pierce 0.
  - It applies when the attacker, or the attacker's owner found by `0x102CB180`, is a player. That covers players, their missiles, mercs and summons.
  - So −1 resist below zero is worth only ½ point. Because of `trunc`, it moves in steps of 2.
- **Bypass flags:** skill_bypass_undead/demons/beasts (103/104/106) matching the target means positive resist, DR and absorb are all ignored. Negative resist still applies.
- **Stock `0x6FCFB3C0`** (replaced by PD2) has the same immunity rule and the same −100 floor, but no halving and no physical or magic pierce.

### −resist from auras and curses on immune monsters (VERIFIED, 20k cases)
Conviction, Lower Resist, Amplify Damage, Decrepify and similar effects lower the monster's stat directly, so they count before the ≥100 test.
- **PD `0x102C0540`:** each negative contribution to stats 36, 37, 39, 41, 43 or 45 is **halved** (trunc) when the monster's *base* value of that stat (#10587) is **> 99**.
  - Players and mercs are exempt.
  - It is wired at the aura path `0x6FCBA2A0` (Conviction and other auras), curse do-func 30 `0x6FC702BF` (Amplify, Lower Resist, Decrepify…) and `0x6FC703A0`.
- **Stock `0x6FC6E230`** divides by 5 instead.
  - The Confuse path `0x6FC70BDA/0x6FC70C89` (do-func 61) is not redirected and still uses /5.
- **Example:** fire-immune base 110 with Conviction −85 → −42 → 68. That is below 100, so pierce now applies normally, and Lower Resist would stack the same way.

## 3. Elemental damage and masteries
- **Item/charm/aura elemental in the melee fill** (`0x6FCFCD80`, VERIFIED, 20k cases):
  `v = skillElem + min' + rand(max'-min')`, where `min' = (min<<8)·(1+m/100)` and `max'` likewise.
  - Stat pairs and masteries: fire 48/49 with mastery 329, ltng 50/51 with 330, cold 54/55 with 331, **magic 52/53 with 357** (357 is already in stock here).
  - The whole value is then multiplied by src/128.
  - The mastery does **not** multiply the skill's own elemental damage that is already in the struct.
- **Skill elemental** (D2Common #11091 min `0x6FDA0360`, #10121 max `0x6FDA0460`, READ):
  1. `e = base<<HitShift`
  2. `e += trunc(e·synergy/100)` (EDmgSymPerCalc)
  3. `e += trunc(e·mastery/100)`
  - Synergy and mastery **multiply**; they are not added together.
  - PD hook `0x10268A50` adds magic mastery 357 for EType 3.
  - Missile damage uses D2Common #10413 `0x6FDBBAA0` / `0x6FDBAB20`, where PD's `0x10267270` maps 48–55 and 57–58 to 329, 330, 357, 331 and 332.
- **After that, resist:** each element goes through §2. Magic now has pierce 358 and the halving.

## 4. Crushing blow: PD event 16 = 0x102AF610 (VERIFIED, 20k cases; damage formula and divisors)
```js
chance = s136 + typeParam(136) (+ Smite calc Skills+0x140); fires if rand(100) < chance   // one roll per hit: see crit_cb.md (event nodes per layer, crushingBlowChance)
div = defender player or merc (#11104) ? 10
    : monster: primeEvil (MonStats `primeevil`, #10278) ? (mapBossList(0x102C7C00) ? 30 : 70 + 10*(100 - hp%))
               : 8                                  // normal, champion, unique, superunique, Andariel, Duriel alike
      + (monster ? playerCountHp%(stat100)/50 : 0)  // PD table 0,0,70,140,...,490 → +1.4 per extra player
div *= (event == domissiledamage) ? 1.5 : 1
if (stat36 >= 100) no effect                       // phys immune; pierce 425 NOT used for CB
cb   = life * (1 + s268/100 + typeParam(268)/100 [+ Smite calc2/100]) / div   // CURRENT life, double
life = trunc(life - (cb - trunc(trunc(cb)*res36/100)));  if (life < 1) → 0, kill flag
```
- **Timing:** CB happens on the damage event, before this hit's own damage is subtracted. The hit's physical damage is then clamped to the reduced life.
- **Effect size:**
  - Normal monsters lose 1/8 of current life (12.5%) per proc.
  - Prime-evil bosses (Mephisto, Diablo, Baal, ubers, many PD2 map bosses) lose 1/70 at full life, dropping to 1/1070 at 0% (hp% = #11027).
  - Monsters on PD's map-boss list lose 1/30.
  - Stat 268 `item_crushingblow_efficiency` scales the fraction linearly (+100 → ×2).
- **Player count:** the divisor adds (monster HP bonus %)/50. PD patches the D2Game table `0x6FD1B614` to 0/0/70/140/210/280/350/420/490; above 8 players it is (n−2)·50.
- **READ only:** `playerCountHpBonus` (the PD table values come from the patch records), the Smite branch, and the PvP branch.

## 5. Open wounds: PD event 15 = 0x102AF060 (VERIFIED, 20k cases; damage value)
```js
chance: rand(100) < s135
lvl  = clvl < 2 ? 1 : clvl
dmg  = levelScale([9,18,27,36,45], lvl) + 25 + 5*s501           // D2Game 0x6FCCC940, deep_wounds x5
r    = res36 < 100 ? res36 - s425 : 100;  if (r < 0) r = trunc(r/2)
dmg  = trunc(dmg * (100 - r) / 100);  if (target owned by a player) dmg = trunc(dmg/4)
state 62 'openwounds', 125 frames, stat 74 hpregen = -dmg   → dmg/256 life per frame, dmg·25/256 per second
```
`levelScale` gives a slope of 9 per level up to level 15, then 18, 27, 36 and 45 per level for 16–30, 31–45, 46–60 and 61+. Example: clvl 90 gives `126+270+405+540+1350 = 2691`, so dmg = 2716.

**Stacking (VERIFIED, see `pierce_ow.md` §2):**
- A target has one OW state, owned by whoever created it.
- The owner's procs add their dmg while stat 189 `openwounds_stack` is below 3. At 3, a proc only resets the 125-frame duration; the drain is unchanged.
- Another attacker's proc adds nothing, resets the duration, and sets the stack count back to 1.
- `damage.js openWoundsTarget()` models this.

## 6. Leech: PD 0x102700F0 → stock 0x6FCFBA40 (VERIFIED, 20k cases), heal cap PD 0x10268600 (VERIFIED, 20k cases)
```js
L = s60, M = s62 (percent, from the attacker; missiles: from the missile's copied stats)
PvP map target → 0;  missile → L >>= 1, M >>= 1                 // PD: ranged leech halved
drain = MonStats Drain / Drain(N) / Drain(H) for the difficulty (blank = 0 → no leech); non-monster 100
L64 = trunc((L<<6)/LifeStealDivisor), M64 = trunc((M<<6)/ManaStealDivisor)   // PD2 data: 1 / 2 / 3
gain = trunc( mulDiv(drain, mulDiv(L64, phys, 100), 100) / 64 )   // <<8 units; drain step skipped at 100
```
- **phys** is the physical damage after crit and resist/DR/absorb, clamped to the monster's remaining life. Overkill and **elemental damage never leech**.
- **Drain data:**
  - Regular Hell monsters use Drain 33–100%.
  - Diablo 100/50/20. Uber Diablo and the Diablo clone 15. Mephisto NM/Hell blank, which means no leech.
- **Stat 488 `lifedrain_percentcap`:** PD wraps the leech heal SetStat. The heal is refused when both the old and the new life are ≥ (100 − cap)% of max life. Otherwise the full new value is set; it is not clamped to the threshold.
- **Order:** mana is added first, then life.

## 7. +1% skill damage vs −1% enemy resistance
```
final = base · (1+syn/100) · (1+mastery/100) · (100 − r_eff)/100
r_eff = res ≥ 100 ? res (immune, pierce ignored) : max(half_if_negative(res − pierce), −100)
```
- **+1 skill damage %** always gives ×(1 + 1/(100+syn)). It is a smooth percentage of whatever reaches the target.
- **−1 enemy resist** gives:
  - **+1 point of r** while `res − pierce ≥ 0`, which is a relative gain of 1/(100 − r). This is worth more than +1% skill damage whenever 100 − r < 100 + syn, i.e. almost always.
  - **+½ point** once below 0 (PD halving), in steps of 2 because of `trunc`: −1 → 0, −2 → −1, −3 → −1, …
  - **0** at the floor. After halving that is r_eff = −100, which needs raw res − pierce ≤ −200 (×2 damage).
  - **0 against immunes (res ≥ 100)** until auras/curses (halved on base ≥ 100) bring the total below 100. At that point all pierce applies at once.
- **Physical works the same way** through stat 425 and Amplify/Decrepify (stat 36). Crushing blow ignores stat 425 and does nothing against physical-immune (≥100) targets. Open wounds uses 425 with the same halving.
- `damage.js`: `resistSlope(res, pierce)` (effective points per point of −res) and `damageFactor()`.

## 8. Caveats
- **Out of scope:**
  - Monster resist values themselves: MonStats per difficulty, MonUMod, PD map modifiers.
  - The PvP rules.
  - The stock melee `arg5` cases.
  - The poison and burn length math.
  - `0x102713C0` (the PD replacement of the cold/poison-length pre-step, not read).
- **Missile physical damage:**
  - It is built when the missile is created (D2Common #10413 `0x6FDBBAA0` → #10484 `0x6FDBB4A0` copies it into the missile's stats 21/22/25…).
  - It is rolled on hit in `0x6FC5A4E0`.
  - This build-up was only partly READ. The weapon/grip/Str-Dex/mastery terms match the melee formula, but the percent rounding order differs and is not modelled.
- **Stubs in the native test:**
  - Stat getters (fake stat arrays).
  - The weapon/grip/StrBonus/DexBonus/mastery lookups.
  - PD's item-type-param sum `0x102D30B0`.
  - The prime-evil, merc, map-boss and hp% queries.
  - The PD custom RNG for CB/OW chance (a fixed value).
  - State creation/lookup (captured).
- **Real code in the native test:**
  - RNG `0x6FC211D0/0x6FC21480`.
  - MulDiv.
  - The owner lookup `0x102CB180` and PvP-map test `0x102CEAE0`.
  - The level table and the player-count table.
  - Absorb.
  - Everything listed as VERIFIED.
- **Server:** this is the client-install code. PD2 realm servers could differ.
