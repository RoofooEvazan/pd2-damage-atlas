# Part A — How a player's attack damage is built (physical and weapon-based)

Scope: everything from the player's stats to the finished damage record, before the defender pipeline
(resist, crit/DS, CB, OW, absorb, DR, leech are Part B; missiles/spells/DoT/summons/auras are Part C; monster→player
and PvP are Part D). Code: stock 1.13c D2Game/D2Common/D2Client plus ProjectDiablo.dll (PD) runtime hooks; data: PD2
`data.zip`. Flow-chart data: `adv/re/flow/flow_A.json`. JS: `adv/re/damage.js` (`physRange`, `physHit`, new
`kickDamage`), `adv/re/charscreen.js`, `adv/re/skilldmg.js`.

Status key: **VERIFIED** = the real code ran natively in `harness/` and matched the formula on every case;
**READ** = read from disassembly, not executed; **DATA** = read from PD2 `.txt`/`.bin`.

Units: the server keeps damage in 1/256 points (`<<8`); `trunc` = C division toward zero; `sNNN` = the attacker's
stat NNN (ItemStatCost id), totals from `#10973` (`0x6FD88B70`) unless said otherwise.

New native checks in this part:
- `harness/kickpd.c` — PD kick damage `0x1026EAD0`, **30,000 cases, 0 mismatches** (mutations 28,972 / 2,379).
- `harness/edtab.c` — PD `0x102720C0` (which skill calc is the "damage %" for area/splash hits), all ids 0–599.

---

## 1. Architecture of one melee hit (READ, parts VERIFIED)

```
srvstfunc (animation start) ─┐  or srvdofunc (event frame), per skill
                             ▼
  zero record (0x70 bytes)  →  hit roll 0x6FCFE5A0 → PD 0x10270EB0 (hit_pd2.md)
  → flags |= Skills.HitFlags(+0x12E) / ResultFlags(+0x130); HitClass(+0x134) → rec+0x60
  → rec+0x0C  = skill damage %  (a Skills.txt calc, per do-func: §4 table)
  → rec+0x65/0x68 = conversion element / %  (only if Skills.EType > 0: calc4)
  → skill elemental  0x6FCBF210 (EMin..EMax, synergy, elemental mastery)          [some skills]
  → skill physical   0x6FCBF1A0 (MinDam..MaxDam, synergy, <<HitShift)             [some skills]
  → 0x6FCFDDE0 : fill 0x6FCFD450 (unless rec flag 2) → resist loop 0x6FCFC0B0 (B) → queue copy on attacker+0xAC
event frame: 0x6FCFED30 finds the queued copy (attacker,target) → range re-check → apply (B pipeline)
```

- The **fill** `D2Game 0x6FCFD450(game, attacker, target, rec, a5, srcDam)` is where weapon damage is built:
  1. `0x6FC57540(a5)`: dual-wield hand selection (§3); with `a5 ≠ 0` (kicks) both weapons' stat lists are detached.
  2. rec flag 0x20 set. Player (or alignment 2) vs monster: stat 120 (`0x6FCFA890`, defense), `+s121` if demon
     (#10255), `+s122` if undead (`0x6FCFA8C0`), `+s180` vs montype (`0x6FCFAD70`) are **added to rec+0x0C**.
  3. If rec flag 1 is clear: physical roll `0x6FCFC530` (§2) → rec+0x08.
  4. Crit/DS block `0x6FCFD52C` → PD `0x102ED930 → 0x1026F8C0 → 0x10270E00` (B.crit), only when `a5 == 0` and a weapon.
  5. Item elemental `0x6FCFCD80` added **on top of** the skill elemental already in the record, then
     **the whole element × SrcDam/128** (fire/ltng/cold/magic; poison ×src/128; cold length ×src/128) (§7).
  6. Leech % (s60/s62), bypass flags (s103/104/106), poison length (s59, s101 override, ÷ s326 poison_count),
     stun length s66.
  7. On-hit "attacking" event (3) `0x6FC57030`.
  8. **Conversion** (rec+0x65 = element, rec+0x68 = %): `conv = mulDiv(phys, %, 100)`, `phys -= conv` (≥0), conv
     added to the element (type 10 = random 1–5; poison gets conv/8 and length ≥ 50; cold/freeze length ≥ 50).
  9. `0x6FC970E0` (PvP/merc adjustments, Part D), `0x6FC57260` restores the hand lists.
- `srcDam` 0 in the table means 128 (`0x6FCFD47A`), so a blank SrcDam = full weapon damage in melee.
- Stat 111 and all other `sNNN` reads in the fill use the attacker's **total** with only the swinging hand attached.

## 2. Weapon physical damage — D2Game 0x6FCFC530 + roll 0x6FCFBED0 (VERIFIED, damage.md §1a)

```
if weapon:  [mn,mx] = grip(#10051)==2 ? [s23,s24] : [s21,s22]
else:       mn = s21>0 ? s21 : 1 ;  mx = s22>1 ? s22 : 2               // unarmed / claws counted as weapon
mn = (mn<<8) + (s111<<8);  mx = (mx<<8) + (s111<<8)                   // item_normaldamage, TOTAL, before %
if (mn < 1) mn = 256;  if (mx <= mn) mx = mn + 256
pct = rec+0x0C (skill % + s121/s122/s180) + s25
    + (weapon ? trunc(str*StrBonus/100) + trunc(dex*DexBonus/100) + mastery(PD 0x102727D0 mode 1) : str)
pct = max(pct, -90)
min' = mn + pct100(mn, s18 + pct);  max' = mx + pct100(mx, s17 + pct)
phys = min' + rand(max' - min')  (+ rec+0x08 base, i.e. skill physical already in the record)
phys = src==128 ? phys : trunc(phys*src/128)   (64-bit path above 0x100000)
```
- **Base damage stats.** s21/s22 (1-hand), s23/s24 (2-hand) are *player totals*: the weapon's base damage already
  multiplied by its own ED (op-13 rule, charscreen.md), plus every +min/+max from charms/rings/jewels, all
  inside 21–24 and therefore multiplied by the full percent below. Grip 2 (`#10051 0x6FD6FB80`): a `2handed` item
  alone, or a Barbarian holding a `1or2handed` item with the other hand empty. Bows/crossbows only have 23/24.
- **Throw damage (s159/s160)** is never used by melee; it is only read by the missile builder (§8) when the
  weapon type is Throwable (`0x6FD74C30`). Melee with a javelin/throwing weapon uses 21/22.
- **Ethereal**: the +50% base is in the item's 21–24 at creation; PD2 adds stat 484 `item_dmgpercent_pereth`
  (op 4 on stat 183 `equipped_eth`) → +ED% into 17/18. Stat 183 is computed live by PD's stat-getter hook
  `0x102670A0` (all four `0x6FD88A80` calls in #10973/#10910 → PD `0x102EDD40`): number of equipped
  (page 0xFF/1, body slot ≠ 0/11/12) ethereal items (#10225) (READ).
- Stat 487 `item_dmgpercent_permissinghppercent` (op 4 on stat 184 `missing_hp`); the hook computes
  `missing_hp = trunc((1 − life/maxlife)·100)` in double (READ).
- **Weapon ED vs off-weapon ED**: the weapon's own 17/18 are consumed into its 21–24 and never reach the player;
  off-weapon 17/18 (Fortitude, ED jewels in armor, PD2 per-eth/per-missing-hp) apply to the whole base
  (charscreen.md "op-13 rule").
- **s25 `damagepercent`** is the channel for every skill/aura/shapeshift damage bonus: Might (owner: `edmn`
  passive + `edmn/2` aura), Fanaticism (`ln56` + `ln56/2`), Concentration (`ln34` + `ln34/2`), Battle Command
  (`ln34`), Werewolf (`ln56`), Werebear (`ln12`), Maul charges (`lvl*40`, lvl = charge count), items (DATA; aura
  owner/party split: auras.md, Part C).
- **Str/Dex**: Weapons.txt `StrBonus`/`DexBonus` (#10213/#10120) per weapon; unarmed uses **+Str%** (100 per point).
  No PD2 change in this function except the mastery call.
- **Roll**: `rand(n)` is in [0, n) → the listed maximum is never reached (max − 1/256).
- **Minimum** after −90% cap: 10% of base.

## 3. Which weapon swings (dual wield) — D2Game 0x6FC572C0, 0x6FC57540 (READ)

- Non-dual-wield classes: `#10101` primary weapon.
- Barbarian/Assassin (`#10747`): by Skills.txt `weapsel` (+0x168):
  | weapsel | skills | hand |
  |---|---|---|
  | 0 | Attack, most skills | right hand if it is a weapon (type 45), else left |
  | 1 | Left Hand Swing/Throw | left hand |
  | 2 | Whirlwind, Blade Dance | with two weapons: right, or left when the used skill's flag 0x2000 (#10397) is set (who sets it per hit not traced) |
  | 3 | Double Swing, Frenzy, Berserk, Double Throw, Dragon Claw, Tiger Strike, Fists of Fire, Claws of Thunder, Blades of Ice | **parity of the sequence position** `unit+0x38>>8`: even → right, odd → left |
  | 4 | Kick, Smite, Dragon Talon/Tail/Flight | **no weapon** (returns 0) |
- The fill then attaches the swinging weapon's stat list and **detaches the other one** (`#10164 attach/detach`,
  `0x6FC57540`), so only the swinging weapon's damage-related stats (21–25, 17/18, 111, elemental 48–58, leech…)
  count for that hit. Non-damage-related stats (crit 258, DS 141, CB 136) count from both weapons.
- Masteries: PD `0x102D30B0` checks the weapon from `#10061` = the item whose GUID is inventory+0x1C (the
  "primary" weapon), not necessarily the swinging one (READ; unclear in practice).

## 4. Skill damage % and extra damage per do-function (READ unless marked)

Offsets in the Skills record (0x23C): calc1 +0x138, calc2 +0x13C, calc3 +0x140, calc4 +0x144, Param1..8 +0x148..
+0x164, SrcDam +0x1A5, EType +0x1DC, weapsel +0x168 (DATA, skills.bin). PD writes its own entries into the
srvdofunc table (`0x104E3114` → `0x6FD274A8`) and srvstfunc table (`0x104E30E0` → `0x6FD27338`) at init
(PD `0x102BE412` / `0x102BE69F`).

| skill(s) | server function | damage % (rec+0x0C) | extra |
|---|---|---|---|
| Attack (0 and class variants) | do1 `0x6FCC2C40` | 0 | fill directly, SrcDam 128; ranged weapon → missile path (`0x6FCC28F0`) |
| Jab | do7 `0x6FC6CB00` (per sequence event) | calc1 | conversion calc4 if EType; skill elem |
| Charged Strike | st6 `0x6FC6C720` (melee) + do11 (bolts) | **calc1 (= bolt count 3–12)** | skill elem (EMin–EMax ltng) in the melee hit |
| Power Strike, Lightning Strike | st10 `0x6FC6C3A0` + do14 | **calc1** (PS **1000**, LS par1 16) | ltng EMin–EMax with mastery rolled straight into the ltng slot; do14 uses calc1 as the chain radius |
| Guided Arrow, arrows, Multishot, Strafe | missiles (§8) | GA calc1; Multishot calc4 → missile s25; Strafe calc2 | Part C |
| Zeal, Fend, Fury | st37 PD `0x10300A90` + do13 `0x6FC6DAC0` | **calc2** | hits = calc1; PD `0x10270150` adds +20 `inc_splash_radius` to the `temp_splash` state per hit |
| Bash, Concentrate, Berserk | st32 `0x6FC473C0` | calc1 | conversion calc4 if EType (Concentrate: magic, `30+2·blvl`%); **calc2 (Bash `ln34`) added as flat points to the queued record after resist**; calc3 → attack-rate list (Berserk); aurastate applied after the hit |
| Double Swing | do70 `0x6FC47B90` → st32 body per target | calc1 | same as Bash (calc2 blank) |
| Frenzy, Berserk(do) | st78 PD `0x103014B0`, do9 `0x6FC49600` → `0x6FC47A10` | calc1 | conversion calc4 if EType (none in PD2); skill elem |
| Whirlwind, Blade Dance | st38, do76 `0x6FC48BE0` | calc1 | gate PD `0x102BED80` (Param3 frames, dual 2 hits/+6), target PD `0x102CB2C0` (crit_cb.md) |
| Leap Attack | do153 PD `0x102FBC90` (area) | calc1 | area callback §9; roll uses skill ToHit |
| Stun | no do-func; srvmissile `stunsplash` | — | missile with Skill=Stun: MinDam 2–4 (+tiers) + weapon×32/128; radius 3 (Part C) |
| Double Throw, Split Throw | do154/189 (missiles) | calc1 / calc4 → missile s25 | Part C |
| Sacrifice | st29 PD `0x10300920`, do64 PD `0x102FABE0` | calc1 | calc2 self damage % |
| Smite | st75 PD, do150 `0x6FCB9000` | calc1 | shield damage PD `0x102722D0` (charscreen: Armor min/max + Holy Shield), always hits, CB calc3/calc4 (B) |
| Charge | st72 PD `0x10300FA0`, do180 PD `0x102FF1C0` | calc1 | conversion calc4 if EType (none); skill elem |
| Joust | do162 PD `0x102FCB80` | calc1 | calc2 crit chance |
| Vengeance | st23, do174 PD `0x102FDEB0` | weapon hit, then three elemental parts = calc1..3 % of the rolled weapon damage (+ fire/ltng/cold mastery), float muldiv | READ, not reproduced |
| Poison Strike (Poison Dagger) | do164 PD `0x102FCEC0` | calc1 | poison skill elem; two records (hit + cloud) |
| Maul, Feral Rage | st56 `0x6FC64D70`, do120 `0x6FC66000` | calc1 (Maul 0, Feral Rage `ln56+Fury·6`) | charges n = min(s169+1, calc2) stored in state; aurastats evaluated with **lvl = n** (Maul `damagepercent = n·40`) |
| Rabies | st57 `0x6FC64C20`, do121 | calc1 (blank) | poison from EMin/EMax via missile `rabiesplague` |
| Hunger | do165 PD `0x102FD110` | — | state: life steal, OW, deep wounds (C) |
| Fire Claws | st23, do175 PD `0x102FE410` | weapon hit (fill) | firestorm missiles, count calc2 (C) |
| Tiger Strike, Fists of Fire, Cobra Strike, Claws of Thunder, Blades of Ice, Phoenix Strike | st23/78, do170 PD `0x102FD5B0` | **0** on the charging hit | charges (max 3) then finisher = `srvprgfunc[n]` via the do-func table (Part C for finisher effects) |
| Dragon Claw | st25 `0x6FCB1B40`, do46 PD | calc1 | two hands by weapsel 3 |
| Dragon Talon, Dragon Flight | st24 / do42 `0x6FCB5FE0` → `0x6FCB2BB0`; do178 PD | **calc2** (PD patch `0x6FCB2C5A → 0x1026EAB0` replaces the stock inline `Param1+(lvl−1)·Param2`) | kick damage §6 |
| Dragon Tail | st27 `0x6FCB2940` → kick, do50 PD `0x102FA2F0` | **0 on the kick** | explosion: new kick roll × (100 + calc1 + s329)/100 as fire in an area (READ) |
| Blade Sentinel/Fury/Shield, Blade Dance | missiles / area / WW | see Part C | |

PD `0x102720C0` (VERIFIED, `harness/edtab.c`) is the lookup used by area/splash hits for "the current skill's
damage %": calc1 for 0 Attack, 10 Jab, 96 Sacrifice, 97 Smite, 107 Charge, 126 Bash, 133 Double Swing,
144 Concentrate, 147 Frenzy, 151 Whirlwind, 152 Berserk, 232 Feral Rage, 260 Dragon Claw, 378 Joust,
380 Blade Dance; calc2 for 30 Fend, 106 Zeal, 248 Fury, 255 Dragon Talon, 265 Cobra Strike, 275 Dragon Flight;
**0 for every other skill**.

## 5. Masteries (READ; PD mastery chooser VERIFIED in hit_pd2.md)

- Melee: PD `0x102727D0(unit, weapon, skill, mode)`: mode 0 → 342 (AR), 1 → 343 (damage), 2 → 344 (crit); the
  throw set 345/346/347 when the weapon's ItemType is Throwable (#11088 → #10108) **and** the skill's itypea1 is
  throwable. Value = PD `0x102D30B0` typeParam: the **sum** of the stat's entries whose layer (passiveitype) is an
  item type the `#10061` weapon matches, **layer 0 excluded**, with the hand rules:
  - layer 114 `2han`: only with the other hand empty (or the same item);
  - layer 116 `1han`: on a non-2han weapon always; on a 2han-typed weapon only with the other hand occupied; and
    for Barbarian/Assassin (#10747) a 2han weapon that is not `1han`-typed also counts when the other hand is
    occupied.
  - Item-type test `#10744` uses Weapons.txt `type` **and** `type2` (the `engine.js` `itemIsType` /
    `passiveLayerMatches` copy matches this; checked against the disassembly in this pass).
- Missiles (bows, throws, Guided Arrow, Multishot, javelin skills): D2Common `#10413 0x6FDBBAA0` calls **stock
  `#10804 0x6FD9F640`** (not PD-patched here): **maximum** (not sum) of matching layered entries, no 1han/2han hand
  rules; throw set 345/346/347 via `0x6FD9F200` when the weapon is Throwable and the skill's itypea1 is type 48
  and range type 2.
- Mastery values (DATA): Sword `ln34` (P3 28,P4 5), Claw and Dagger `ln34` (40,15), Two-Handed Weapon `ln34`
  (45,15), One-Handed Weapon (layer `1han`), Mace, Spear, Throwing (346 `ln34`), Javelin and Spear
  (343 and 346 `ln12`, crit multiplier `edmn` on `spea`).
- Elemental masteries for the skill's own elemental: `#10121/#11091` → PD `0x10268A50` (fire 329, ltng 330,
  cold 331, poison 332, **magic 357**) multiply the synergised value (skilldmg.md, VERIFIED).

## 6. Kicks — PD 0x1026EAD0 (VERIFIED, harness/kickpd.c 30,000 cases)

Reached from D2Game `0x6FCB2A22`/`0x6FCB2C82` (stock `0x6FCB2360` replaced via PD `0x102EDA70`) and PD do50.
```
dex = s2
kmin,kmax,kpct  ← D2Common #10323 0x6FDA1C60 (READ):
     kmin = kmax = s137 (item_kickdamage)
     boots worn: kmin += Armor.mindam + s137 ; kmax += max(Armor.mindam, Armor.maxdam) + s137   (s137 counted twice)
                 kpct += s17 + max(trunc(str·bootsStrBonus/100)+trunc(dex·bootsDexBonus/100)+s25, −90)
                 (weapon stat lists detached while reading, so the weapons' own 17/25 are not included)
wmin/wmax = the wielded weapon's own stat-list 21/22 (only if it is a weapon, #10744 type 45 + #11006/#11160)
min = ((s21 − wmin + s111)<<8) + (kmin<<8) + trunc(dex·256/4)
max = ((s22 − wmax + s111)<<8) + (kmax<<8) + trunc(dex·256/3)
pct = rec+0x0C (skill %: Talon/Flight calc2, Tail 0) + kpct + 100 (+s122 undead, +s121 demon)
min = trunc(min·pct/100), max = trunc(max·pct/100)    (double, PD 0x102CE840, cap 0x7FFFFFFF)
phys = min + PDrand % (min<max ? max−min : 1)         (PD rand 0x102C5D10)
→ crit/DS PD 0x1026F8C0 → fill 0x6FCFD450 with a5=1 (no weapon, elemental from non-weapon items ×128/128)
→ skill elemental 0x6FCBF210
```
- So kicks use: off-weapon +min/+max (charms, rings…), `item_normaldamage`, Dexterity (¼ to min, ⅓ to max,
  before the %), boots kick damage and boots Str/Dex bonus, s25, s17 (as a single "max" % used for both ends),
  **no weapon base damage, no Strength (except through boots StrBonus), no masteries, no s18**.
- JS: `damage.kickDamage(o)` (returns the min/max in 1/256 points).

## 7. Elemental damage in melee — 0x6FCFCD80 (VERIFIED, damage.md §3)

`v = existing + ((min<<8)·(1+m/100)) + rand(range)` per element with `(min,max,mastery)`: fire 48/49/329,
ltng 50/51/330, cold 54/55/331, magic 52/53/**357** (stock stat in this spot), then × SrcDam/128. Poison 57/58/332
per frame, length = s101 or s59 ÷ max(s326,1), × SrcDam/128. The mastery multiplies only the item part.
Elemental from the non-swinging weapon is excluded for that hit (§3).

## 8. Missile weapon damage (bows, throws) — D2Common #10413 0x6FDBBAA0 (READ; details Part C)

- SrcDam from the linked skill (Missiles `SrcDamage` 255 → 0).
- Weapon base: **s159/s160 when the weapon is Throwable** (`0x6FD74C30`), else grip 2 → 23/24, else 21/22; ×src/128.
  A missile flag + bow/xbow check (`0x6FD74BF0`) halves SrcDam.
- pct = stock #10804 mastery + str·StrBonus/100 + dex·DexBonus/100 + s25 (+ Dex for mercs `#11104`), s18/s17 for
  min/max; s121/s122/s120 copied into the missile record; the skill's damage % arrives as missile **stat 25** set
  by the do-func (Multishot calc4, Split Throw, Double Throw) or through the skill-linked calcs.

## 9. PD2 splash and area hits

### Melee splash (stat 359 `item_splashonhit`, Properties `splash` → skill 358 `proc_SplashDamage`)
- Event `domeleedamage` func 20 (skill-on-event) casts skill 358; do-func 163 = PD `0x102FCCE0` (READ).
- **Radius** = Skills 358 calc3 `max(6 − lvl + stat('inc_splash_radius'.accr)/20, 0)` + typeParam(478)/20
  (the layered part, e.g. Two-Handed Weapon Mastery `min(20+lvl,40)`) − 1 in one special case (`0x1029A300`).
  `accr` goes through PD's stat hook: `478 += min(489 × missing_hp%, 40)` when 478 > 0. Sources of 478 at layer 0:
  items, Werebear (20), Concentrate/Berserk state (20), Zeal/Fend/Fury `temp_splash` (1 + 20 per hit).
- **Damage** = a full new melee record for each other enemy in the radius (area iterator `0x102EC960`, flags
  0x8583, callback PD `0x1026F8F0`, the originally hit target excluded):
  - no hit roll (always hits); SrcDam and HitFlags/ResultFlags/HitClass of the **current skill**;
  - damage % = PD `0x102720C0(current skill)` (§4 list) — skills not in the list lose their damage %;
  - the fill (weapon + item elemental), then resist (B) on the local record, **then** the current skill's own
    elemental (`0x6FCBF210`) and physical (`0x6FCBF1A0`) are added — after resistance and after the total, so only
    their poison/chill lengths take effect (VERIFIED, `exploit_checks.md` §2);
  - life leech % (rec+0x38) ÷ ctx divisor 2 (min 1); the ctx %-scale (+0x38) is 0 for splash (no reduction);
  - Dragon Talon/Tail/Flight (ids 255/270/275) instead copy the attacker's last queued kick record (already
    resisted against the main target) and resist it again.
  - applied through PD `0x10271CF0 → 0x10271E00` (B).
- There is **no splash damage fraction** in the code path: a splashed enemy takes the same roll formula as the
  main target (READ).

### Leap Attack and the Blade Creeper AI (same callback)
- Leap Attack do153: ctx damage % = calc1, ToHit = skill ToHit (+ stat 119 in the roll: AR% counted twice, hit_pd2.md),
  radius from calc, leech divisor 3, no %-scale; plain Leap (132) skips the damage.

## 10. Character screen vs server (D2Client, READ/VERIFIED in charscreen.md, skilldmg.md)

| item | server | character screen |
|---|---|---|
| s111 item_normaldamage | total of all items, added **before** the %, ×SrcDam | weapon's own s111 only, added **after** the % |
| unarmed base | min = s21 or 1, max = s22 or 2 | s21+1 .. s22+2 |
| top of the roll | never reaches max (rand [0,n)) | shows max |
| demon/undead/montype % (121/122/180) | added | not shown |
| crit / DS / CB / OW / splash | applied | not shown |
| Power/Charged/Lightning Strike calc1 % | added to the melee hit | not shown (descdam 5/17) |
| Bash calc2 flat | added after resist | not in the damage box |
| Concentrate magic conversion | physical → magic | not shown |
| kicks | PD 0x1026EAD0 (§6) | PD descdam 15/16 handlers (float, READ) |
| masteries | PD 0x102727D0 (melee), stock #10804 (missiles) | PD 0x102728B0 |
| dual wield | per swing with the other hand's stats detached | per hand (0x6FADD500) |

## 11. PD2 hooks on this path

| site | kind | effect |
|---|---|---|
| D2Game 0x6FCFC6A1 → PD 0x102727D0 | static rel call | melee mastery: throw/melee set, sum of layered entries with hand rules |
| D2Game 0x6FCFD52C–0x6FCFD5B4 → PD 0x102ED930 | static, 131 NOPs | crit/DS replaced (B) |
| D2Game 0x6FCFD746 → PD 0x102ED2A0 → 0x1026ED30 | static | monster mana-drain roll scaling (monster attackers only) |
| D2Game 0x6FC6DC47 → PD 0x10270150 | static | Zeal/Fend/Fury: +20 inc_splash_radius per hit |
| D2Game 0x6FCB2A23 / 0x6FCB2C83 → PD 0x102EDA70 → 0x1026EAD0 | static | kick damage replaced (§6) |
| D2Game 0x6FCB2C5A → PD 0x102ED1C0 → 0x1026EAB0 | static + NOPs | kick damage % = calc2 |
| D2Game 0x6FC4966F / 0x6FC496B0 → PD 0x102F1040 → 0x10272980 | static | Frenzy per-hit handler replaced |
| D2Game 0x6FC48CE5 / 0x6FC48D22 | static | Whirlwind hit gate / target (crit_cb.md) |
| srvdofunc table 0x6FD274A8 (via 0x104E3114), srvstfunc table 0x6FD27338 (via 0x104E30E0) | code-init writes PD 0x102BE412/0x102BE69F | do 4,6,8,25,39,44–51,54–57,64,71,104,112–115,118,119,139,143,148,153–189; st 28,29,37,66–78 |
| D2Common 0x6FDA0443 / 0x6FDA054A → PD 0x102EEAA0 → 0x10268A50 | code-init | skill elemental mastery incl. magic 357 |
| D2Common 0x6FD88BC5 / 0x6FD891E4 / 0x6FD893CB / 0x6FD8947E → PD 0x102EDD40 → 0x102670A0 | static | stat getter: 183 equipped_eth, 184 missing_hp, 478 += min(489·missing%, 40) |
| D2Common 0x6FDBAC20… (19 sites) → PD 0x102EDD50 → 0x10267270 | static | missile elemental mastery map (Part C) |
| D2Client +0x2C348, +0x2C56E, +0x30A10, +0x3136F → PD 0x102728B0 | static | display mastery |
| D2Client 0x6FAE313B → PD 0x102ED140 → 0x102F91B0 | static | display throw damage |

Stock code on this path that PD2 does **not** change: 0x6FCFC530 except the mastery call, 0x6FCFBED0,
0x6FCFCD80, the conversion block, 0x6FC572C0/0x6FC57540 (hand choice), st32/do9/do13 bodies, #10323.

## 12. Quirks (verdicts; ids as in flow_A.json)

1. **A.q.power_strike_1000** (VERIFIED, `harness/pstrike.c`; likely bug) — Power Strike calc1 = 1000 is read by
   st10 as the melee hit's damage % (+1000% on the weapon roll, ×11 with no other %) and by do14 as the nova-target
   search radius. The tooltip shows no %. Details: `exploit_checks.md` §1.
2. **A.q.calc1_as_ed** (VERIFIED; intended but surprising) — Charged Strike (+3..12% = its bolt count) and Lightning Strike (+16% =
   Param1, its chain radius) get their calc1 as a melee damage %. Stock-inherited.
3. **A.q.bash_post_resist** (READ; intended but surprising) — Bash's calc2 (`ln34` = +lvl points) is added to the
   queued record after resistances: not reduced by physical resist/DR, not multiplied by ED/crit.
4. **A.q.area_skill_dmg_after_resist** (VERIFIED, `harness/splash.c`; likely bug; corrected) — in PD's area
   callback (splash, Leap Attack, Blade Creeper) the skill's own elemental/physical is added after the resist step
   **and after the total +0x4C is summed**. The execute clone takes only the total off life, so that damage is
   **not dealt at all**; only its poison rate/length and chill length get through, ignoring resistance and immunity
   (Rabies/Poison Dagger poison, Blades of Ice chill). Details: `exploit_checks.md` §2.
5. **A.q.splash_ed_table** (VERIFIED table; intended but surprising) — splash keeps the current skill's damage %
   only for the 21 listed skills; e.g. Vengeance, Power/Charged/Lightning Strike, Stun, Rabies and charge-up
   finishers splash with 0%.
6. **A.q.kick_splash_double_resist** (READ; likely bug) — splash from Dragon Talon/Tail/Flight reuses the main
   target's already-resisted kick record and resists it again.
7. **A.q.kick_formula** (VERIFIED; intended but surprising) — kicks ignore weapon base damage, Strength (except
   boots StrBonus), masteries and s18; Dexterity adds dex/4 to min and dex/3 to max before the %.
8. **A.q.kick_137_twice** (READ; likely bug, stock) — with boots, #10323 adds s137 `item_kickdamage` twice.
9. **A.q.dragon_tail_no_ed** (READ; intended but surprising) — the Dragon Tail kick itself gets no skill %;
   calc1 only scales the fire explosion.
10. **A.q.missile_mastery_max** (READ; unclear) — missiles use stock #10804: the largest matching mastery entry,
    no hand rules; melee uses PD's sum with hand rules.
11. **A.q.max_never_reached** (VERIFIED; intended but surprising) — every weapon roll is `min + rand(max−min)`.
12. **A.q.normaldamage_scope** (VERIFIED/READ; intended but surprising) — s111 from all items, before the %.
13. **A.q.concentrate_magic** (DATA+READ; intended but surprising) — Concentrate converts `30+2·blvl`% of
    physical damage to magic (after crit, before resist). PD2 Berserk no longer converts (EType blank).
14. **A.q.zeal_splash_growth** (READ; intended but surprising) — Zeal/Fend/Fury add +20 inc_splash_radius per
    hit (+1 splash radius per consecutive hit) while the sequence lasts; needs a splash source.
15. **A.q.dual_mastery_primary** (READ; unclear) — masteries match the inventory "primary" weapon (#10061).
16. **A.q.stun_missile** (DATA/READ; intended but surprising) — Stun has no do-function: its hit is the
    `stunsplash` missile (radius 3, 25% weapon damage via SrcDam 32, plus MinDam).
17. **A.q.maul_charge_lvl** (READ; intended) — Maul's state stats are evaluated with lvl = charge count.
18. **A.q.charge_hit_no_ed** (READ; intended but surprising) — charge-up skills' charging hits carry no skill %.

## 13. Discrepancies with our JS models

- `engine.js` mastery matching (type + type2, `passiveLayerMatches`) matches PD `0x102D30B0` (re-checked); no change.
- `engine.js`/`combat.js` apply the melee mastery rules to bow/throw damage; the server uses stock #10804 there
  (max, no hand rules). Only matters with two matching mastery entries — not changed.
- `skilldmg.js` / the page do not model: Power/Charged/Lightning Strike calc1 %, Bash post-resist flat,
  Concentrate conversion, kick damage (now available as `damage.kickDamage`), splash.
- Added `damage.kickDamage` (additive). `adv/engine/test_*.js` and `harness/damage_check.js` still pass.

## 14. Open questions

- Exact per-sequence event counts (hits per Jab/Frenzy/DS/Dragon Claw sequence) — see skill_speed.md.
- Vengeance (PD do174) and Dragon Tail explosion arithmetic (float muldivs) — READ only.
- Charge-up finisher (srvprgfunc) damage — Part C.
- inventory+0x1C update timing for dual wield (which weapon the mastery check sees per swing).
- `0x1029A300` condition that shrinks the splash radius by 1.
