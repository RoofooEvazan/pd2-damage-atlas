# Part C: spell, missile, damage-over-time, aura, curse, summon and mercenary damage (PD2 = 1.13c + ProjectDiablo.dll)

Part of the 4-way damage deep dive. A = weapon/physical build-up (incl. weapon-based skills), B = defender pipeline
(resist, pierce, halving, crit, CB, OW, absorb, DR, leech, thorns), D = monster->player and PvP. This file covers what
happens between "a spell is cast" and "the damage struct reaches B", plus everything that ticks afterwards.
Flow data: `adv/re/flow/flow_C.json` (34 stages, 57 edges, 118 skills; schema `adv/re/flow/SCHEMA.md`; rebuilt by `adv/re/flow/gen_flow_c.py` from the data.zip tables).

**Status key** (as FINDINGS.md): **VERIFIED** = the game's own code ran natively in the harness and matched a model on
every case; **READ** = read from disassembly; **DATA** = follows from the shipped tables; **INFERRED** = deduced from READ facts.
Sources: client-install `D2Game.dll`/`D2Common.dll`/`Fog.dll` 1.13c, `ProjectDiablo.dll` (PD, base 0x10000000),
PD2 `data.zip` tables. Realm servers could run different server code.

## 0. Native checks run for this part

`harness/spells.c` + `harness/spells.py` (new). Build and run from `harness/`:

```
gcc -m32 -nostdlib -static -O1 -fno-pic -no-pie -fno-stack-protector -ffreestanding -o spells spells.c
python3 spells.py 20000
```

| mode | real code | cases | result |
|---|---|---|---|
| 1 | PD length pre-step 0x102713C0 (cold/freeze/poison/burn lengths) | 30,000 | 0 mismatches |
| 2 | D2Game missile element roll 0x6FC59B40 (+ real RNG 0x6FC211D0, muldiv 0x6FC214D0) | 30,000 | 0 |
| 3 | D2Game missile damage-struct builder 0x6FC5A4E0 (all stat slots, poison_count, %dmg, flags, RNG order) | 30,000 | 0 |
| 4 | D2Game Static Field target callback 0x6FC62EC0 (+ real struct fill 0x6FCBEFE0, alldiv/allmul) | 30,000 | 0 |
| 5 | PD damage func 17 (Multiple Shot pierce falloff) 0x102F2030 on the real type table | 30,000 | 0 |
| 6 | PD poison application 0x10271A70 (find / create / replace rule) | 30,000 | 0 |
| 7 | PD missile hit func 1 (Fire Ball) 0x102F2660: return value, radius, filter, callback | 2,000 | 0 |

Stubs: stat getters, state and stat-list helpers (fake list), timers (recorded), execute (captured), area iterator
(captured), D2Common calc #10786 (constant), path x/y. A mutation run (five deliberate model changes) produced 247-946
mismatches per mode, so the checks can fail. Earlier harnesses cover the building blocks this part relies on:
`skdmg.c` (58,455 cases: #10121/#11091 synergy+mastery, missile functions, PD mastery map), `skcalc.c` (24,315 calc
evaluations on Fog's VM), `damage.c` (resist/pierce/halving, OW, CB, leech), `minions_harness` (summon level, setters,
merc stats).

## 1. Skill damage formula (C.skill_level ... C.elem_len)

All values are 1/256 of a point. `tier(l, a1..a5)` = level brackets 2-8, 9-16, 17-22, 23-28, 29+.

```
lvl  = hard + allskills(127) + class(83) + tab(188) + oskill(97) + singleskill(107) + element(126)   (#10306 0x6FDA05C0)
blvl = hard points (skill+0x28)  -> used by every synergy calc skill('X'.blvl)
E    = (EMin + tier(lvl, EMinLev1..5)) << HitShift          (#10121 0x6FDA0460; max #11091 0x6FDA0360)
E   += muldiv(EDmgSymPerCalc, E, 100)                        min side only if E > 256 or EMinLev1 != 0
E   += muldiv(mastery, E, 100)                               (only when the caller passes mastery = 1)
P    = ((MinDam + tier(lvl, MinLevDam)) [+ weapon x SrcDam/128])  + muldiv(DmgSymPerCalc, ., 100)  then << HitShift (no mastery)
len  = ELen + bracket(ELevLen1..3) + muldiv(ELenSymPerCalc, len, 100)     frames
```

- **Claims in skilldmg.md re-checked (VERIFIED, existing harness):** mastery multiplies the synergised value (order base ->
  +syn% -> +mastery%, each truncating); the low-damage synergy gate on the min; empty `HitShift` = 0 (80 missiles and
  Poison Javelin rely on it); Fog precedence (`?` binds tighter than every operator on both sides).
- **Mastery map (PD 0x10268A50 skills, 0x10268A10 own-damage missiles):** fire 329, lightning 330, cold+freeze 331,
  poison 332, **magic 357 (PD2 only)**. Other elements: none. Masteries come from the *calc unit*: for pet missiles that
  is the pet (see §5).
- **PD2 skill-damage stats:** none of the new ItemStatCost rows is a generic "+% skill damage". Skill damage is raised by
  levels (stock stat 126 per element; the PD2 rows 362-366 `item_elemskill_*` are not referenced by PD code - OPEN whether anything reads them), synergies, masteries (329-332, 357) and per-skill stats used in calcs
  (443 extra_bonespears, 481 extra_holybolts, 463 extra_hydra, 485 corpseexplosionradius, 460 gustreduction, 490
  eaglehorn_raven...). Resist reduction is pierce (333-336, 358 magic, 425 phys) or curses/auras.
- **SrcDam skills** (weapon share) are part A. For spells SrcDam is 0. A missile with `SrcDamage = -1` (byte 255) uses
  SrcDam 0 even if its skill has one (poison javelin clouds, fury bolts, poison dagger cloud).
- **Rolls:** a skill-linked missile stores min and max (1/256) at creation; the roll happens at each hit (§2.4).
  Auras roll once per tick for all targets (§4.1); PD splashes roll once per explosion (§2.6); Static Field and CE
  compute from the target/corpse (§4.4/4.5).

## 2. Missiles

### 2.1 Creation (C.missile_create) - D2Common #10413 0x6FDBBAA0, stats #10484 0x6FDBB4A0 (READ, parts VERIFIED)
- Skill-linked (Missiles `Skill` column, or `MissileSkill` flag -> the owner's current skill): phys #10567/#10297,
  elem #10121/#11091 **with mastery**, length #10510, element from the skill's EType, all at the **missile level**
  (the casting skill's level; sub-missiles inherit it). SrcDam = Skills.SrcDam unless Missiles.SrcDamage = 255.
- Own-damage missiles (no Skill): Missiles.txt MinDamage/EMin... at the missile level; mastery only with `ApplyMastery`.
- SrcDam != 0: item elemental damage added (PD hook 0x102EDD90), then every slot x SrcDam/128 (0x6FDB9C60).
- Damage struct -> missile stats: 21/22 phys (<<8), fire 48/49 + **fire length 315**, ltng 50/51, magic 52/53,
  cold 54/55/56, poison 57/58/59 (+326 poison_count), life/mana/stamina drain 60-65, stun 66, burn 316/317 (length 315),
  25 damage% (+121 vs demon, 122 vs undead, 120), 141 DS, bypass flags 103/104/106, 180 damage_vs_montype list.
  **Masteries are not copied** (the element roll at hit time reads 329... from the missile and gets 0; no double count).
- `DamageRate` -> missile stat 327 `damage_framerate` (D2Game 0x6FC8FEA9); copied into damage struct +0x54 at hit and
  used to scale the target's flat DR/MDR by DamageRate/1024 (B.dr/B.mdr). Values: 24 (Fire Arrow trail), 41 (fire
  walls, Immolation, Meteor/Armageddon fire, Firestorm, Shock Web), 82 (Inferno, Inferno Sentry), 205 (Wake of Fire).

### 2.2 Collision (C.missile_collide) - PD 0x102723E0 replaces stock 0x6FC5EDE0 at all 12 callers (READ)
1. Owner/target filters; `NextHit` check (PD 0x102F42A0): the target carries timers type 0xF keyed
   (root player id, missile class) - a missile of that class from that player skips the target until NextDelay
   frames have passed (PD 0x102F4320 records it on hit). Non-player owners use key (-1,-1).
2. **Attack roll only if Missiles.ToHit = 1** (108 missiles; all bows/javelins; no player spell except Immolation/Blaze
   ignite missiles and Blade Sentinel). Spells never miss. The roll is PD 0x10271740 (FINDINGS "Hit roll").
3. `pSrvHitFunc` (table D2Game 0x6FD2DA48, 71 slots). Return bits: 1 = end, **2 = apply the missile's own damage to
   the struck unit**, 4 = stop now. On a miss with `AlwaysExplode` the hit func still runs.
4. Direct damage (bit 2): struct from the missile's stats (0x6FC5A4E0 via PD 0x1026E9F0), `pSrvDmgFunc`, then apply
   0x6FC5AFF0 -> 0x6FC5AE10 (hit class, knockback, stat 327 -> +0x54) -> execute 0x6FCFE0C0 with resist flag 1.

### 2.3 Hit functions that matter for player skills
| id | code | used by | behaviour |
|---|---|---|---|
| 1 | **PD 0x102F2660** | Fire Ball, Combustion, Hydra fireball, explosion sub-missiles (Exploding/Freezing Arrow, Raven, Bone Spirit) | splash radius sHitPar1 (or skill calc1, min 1) with one roll; **returns 3 when a unit was struck -> that unit also takes the direct hit** (stock returned 1). VERIFIED return value |
| 62 | PD 0x102F3770 | Fire Bolt, Ice Bolt, Blizzard shards | builds the *skill's* damage at missile level + sHitPar2, splash radius sHitPar1 (3/4), returns 1 (no direct hit) |
| 63 | PD 0x102F38A0 | Fire Arrow, Exploding Arrow | spawns HitSubMissiles on the target (collide immediately) and a line of `firearrow firewall` patches |
| 7 | PD 0x102F28E0 | Holy Bolt, FoH bolts, Holy Nova | heals allies (roll between two skill fields) else damages; sHitPar2 0 = **every monster** (stock: undead only) |
| 50 | PD 0x102F31A0 | Inferno/Arctic Blast debuff, Vine trail | while missile frame < range - sHitPar1: apply the skill's auratargetstate (-res debuff), return 2 (damage too) |
| 61 | PD 0x102F3590 | Poison Strike cloud | PD cloud |
| 12 | stock | Chain Lightning, Lightning Strike, Psychic Hammer, CL Sentry | re-target chain |
| 14 | stock | Meteor, Phoenix/FoF meteors | impact + fire field |
| 22 | stock | Fist of the Heavens | bolt + holy bolt ring |
| 29 | stock | Frozen Orb | releases bolts |
| 5/36/3 | stock | Power Strike nova, Thunder Storm, Shattering Arrow, traps | novas/bombs |

### 2.4 Per-hit roll (C.missile_roll) - D2Game 0x6FC5A4E0 (VERIFIED, mode 3; element roll 0x6FC59B40 mode 2)
```
phys  = (s22 > 0 && s21 > 0) ? min + rand(max - min) : 0            (swap if min > max)
elem  = (min > 0 && max > 0) ? min' + rand(max' - min') : 0         min' = min + muldiv(M, min, 100), M = missile stat (0)
order : phys, fire, magic, lightning, cold, (cold len = s56), poison, (len = s59 / s326 if s326 > 1), s62, s60, s64,
        burn (316/317, len 315), stun s66
target: phys += trunc(phys x (s25 + demon?s121 + undead?s122)/100) with pct >= -90 (32-bit product)
flags : s141 -> DS (stock x2 here, PD redirects the block to 0x10270C50), s103/104/106 -> bypass flags 0x100/0x200/0x400
rand(n): seed at missile+0x20/0x24, lo' = lo*0x6AC690C5 + hi (64-bit), result = lo' mod n; n <= 0 -> 0 without advancing
```
Consequences: each missile of a multi-missile cast rolls separately; the %damage on a missile only multiplies its physical
part; an element whose minimum is 0 deals nothing.

### 2.5 Damage functions (C.missile_dmgfunc) - table 0x6FD2DB68; PD writes 15-19 at 0x102F2320
| id | code | users | effect |
|---|---|---|---|
| 1 | 0x6FC59CE0 | Magic/Fire/Cold/Guided Arrow | convert min(100, DmgCalc1)% of physical to the missile's element (A) |
| 3 | 0x6FC59670 | fire walls, meteor fire, Immolation fire | rand(128) < dParam1 (19) sets result flag 0x4000 (hit-recovery chance per tick) |
| 5 | 0x6FC5A790 | Blessed Hammer | + base x dParam1% vs undead, dParam2% vs demon; **PD2 row has both empty** |
| 15 | PD 0x102F1E20 | Psychic Hammer | magic x (100 - (calc1 - n) x calc2)%, 0 when >= 100% (n = missile step count) (READ) |
| 16 | PD 0x102F1F30 | Double Throw | x calc3% when calc2 == n (A) |
| 17 | PD 0x102F2030 | Multiple Shot | every damage slot x max(20, 100 - 20(pierce_count-1))% (VERIFIED) |
| 18 | PD 0x102F2100 | rathmabonespear | boss |
| 19 | PD 0x102F2230 | Blade Fury | (A) |

### 2.6 Splash (C.splash) - PD 0x10271BD0 via callback 0x102710A0 (READ)
One struct per explosion; for each unit within the radius passing filter 0x8583 (players+monsters, enemies, not in
town): block/avoid/evade check PD 0x1026FCF0 (block -> 0x200, evade/avoid -> 0x8000, hit bit cleared), then execute
with resist. No attack rating roll. The owner is never included (area iterator skips it).

### 2.7 Crit on missiles (C.missile_crit) - D2Game 0x6FC5A727 -> PD 0x102EEBD0 -> 0x10270C50 (READ)
Direct owner found -> PD 0x10270E00 with the owner's stats and the missile's skill: crit = weapon-matched 344 + 337 +
258 (cap 75) -> x(200 + 256 + typeParam)%, else DS 141 (cap 75 + 210) -> x(150 + 257)%; **physical slot only**.
So Tornado, Twister, Meteor impact, Volcano, Molten Boulder, Armageddon rocks and Shock Wave can crit/deadly-strike with a
Druid/Sorceress's 258/337/141 items. Missiles of pets use the pet's stats.

### 2.8 Per-frame fields and pierce
- Collision=1 missiles (pSrvDoFunc 5 etc.: Fire Wall, Immolation fire, Meteor fire, Molten Boulder path, Armageddon fire,
  Firestorm, Blaze patches, Shock Web, FoF firewall) hit every unit under them each frame with `(E << HitShift)/256`
  points. Example Fire Wall: EMin 6 HitShift 3 -> 48/256 = 0.19 points per frame = 4.7 per second at level 1 before
  synergy and mastery. Flat DR/MDR is scaled by DamageRate/1024 (§2.1).
- `Pierce` column + PD stat 469 `pierce_count`; Multiple Shot loses 20% per extra pierce (func 17). NextHit decides
  whether one missile can hit the same unit again (Bone Spear 5 frames, Tornado 25).
- Multiple missiles: PD do-func 8 (0x102F9610): count = calc1 (Bone Spear 1 + lvl>=15 + lvl>=25 + extra_bonespears;
  Holy Bolt likewise + extra_holybolts; Teeth min(ln12,24)); side groups get flag 0x10000; stat 25 = calc4 on each.
  Rings: stock do 22 (Nova, Frost Nova, Poison Nova, Holy Nova, War Cry), PD 0x102BF8A0 (Combustion 32 missiles).

## 3. Damage over time

### 3.1 Length pre-step (C.len_prestep) - PD 0x102713C0 (VERIFIED, mode 1)
```
if cold_len or freeze_len (unsigned > 0):
    stat153 cannotbefrozen != 0      -> both 0         (chill removed too)
    stat118 halffreezeduration == 1  -> both >>= 1     (logical shift)
    stat118 >= 2                     -> both 0         (PD2; stock halved for any nonzero)
poison_len > 0 and state 133 -> 0;  burn_len > 0 and state 131 -> 0
```
Poison length is also reduced by stat 110 through the resist table (entry 7, B.resist).

### 3.2 Poison (C.dot_poison) - PD 0x10271A70 (VERIFIED, mode 6)
```
if !(len > 0 || rate > 0) return                     rate = poison +0x28 after resist (1/256 per frame)
monster defender: its regen timer (type 3) is re-armed for the next frame
end = frame + len
list = attacker is player            ? per-source state (PD 0x10269230, keyed by attacker type+id)
     : root owner is a player        ? per-source state keyed by the MINION unit
     : shared state 2
if !list: create state 2 with hpregen(74) = -rate
if -rate <= list.74: list.74 = -rate; expire = end; timer 0xC at end     (new rate >= old rate: replace + restart)
else: ignore                                                             (weaker poison dropped, even if longer)
```
- One poison per player per monster; never stacks. A stronger or equal poison restarts the timer at its own length
  (can shorten it). Each player, each merc and each Poison Creeper keeps its own.
- Tick: D2Game 0x6FC96740 every frame: life += stat 74 total (monster natural regen + all poison/burn/OW drains).
  Death is credited to the poison / OW source (states 2 / 62).
- Stock 1.13c kept one poison state per monster regardless of source.
- Totals: points per second = rate x 25/256; per poison = rate x len/256. Skill poison: rate = E (with mastery, HitShift
  usually 3-5), len = #10510. Example Poison Nova: EMin 16 << 4 = 256 -> 1 point/frame at level 1.

### 3.3 Burning (C.dot_burn) - D2Game 0x6FCFC940 (READ)
Needs rate > 0 **and** length > 0; state 115, hpregen = -rate; replaced when the new rate >= old; one state per monster.
Burn is not in the resist table (unresisted, unshared). Only EType `burn` (11) fills it (missile stats 316/317, length
315). In PD2 data the only such missile is `dragonflightmaker`, which no skill spawns, so burning is effectively unused.
A fire missile with ELen (e.g. tigerfurytrail) sets stat 315 but no rate -> nothing.

### 3.4 Chill and freeze (C.chill) - D2Game 0x6FCFC780 / 0x6FCFDAC0 (READ)
Chill value from 0x6FCFAE20; against monsters the length is divided by DifficultyLevels MonsterColdDivisor (1/2/4),
minimum 1 frame; state 11 with velocitypercent/attackrate/other_animrate. PD 0x102C0AD0 (hook 0x6FCFC88E) also writes
FCR (stat 105, not below cancelling the unit's own FCR) and leap speed (423 = 3 x chill) - relevant to players (part D).
Freeze uses MonsterFreezeDivisor likewise.

### 3.5 Other ticks
Open wounds (B.ow, PD 0x102AF060) uses the same regen tick. PD2 Thorns aura grants open wounds and deep wounds.
Blood Golem: item_openwounds 100 + deep_wounds edmx. Hunger: open wounds dm12+5.

## 4. Auras, curses and special spells

### 4.1 Offensive aura ticks (C.aura_tick) - do-func 66 0x6FCBAF50, 81 0x6FCBABC0 (READ)
Every `perdelay` = 25 frames: struct from 0x6FCBF210 = one roll `min + rand(max-min)` of #10121/#11091 **with
mastery**, element = EType; flags |= 0xD; ResultFlags/HitFlags from Skills; every enemy in `aurarangecalc` passing
`aurafilter` gets that struct through execute with resist (callback 0x6FCBA7C0). Holy Fire 5-11 << 6 (E/4 points),
Holy Freeze << 7 + chill aurastats, Holy Shock << 7, Sanctuary << 6 magic + magicresist -(5+lvl/2) (and physical resist
0 vs non-prime-evil undead for the owner's attacks, state 47). The owner also gets passivestats (e.g. Holy Fire
firemindam/maxdam = edns x par5/256, synergised, no mastery) which the attack's elemental roll (0x6FCFCD80, part A)
multiplies by the owner's mastery. Fire Golem's "Holy Fire Fire Golem" uses the golem's copied fire mastery.

### 4.2 Stat auras and buffs (C.aura_stats)
Might, Concentration, Fanaticism, Heart of Wolverine, Enchant (fire min/max = edmn/edmx, no mastery), Cold Enchant, Venom
(poison min/max edns/edxs, override length edln) are stat lists evaluated with the caster (auras.md). Conviction:
fire/cold/light resist -min(ln34,150), defense -dm56; with the PD immune rule (base > 99 -> each negative contribution
halved, PD 0x102C0540, VERIFIED in damage.md).

### 4.3 Curses (C.curse_apply) - do-func 30 -> PD 0x102BFE20 (READ)
```
r = target stat 109 curse_resistance;  if r > 75 and target is a player: r = 75
length = trunc(length x (100 - r)/100)  (double);  0 -> no curse          stock: r >= 100 immune, length -= length*r/100
e = target stat 504 curse_effectiveness (cap 75 for players): the curse's carried stat value x (100 - e)/100
curses allowed = caster stat 368 max_curses + 1 (Curse Mastery gives blvl/10); state 196 Dark Pact consumes them
```
Amplify Damage: damageresist -(par5 + par6 lvl + CurseMastery/2). Lower Resist: fire/light/cold/poison
-(par5 + par6 lvl + CurMas/2). Decrepify: damageresist -(10 + lvl/3), speed, FCR, leap. Iron Maiden: calc1 % reflect of
the monster's melee damage (B.thorns). Life Tap: calc1 % leech for attackers (B.leech). All -res curses on base >= 100
are halved by PD 0x102C0540 and count before the immunity test (damage.md §2).

### 4.4 Static Field (C.static_field) - PD do 160 0x102FC760 -> stock callback 0x6FC62EC0 (VERIFIED, mode 4)
```
L = life >> 8;  if L < 1 or L <= muldiv(StaticFieldMin, maxlife >> 8, 100): no hit
d = trunc(L x calc1/100) (25%, big-value branches), if L - d < 1: d = L - 1;  d <<= 8;  d = max(d, calc2)
res = target resist of the element (lightning 41):  if res < 0: d = d x 100/(100 - res)   (100 - res == 0 -> 0)
then normal execute: pierce, PD negative-resist halving, DR, resist
```
StaticFieldMin in PD2 = 55 / 70 / 85 % (normal / nightmare / hell; stock 0/33/50). Filter PD 0x1026F150 skips monster
classes 0x315, 0x3A5-0x3A8 (Rathma/Mendeln), 0x458. The -lightning-resist debuff (aurastat, length 125 + 5 x Lightning
Mastery blvl) is applied as a state after the damage.

### 4.5 Corpse Explosion (C.corpse_explosion) - PD do 55 0x102FA7F0 (READ)
Corpse life = (MonStats x MonLvl HP min + max) << 7 (the average, 1/256) for the corpse's class and level (normal /
L- columns by game type), players: stat 7. Roll lo..hi = life x calc1 .. life x calc2 % (PD2 par1 5, par2 10; double
math, 0x102CE840), scaled by casterLvl/corpseLvl when the corpse is higher. calc3 = 50 % of it goes to the skill's EType
(fire), the rest stays physical. The skill's own #10567/#10121 damage (phys 2-5, fire 2-5 + tiers + synergies) is added.
Radius aurarangecalc (with corpseexplosionradius past the 32 cap, Fog precedence). Area callback 0x1027D970. The same
do-func serves `mon death sentry` (Death Sentry). DifficultyLevels MonsterCEDamagePercent is not read here.

### 4.6 Other spells
- Blessed Hammer: skill-linked magic E with magic mastery 357; dmg func 5 without dParams (no undead bonus in PD2 data);
  each hammer hits each monster it passes once.
- Fist of the Heavens: lightning bolt (hit func 22) + `fistoftheheavensbolt` ring (own damage EMin 10-12 + EDmgSym,
  ApplyMastery -> magic mastery) hitting all monsters (hit func 7).
- Bone Spear/Spirit, Teeth: skill-linked magic; NextHit 5/4 -> one spear/tooth per monster per window.
- Frost Nova/Nova/Poison Nova/Holy Nova: ring, NextDelay 4 -> one hit per monster per cast.
- Inferno/Arctic Blast: PD do 182 streams debuff missiles (hit func 50): per-frame hits (E<<5 / E<<6) + a -resist state
  refreshed while the stream touches the monster.
- Thorns/Bone Wall/Bone Prison/Iron Golem/Spirit of Barbs: stat 78 reflect (B.thorns).

## 5. Summons and mercenaries

### 5.1 Summon build (minions.md, VERIFIED building blocks)
- Level (0x6FC6E2A0): min(clvl, max(1, 3clvl/4 + lvl)) for Valkyrie, skeletons, golems, sentries, Blade Sentinel; Raven
  clvl + par1 + lvl; spirits/wolves/bear clvl; vines 3clvl/4 + lvl; shadows clvl; hydras their spawn level.
- Setter 0x6FC6F970 (PD wrapper 0x102CA430), **owner as calc unit, once at summon time**: passivestats, aurastats,
  calc1 %life, sumskill levels.
- **Mastery copies (DATA):** Hydra, Lesser Hydra, every sentry, Fire Golem: `passive_*_mastery = stat(...).accr` of
  the owner; Skeletal Mage: SM.lvl x SM.par3 + owner mastery (all four); Raven: cold damage aurastat already multiplied
  by (100 + owner cold mastery); Poison Creeper: poison mastery from its synergies. Snapshots, not live.
- **Synergy copies (DATA):** traps and hydras list their synergy skills as sumskills with level = owner's hard points, so
  the pet's own `skill('X'.blvl)` equals yours. The pet casts its attack skill at the trap/hydra level (`lvl`), the
  missile is skill-linked to the player skill (Lightning Sentry, Hydra...) so its damage is that skill's EMin/EMax at
  that level with the copied synergies and mastery.
- **Pierce:** read from the attacker (the pet). Only Blade Sentinel (passive_phys_pierce = your 425/2) and Valkyrie
  (passive_ltng_pierce = 2 x Pierce hard points) get any. Negative-resist halving applies to pets (root owner is a player).
  Curses/auras on the monster help pets like everyone.
- **Crit:** pets' own stats only (usually none); pet missiles resolve the pet as owner.
- Physical pets: attack rebuilt each attack from MonStats x MonLvl (noRatio = raw) + stats; SkillDamage pets (Raven,
  Spirit Wolf, Dire Wolf, Grizzly, Decoy) add the skill's MinDam/MaxDam at the owner's level (0x6FCBE330).
- Revives: corpse monster at min(mlvl, clvl).
- Poison from pets keys the poison state by the pet (each creeper/mage separately).

### 5.2 Mercenaries (minions.md)
Hireling.txt row by (Version 100, Id, Level); per-level growth VERIFIED (0x6FC68BA0). Merc damage = its weapon + items +
Hireling 2H base damage 23/24, its own str/dex/masteries (none), own auras (do 65 aurastats only), party auras from the
player (aurastats only). Merc spells (Act 3 fire/cold/lightning, Act 1 fire/cold arrow via `Merc Fire Arrow` missiles)
are skill-linked missiles with the merc's skill level and no masteries. Owner stats do not transfer; negative-resist
halving applies (owner player); monsters' leech/drain rules as B. Merc resist loop also runs entries 9-11 (merc leech).

## 6. Addresses and PD2 hooks on these paths

| where | stock | PD2 |
|---|---|---|
| srvdofunc table | D2Game 0x6FD274A8 | PD 0x102BE412 writes 4,6,8,25,39,44-51,54-57,64,71,104,112-115,118,119,139,143,148,153-189 (map in §7) |
| srvstfunc table | 0x6FD27338 | PD 0x102BE69F writes 28,29,37,66-78 |
| missile hit funcs | 0x6FD2DA48 | PD 0x102F4210 writes 1,6,7,30,34,50,52,60-65 |
| missile dmg funcs | 0x6FD2DB68 | PD 0x102F2320 writes 15-19 |
| item events | 0x6FD277A8 | PD 0x102BEA10 (thorns 0x102AEE30, OW 0x102AF060, CB 0x102AF610, ...) |
| missile collision | 0x6FC5EDE0 | PD 0x102723E0 (all callers) |
| missile DS block | 0x6FC5A730 | PD 0x102EEBD0 -> 0x10270C50 |
| mastery lookup | 0x6FD9F870 | PD 0x10268A50 (code-init D2Common 0x6FDA0443/0x6FDA054A), missiles 0x10268A10 (0x6FDBBBCF/0x6FDBBBE2) |
| #10413 skill-missile test / item elemental | 0x6FDBB180 / 0x6FDBABE0 | PD 0x102EEBE0 (0x6FDBBD1B, 0x6FDBBF88) / 0x102EDD90 (0x6FDBC176, 0x6FDBC1A3) |
| length pre-step | 0x6FCFA7B0 | PD 0x102713C0 (via 0x6FCFC180 -> 0x102ED040) |
| poison apply | 0x6FCFCAB0 | PD 0x10271A70 (record {3, 0xDE2AF, rel}) |
| chill stat write | 0x6FCFC88E | PD 0x102C0AD0 |
| stun apply | 0x6FCFE288 | PD 0x102ED5A0 |
| state create (auras/curses) | 0x6FCC00B0 | PD 0x102BFE20 (curse_resistance, curse_effectiveness, max_curses) |
| -res on immunes | 0x6FC6E230 (/5) | PD 0x102C0540 (/2) |
| summon setter | 0x6FC6F970 | PD 0x102CA430 wrapper (bookkeeping) |
| Static Field post-hit | 0x6FCFCF20 at 0x6FC630BE | PD 0x102ED340 |
| execute clone | - | PD 0x10271E00 (used by PD's own 0x10271CF0 apply) |

### 7. PD2 do-functions by skill (damage-relevant; PD addresses)
8 multi-missile 0x102F9610 (Magic/Fire/Cold Arrow, Multiple Shot, Teeth, Bone Spear, Holy Bolt, Shock Wave, Ice
Barrage); 25 Enchant/Cold Enchant 0x102F98D0; 44 Blade Sentinel 0x102F9AC0; 45 sentries 0x102F9D10; 48 Blade Fury
0x102F9F80; 49 shadows/Decoy 0x102FA0E0; 51 Mind Blast 0x102FA4B0; 54 Blade Shield 0x102FA770; 55 Corpse Explosion /
mon death sentry 0x102FA7F0; 56/57 golems 0x102FAB50/0x102FAB90; 64 Sacrifice 0x102FABE0; 114 Raven 0x102FB340;
115 vines 0x102FB5B0; 118 Twister/Tornado 0x102FB6D0; 119 spirits/wolves/bear 0x102FB7F0; 153 Leap Attack 0x102FBC90;
154 Double Throw 0x102FBF30; 156 Gust 0x102FC320; 157 Blood Warp 0x102FC440; 158 Bash 0x102FC5B0; 159 Combustion
0x102FC6A0; 160 Static Field 0x102FC760; 161 Desecrate 0x102FC940; 162 Joust 0x102FCB80; 163 splash procs 0x102FCCE0;
164 Poison Strike 0x102FCEC0; 165 Hunger 0x102FD110; 166 Firestorm 0x102FD320; 170 charge finishers 0x102FD5B0;
174 Vengeance 0x102FDEB0; 175 Fire Claws 0x102FE410; 176 Holy Light 0x102FE610; 178 Dragon Flight 0x102FEAD0;
180 Charge 0x102FF1C0; 182 Inferno/Arctic Blast 0x102FF7F0; 188 Lightning Bolt/Fury 0x10300390; 189 Split Throw
0x10300400. Stock ones used by spells: 10 Guided Arrow/Bone Spirit, 17 Charged Bolt, 22 novas/War Cry, 23 Blaze,
24 Fire Wall, 26 Chain Lightning/Psychic Hammer, 28 Meteor/Blizzard/Fissure, 29 Thunder Storm, 30 curses, 43 Fire
Blast/Shock Web, 60/62 bone walls, 65/66/81 auras, 73 Blessed Hammer, 80 FoH, 121 Rabies, 123 Volcano, 124
Armageddon/Hurricane, 144 hydras.

## 8. PD2 vs stock (this part)
| topic | stock 1.13c | PD2 | status |
|---|---|---|---|
| magic mastery | none | 357 on skills and ApplyMastery missiles | VERIFIED |
| magic pierce, phys pierce | none | 358, 425 | VERIFIED (damage.md) |
| poison | one state per monster, replace if stronger | one per player / per player-owned minion | VERIFIED |
| half freeze duration | any value halves | 1 halves, >= 2 removes chill+freeze | VERIFIED |
| Static Field floor | 0/33/50% | 55/70/85% (data) | VERIFIED |
| Fire Ball hit | splash only | splash + direct hit on the struck unit | VERIFIED return / READ flow |
| Fire Bolt, Ice Bolt, Blizzard | single-target missiles | small splash (radius 3-4) | READ |
| Holy Bolt / FoH / Holy Nova | damage undead only | damage all monsters | READ |
| Blessed Hammer | +50% vs undead (dParam1) | none (data) | DATA |
| missile crit | own stat 141 x2 | owner crit/DS rules on phys | READ |
| curses | res >= 100 immune, one curse | 75 cap for players, curse_effectiveness, max_curses | READ |
| -res on immunes | /5 | /2 | VERIFIED (damage.md) |
| chill | speed only | also FCR and leap speed | READ |
| pets | - | masteries and synergy skill levels copied through data | DATA |
| Multiple Shot | no falloff | -20% per pierce (floor 20%) | VERIFIED |

## 9. Stat table (part C)
| stat | name | role |
|---|---|---|
| 12 | level | missile level, CE level scaling |
| 21/22 | mindamage/maxdamage | missile physical |
| 25 | damagepercent | missile phys % (only phys), auras (Might/Conc/Fana), pets |
| 48-59 | elemental min/max/len | missile elements |
| 74 | hpregen | poison/burn/OW drain, natural regen |
| 78/128 | attackertakes(light)damage | Thorns, Bone Wall, Iron Golem, Spirit of Barbs |
| 97/107/126/127/83/188 | skill bonuses | skill level |
| 109 | curse_resistance | curse length |
| 110 | item_poisonlengthresist | poison length (resist entry 7) |
| 118/153 | halffreezeduration / cannotbefrozen | length pre-step |
| 121/122 | demon/undead dmg% | missile phys vs type |
| 141/258/337/256/257 | DS / crit / multipliers | missile phys crit (owner) |
| 315/316/317 | firelength / burning min/max | burn state |
| 326 | poison_count | divides missile poison length |
| 327 | damage_framerate | = Missiles.DamageRate, DR/MDR scaling |
| 329-332, 357 | masteries | skill elemental x |
| 333-336, 358, 425 | pierce | B.pierce |
| 362-366 | item_elemskill_* | PD2 rows, no code reference found (OPEN) |
| 368 | max_curses | curse count |
| 443/481/463/461/462/464/475/476/459/509/192/193 | extra_* | missile/pet counts |
| 469 | pierce_count | Multiple Shot falloff |
| 485 | corpseexplosionradius | CE radius |
| 490 | eaglehorn_raven | Raven damage |
| 501 | deep_wounds | OW damage |
| 504 | curse_effectiveness | target-side curse value reduction |

## 10. Discrepancies with the JS models
- **`adv/engine/combat.js` (fixed):** poison was added into the per-hit damage and multiplied by attacks per second,
  i.e. stacked. PD keeps one poison per player, so the DPS now uses `after x min(hits/s, 25/len)` for poison
  (`attackOn`). Per-hit numbers are unchanged; all engine tests still pass.
- `adv/re/skilldmg.js`: numbers are per missile / per tick. Not modelled: the Fire Ball direct+splash double hit, PD2
  splash of Fire Bolt/Ice Bolt/Blizzard, Static Field (percent of life), CE, NextHit limits, missile crit. Its `server`
  rows are correct per missile. No code change (needs design).
- `adv/re/minions.js`: computes the copied mastery stats (passivestats) but not the pets' spell damage; notes it.
- `adv/engine/engine.js` differences listed in skilldmg.md (C precedence, no synergy gate) remain; the Fog precedence
  matters for CE radius, Static/Inferno -resist and Summon Grizzly count.

## 11. Open questions
1. Fire Ball double hit: confirm in game (or run the whole PD collision 0x102723E0 natively) that the struck monster takes
   splash + direct. The return value is VERIFIED; the collision handling of bit 2 is READ.
2. Curse struct field +0x18 scaled by curse_effectiveness: which stat each curse carries there (READ only).
3. CE level scaling direction and exact operand order (caster vs corpse level) - READ from stack offsets, not executed.
4. Psychic Hammer falloff counter (PD 0x102773E0) meaning (steps vs distance).
5. Whether PD's execute clone (0x10271E00, used by PD's own apply 0x10271CF0) differs from stock execute in any damage
   step (it calls the same helpers; not compared line by line).
6. Inferno/Arctic Blast debuff stacking: one state per caster or shared (state create PD 0x102BFE20 path not traced for
   these states).
7. Trap/sentry shot counts and timing (PD do 45) and Blade Sentinel hits are not traced.
