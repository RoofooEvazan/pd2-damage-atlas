# Item procs, skill cooldowns and charges (PD2 = 1.13c + ProjectDiablo.dll)

Status key: **VERIFIED** = the real code was run natively and matched a model (harness `harness/procs.c`, 11 modes ×
40,000 cases, plus a mutated-model pass that must fail in every mode; `harness/procs_rng.c` for the JS model).
**READ** = from the disassembly only. **DATA** = from the tables (PD2 `excel_live` / pd2data).

Strafe arrows and the "on attack / on striking per arrow" question are in `strafe_procs.md`; not redone here.

Files: this page, `adv/re/procs.js` (UMD model: rolls, stacking, cooldown values and slots, charges, replenish),
`harness/procs.c`, `harness/procs_rng.c`.

---

## 0. Summary

| trigger (stat) | event(s) | func | roll | skill level | cooldown checked? | notes |
|---|---|---|---|---|---|---|
| on attack (195) | 7 `domeleeattack` (8 `domissileattack` is never raised) | 20 | D2 LCG | item level | no | **melee swings that hit only** (§2.2) |
| on striking (198) | 5 `domeleedamage`, 6 `domissiledamage` | 20 | D2 LCG | item level | no | melee hits, and missiles that carry weapon damage |
| when struck (201) | 1 `damagedinmelee`, 2 `damagedbymissile` | 21 | D2 LCG | item level | **yes** | needs the "get-hit request" bit |
| on kill (196) | 9 `kill` | 30 | D2 LCG | item level | **yes** | any killing hit, including proc damage |
| on death (197) | 10 `killed` | 30 | D2 LCG | item level | **yes** | cast at the killer |
| on level-up (199) | 12 `levelup` | 30 | D2 LCG | item level | **yes** | cast at own position |
| on cast (200, PD2) | 14 `dospellcast`, PD2 timer | 33 (PD) | PD RNG | item level | no | 5 frames after the cast; SC-type skills only |
| on block (202, PD2) | 15 `doblock`, PD2 timer | 30 | D2 LCG | item level | **yes** | 2 frames after the block |
| on critical (203), on pierce (205) | 16 / 17 | 30 | – | – | – | **never raised**: dead stats |
| melee splash (359, PD2) | 5 | 20 | D2 LCG | – | no | casts skill 358 `proc_SplashDamage` |

- The roll is always `rand % 100 < chance`, where chance is the **summed** stat value of that (trigger, skill, level).
- Items with the same trigger, skill and level merge into one roll. Anything else is a separate roll. There is **no
  cap** on how many procs fire per hit.
- The proc skill is cast instantly by the item's wearer, at the item's skill level. +skills are not added.
  Synergies, masteries, pierce and +skill-damage stats all come from the wearer.
- Procs cost no mana, use no charges and never **start** a cooldown.
- Cooldowns (Skills.txt `delay`) are in frames. PD2 gives each class skill its own timer, and **one shared timer** to
  every other skill (other classes' skills, oskills, charges).
  - FCR and IAS do not change a cooldown.
  - There is no cooldown in town, except for Werewolf, Werebear and Vampire Form.
- Charges use the charge level only, never +skills. Synergies still use your hard points.
- PD2 adds charge regeneration (stat 422): one charge every `1 + 2500/s` frames, twice as fast in the PvP maps.

---

## 1. The proc stats (DATA)

ItemStatCost rows with an item event (`itemevent1/2`, `itemeventfunc1/2`) that cast a skill. The parameter is the
layer `skill << 6 | level` (DataTables +0xC6C = 6, +0xC70 = 0x3F). The value is the chance.

| stat | name | property | events → func | tooltip string |
|---|---|---|---|---|
| 195 | item_skillonattack | `att-skill` | domeleeattack, domissileattack → 20 | "%d%% Chance to cast level %d %s on attack" |
| 196 | item_skillonkill | `kill-skill` | kill → 30 | "... when you Kill an Enemy" |
| 197 | item_skillondeath | `death-skill` | killed → 30 | "... when you Die" |
| 198 | item_skillonhit | `hit-skill` | domeleedamage, domissiledamage → 20 | "... on striking" |
| 199 | item_skillonlevelup | `levelup-skill` | levelup → 30 | "... when you Level-Up" |
| 200 | item_skilloncast | `cast-skill` | dospellcast → 33 | "... on casting" (PD2) |
| 201 | item_skillongethit | `gethit-skill` | damagedinmelee, damagedbymissile → 21 | "... when struck" |
| 202 | item_skillonblock | `block-skill` | doblock → 30 | "... on block" (PD2) |
| 203 | item_skilloncrit | `crit-skill` | docrit → 30 | "... on critical hit" (no item uses it) |
| 205 | item_skillonpierce | `pierce-skill` | dopierce → 30 | "... on pierce" (no item uses it) |
| 359 | item_splashonhit | `splash` | domeleedamage → 20 | "Melee Attacks Deal Splash Damage" |
| 427 / 453 | map_mon_splash / map_mon_skillondeath | map mods | 20 / 30 | monster-side copies |
| 204 | item_charged_skill | `charged` (func 19) | – | "(%d/%d Charges)" |
| 422 | item_replenish_charges | `rep-charge` (func 17, value = param) | – | "Replenish 1 Charge in 3 Seconds" (Enigma), descstr2 "Charges per 100 seconds" |

Use in the item tables (UniqueItems, Runes, SetItems, Sets, MagicSuffix): on striking 149, when struck 70, on cast
62, on attack 18, level-up 9, death 6, kill 5, block 5, charged 239 (205 of them are magic staff suffixes), crit and pierce 0.

PD2 adds dedicated "Proc" skills: 442 AmpDmg Proc, 443 Weaken Proc, 444 Iron Maiden Proc, 445 Life Tap Proc,
446 Decrepify Proc and 447 LowRes Proc. They are used, for example, on 13 on-striking items and on the magic
suffixes "of Damage Amplification" and "of Lower Resist".

---

## 2. Dispatch

### 2.1 Registration and stacking — READ
- The stat-change callback D2Game `0x6FCF9470` registers event nodes when an item stat changes.
  - One node is registered per (stat, layer) and per event, on the unit list unit+0x90, via `0x6FCBE290` → `0x6FC57190`.
  - Node layout: +0 event, +0xC = `stat<<16 | layer`, +0x14 = function from the item-event table `0x6FD277A8`.
  - PD2 overwrites part of that table at runtime (`0x102BEA10`), but funcs **20, 21 and 30 stay stock**. Func 33 is PD2's.
  - PD2 patches:
    - The record size check (0x150).
    - The "does a node already exist" call `0x6FCF94F9` → PD `0x102C1120`, which special-cases splash (359) for players.
    - `0x6FCF9527` `jle` → `jl`, so that event 0 `hitbymissile` can be an `itemevent2` (thorns).
- The node function reads the **aggregated** stat value for its layer: `#10910(unit, stat, layer)`.
  - Two items with "10% chance to cast level 5 X on striking" therefore give one node at 20%, i.e. one roll.
  - Level 5 and level 7 of the same skill are two layers, so two nodes that roll independently. Both can fire on the same hit.
  - On-attack and on-striking entries of the same skill are different stats, so they are different nodes.
- Dispatcher D2Game `0x6FC57030` walks the whole list and calls every node whose event matches (VERIFIED end to end, mode 11).
  - The calls are made in registration order.
  - There is **no per-hit, per-frame or per-sequence limit**, so every matching node rolls once per event.

### 2.2 Where each event is raised — READ (+ VERIFIED where marked)
| event | raised at | condition |
|---|---|---|
| 5 domeleedamage / 1 damagedinmelee | execute D2Game `0x6FCFE0C0` (melee branch) | the hit bit is set, damage is not suppressed (+4 & 0x20), the target is alive and killable. Blocked, dodged, avoided or evaded hits have no hit bit |
| 6 domissiledamage / 2 damagedbymissile | same, missile branch (from PD `0x1026E7D0`) | same, **plus** the missile must carry damage flag 0x20. The missile gets 0x20 only when Missiles.txt `SrcDamage` (or the skill's `SrcDam` for missiles with a `Skill`) is non-zero (PD `0x1026B090` → #10484 → #10623). Result: arrows, bolts, javelins, throwing weapons and weapon-damage skill missiles carry it; spell missiles don't |
| 7 domeleeattack | melee impact `0x6FCFED30` at `0x6FCFEE4B` | every melee attempt that is still in range — but see below |
| 9 kill / 10 killed | end of execute `0x6FCFE35x` | the hit will kill (+4 & 2). **No** 0x20 requirement, so proc, thorns and pet kills count |
| 12 levelup | `0x6FCFDDD0`, `0x6FCFE743` | other = NULL |
| 14 dospellcast | PD timer (callback `0x102BECF0`) from the executor front PD `0x102C63C0` | §2.6 |
| 15 doblock | PD timer from post-hit PD `0x1026EDD0` | §2.7 |
| 8, 16, 17 | nowhere | on-crit and on-pierce stats are dead (`strafe_procs.md` §4) |

**On attack (195) only fires on melee swings that hit. VERIFIED (mode 11), which corrects `dmg_B_pipeline.md` §1a.7 / §4.9.**
- Func 20 refuses whenever it receives a damage struct without input flag 0x20 (`0x6FCCDE16`).
- At impact, the hit branch sets the struct's +0 to exactly 0x20 (`0x6FCFEDF0`) before the execute call. A missed or blocked swing keeps the queued struct.
- That queued struct is zeroed by every attack routine before the swing-start resolver, e.g. `0x6FC64BDE` and `0x6FC6C41D` (`rep stos`). Swing-start fills +0 only on hits (`0x6FCFDE0B`). So its +0 is 0 (READ).
- Mode 11 runs the real impact function, the real dispatcher and the real func 20.
  - With a hit node, the proc rolls.
  - With a miss or block node carrying +0 = 0, nothing rolls: the seed is untouched and no cast happens.
- Event 7 itself is raised for every attempt; thorns (event 3) still fire on misses.

### 2.3 The roll — VERIFIED (modes 1–4, 11; JS model vs native RNG, 20,000 seeds)
Stock funcs 20 / 21 / 30 use the wearer's seed (unit +0x20 lo / +0x24 hi):
```
chance = GetStat(wearer, stat, layer)            // summed over items
if chance <= 0: no roll
[func 20: if dmg && !(dmg.flags0 & 0x20): no roll]   [func 21: if !dmg || !(dmg.result & 0x04): no roll]
p      = lo * 0x6AC690C5 + hi                    // 64-bit; new lo = p mod 2^32, new hi = p >> 32
proc   = (new lo mod 100) < chance               // signed compare; chance >= 100 always procs
skill  = layer >> 6, level = layer & 63          // no record for skill -> nothing (after the roll)
```
PD func 33 (on cast) uses PD's generator `0x102C5D10` instead, on the same seed words:
```
p = lo * 0x6AC690C5 (64-bit); new lo = (p mod 2^32) + hi
new hi = (p >> 20) mod 2^32 + (hi != 0 && (p mod 2^32) > 0x7FFFFFFF - hi ? 1 : 0)
proc = (new lo mod 100) < chance                 // unsigned compare, chance > 0
```
- The probability of one node is `min(chance, 100)%`; the modulo bias is below 10⁻⁷.
- The rolls consume the wearer's seed, so other procs, crushing blow, etc. change the sequence but not the odds.

### 2.4 Casting the proc — VERIFIED (arguments), READ (what the executor does with them)
| func | with a target | without a target |
|---|---|---|
| 20 (195, 198, 359) | `0x6FD117E0`(caster = wearer, target = other unit, skill, level, **flag 1**) | `0x6FD11730`(wearer, its path x/y, flag 1) |
| 20, skill has `ItemTgtDo` (Teleport, Blink, LucionBlink, 3 monster novas) | `0x6FD117E0`(caster = **struck unit**, target = itself, flag 0) | nothing is cast, but the function still returns 1 |
| 21 (201) | `0x6FD117E0`(caster = wearer, target = attacker, **flag 0**) | `0x6FD11730`(wearer, own x/y, flag 0) |
| 30 (196, 197, 199, 202) | `0x6FD117E0`(caster = wearer, target = other, **flag 0**) | `0x6FD11730`(wearer, own x/y, flag 0) |
| 33 (200) | if other ≠ NULL and Skills `ItemTarget` ≠ 5: `0x6FD117E0` via PD `0x102EFCB0` (flag 1) | at the wearer's **path target** x/y (where you cast), via `0x6FD114F0` with flag 1. Refused while the wearer has state 54 `uninterruptable` |

- "other" is the struck unit (198), the attacker (201), the killed unit (196), the killer (197), or the blocked attacker (202).
  - For a blocked **missile**, the blocking unit itself is used: the PD timer replaces a missile with the unit.
  - Level-up passes NULL.
  - For on-cast (200), "other" is your path's target unit (#10392).
- `0x6FD114F0` then applies Skills `ItemTarget` (READ, jump table `0x6FD11718`):
  - 1 = cast on yourself.
  - 2 = random location around the caster (Teleport).
  - 3 = nearest corpse.
  - 4 = pick a target unit (Bone Spirit, Fist of the Heavens, Psychic Hammer, Holy Light).
  - 0 / 5 = the given unit, or the given x/y.
- The executor runs **immediately**, in the same frame as the triggering hit, through PD `0x102ED4E0` → `0x102C63C0` → stock `0x6FCC19A0`.
  - Arguments: (game, skill, level, 0, 1, flag).
  - With arg4 = 0 there is no mana or charge use and no cooldown start (`0x6FCC1C4B`).
  - With arg5 = 1 the caster does not need to own the skill.
  - There is no cast animation.
- Stock refused procs while the caster had state 54 `uninterruptable`. PD2 NOPs that check in the target version (`0x6FD1181D` `test` → `xor`), but the x/y version `0x6FD11765` still has it.

### 2.5 Flag 0 = cooldown check — READ, check VERIFIED (mode 8)
PD `0x102C63C0` runs before every skill execution. For a **player** caster with **flag 0**:
- If the skill is on cooldown (§4.3 slots), the proc does nothing.
- Otherwise, if the skill is an "SC-type" skill (§2.6), a dospellcast event is scheduled. So a flag-0 proc can itself trigger on-cast procs.
- Flag 1 procs (on attack, on striking, splash, on cast) skip both steps.

Effects:
- **When struck, on kill, on death, on level-up and on block procs of a skill that is on cooldown fail silently.**
  - Example (Mirror Shield, "6% chance to cast level 14 Cloak of Shadows on block"): on an Assassin, the proc does nothing for 125 frames after she casts Cloak of Shadows herself.
  - On other classes, Cloak of Shadows uses the shared slot. The proc then fails while *any* non-class cooldown is running, e.g. right after using Bone Prison charges.
- Procs never start a cooldown, so a proc Cloak does not block the Assassin's own Cloak.

### 2.6 On cast (200), PD2 — READ; roll and cast VERIFIED (mode 4)
The executor front PD `0x102C63C0` (player, flag 0, not on cooldown) schedules timer 7 at `frame + 5` with callback `0x102BECF0`, event 14, other = #10392(path target), when all of these hold:
- the skill is **not** in PD set A {277 Blade Shield, 57 Thunder Storm, 217/218/219/220/513/514 scrolls and books of Identify and Town Portal},
- **and** one of:
  - it is in set B {242 Hunger, 251 Fire Trauma, 256 Shock Field, 261 Charged Bolt Sentry, 262 Wake of Fire Sentry, 272 Inferno Sentry, 276 Death Sentry, 313 "Sentry Chain Lightning", 392 "Sentry Lightning"},
  - or its Skills.txt `anim` is SC,
  - or its `anim` is SQ with `seqnum` 6, 12 or 18 (Inferno, Chain Lightning, Frozen Orb, Arctic Blast),
- and the player is not in town (levels 1, 40, 75, 103, 109; `0x102CEA40`) and not inside the fixed rectangles of PvP maps 157/159 (`0x102CF710`).

Consequences:
- One roll per node per **cast**, 5 frames (0.2 s) after the executor runs. Faster cast rate means more casts per second, so more on-cast procs.
- Proc casts with flag 1 never schedule it, so on-cast cannot chain into itself.
  - Flag-0 procs (when struck, kill, death, level-up, block) of SC skills **do** schedule it.
  - Example: Rainbow Facet's level-up Nova → rolls your on-cast items.
- Mercenaries (type 1) schedule it for skills cast in mode SC (7) or 14, unless the skill is in set A, and for any skill in set B.

### 2.7 On block (202), PD2 — READ
- PD post-hit `0x1026EDD0` runs after every melee impact and every missile collision, for any defender unit.
- When result flags & 0x8010 (shield block 0x10 or Weapon Block 0x8000), and the defender is not in town or a PvP-map safe rectangle, it schedules timer 7 at `frame + 2` → event 15 → func 30 (flag 0, cooldown checked).
- One roll per node per block.

### 2.8 Which hits trigger what — READ
| attack | 198 on striking | 195 on attack | 201 when struck (on you) |
|---|---|---|---|
| melee swing that hits | yes (event 5) | yes (event 7) | yes, if the hit requests hit recovery (flag 0x04). PD resolver `0x10270FD0` sets it on every hit unless the defender has state 54 |
| melee miss / block / dodge / avoid / evade | no | no (§2.2) | no (block → on-block instead) |
| bow, crossbow, throw (any missile with SrcDamage) | yes, per missile per target struck (each pierce again) | never (event 8 is not raised) | yes, if Missiles.txt `GetHit` = 1 |
| spell missile (SrcDamage 0), e.g. your Fire Ball, a Fallen Shaman bolt | no | no | **no**: event 2 needs damage flag 0x20 even for incoming missiles (394 GetHit missiles have no SrcDamage) |
| PD melee splash / area hits (`0x10271CB9` → filtered dispatcher PD `0x102B0160`) | no (funcs 20/21/30 skipped) | no | no |
| thorns, Iron Maiden, poison ticks, auras | no (no 0x20) | no | no |
| any killing blow (including procs, thorns, splash) | – | – | on-kill yes (event 9 has no 0x20 gate) |

**Multi-hit skills.**
- Every impact event is a separate hit. Zeal, Fury and Dragon Claw sequence hits, Frenzy and Double Swing swings, kicks, and Whirlwind targets each roll every 198 and 195 node once per connecting hit.
- For Strafe, Multiple Shot and other multi-missile skills, each missile rolls 198 per target it hits (`strafe_procs.md`).

### 2.9 Can a proc trigger another proc? — READ
| chain | possible? | why |
|---|---|---|
| proc damage → on striking / on attack | only for weapon-type proc skills | the proc's missiles carry 0x20 only with SrcDamage. Spells (Frost Nova, Nova, Chain Lightning, …) don't. Multiple Shot (Demon Machine), Lightning Bolt, Lightning Fury and Poison Javelin (Crackleshot) do (SrcDam 128), so their hits can roll 198 again. The chain is limited only by the odds |
| proc kills → on kill | yes | event 9 has no gate |
| flag-0 procs (201, 196, 197, 199, 202) → on cast | yes, for SC-type skills | §2.6 |
| on cast / on striking / on attack → on cast | no | flag 1 |
| proc → proc of the same node in the same event | no | each node is called once per event |

### 2.10 Who owns the proc — READ (via `dmg_C_spells.md`)
- The caster is the item wearer, except for the `ItemTgtDo` skills in func 20, which the struck unit casts.
- The skill runs at the item level. `lvl` in all calcs is that level.
- **Synergies** read `skill('X'.blvl)` through D2Common `0x6FD9E480`, which returns the caster's own entry (see §5.3). So a Sorceress with 20 hard points in Ice Bolt gets the full Frost Nova synergy on a proc Frost Nova; a Barbarian gets none.
- **Masteries** (329–332, 357), **pierce** (333–336, 358, 425) and +% skill-damage stats are the caster's: missiles are owned by the caster (`dmg_C_spells.md` §1).
- Proc missiles carry no damage flag 0x20, so their hits raise no on-striking or when-struck events (§2.9). Kill credit was not traced.

### 2.11 PD2 limits and rate caps — READ
- On striking, on attack and on cast (flag 1) have no internal cooldown and no per-hit or per-frame cap.
- When struck, kill, death, level-up and block (flag 0) are blocked while that skill's cooldown slot is busy (§2.5).
- On cast is delayed by 5 frames and on block by 2. Neither fires in town or in the PvP-map safe rectangles.
- There is no "once per frame" guard for item procs. The 1-frame `justhit` throttle in `dmg_D_incoming.md` is for monster modifiers only.

---

## 3. Worked examples

1. **Death (runeword): "25% chance to cast level 18 Glacial Spike on attack"** (DATA)
   - Hit roll 80%: P(proc per swing) = 0.80 × 0.25 = **0.20**.
   - The stock-style reading "on every attempt" would give 0.25.
   - With a bow the proc never fires.
2. **Two "of Damage Amplification" (on cast, AmpDmg Proc)** (VERIFIED `combineProcs`)
   - Two items at 8%, level 15 each: **one node at 16%**.
   - One at 8% level 15 and one at 10% level 23: two nodes, P(at least one) = 1 − 0.92·0.90 = **17.2%**. Both can fire on one cast.
3. **Enigma: charged skill + "rep-charge 33"** (VERIFIED `replenishFrames`)
   - The first charged skill of the item gains 1 charge every 1 + ⌊2500/33⌋ = **76 frames (3.04 s)**.
   - In PvP levels 157/159/166: 1 + ⌊75/2⌋ = **38 frames**.
   - Gavel of Pain (16): 157 frames = 6.28 s. Bloodraven's Charge (12): 209 frames = 8.36 s.
4. **Joust level 20, Alma Negra (joustreduction 13)**: `max(104 − ⌊100/2⌋, 38) − 13` = 54 − 13 = **41 frames = 1.64 s** (VERIFIED `cooldownFrames`).
   - With Leoric's Mithril Blade (19) as well, from level 27 on it is 38 − 32 = **6 frames**.
5. **Gust level 20, Quetzalcoatl (gustreduction 50)**: `max(163 − (100 + 50), 13)` = **13 frames = 0.52 s** (the floor). Without the helm it is 63 frames = 2.52 s.

---

## 4. Skill cooldowns

### 4.1 Setting a cooldown — READ, set/clear VERIFIED (mode 9)
- The executor `0x6FCC19A0` evaluates Skills.txt `delay` (calc at record +0x190) with `(unit, skill, level)` after a successful player cast.
  - The call is `#10786` at `0x6FCC1CB6`.
  - This happens only when arg4 = 1 (normal casts) and the caster is a **player** (`[unit] == 0`). SQ skills are skipped while unit+0x38 has high bits set.
  - `level` is the level of that cast: hard + soft points, or the charge level.
- If the value is > 0, PD `0x102C99E0`(delay, unit, game, skill record) runs. It replaces the stock `0x6FCBFF70`, which put state 121 `skilldelay` on the player: one shared delay for everything.
  - All three call sites are patched: `0x6FCC1CD9`, `0x6FCC1CF1` → `0x102EEC00`, and `0x6FCC09E3` → `0x102EEC10`.
  - The last one is in the old srvdofunc 116, which PD replaces with `0x102F76B0`, so it is dead.
```
if delay <= 0 or skill id == 0 or no client: nothing
if skill not in {223 Werewolf, 228 Werebear, 388 Vampire Form} and the player is in a town room (#10331/#10057): nothing
slot = index of skill in PD's list for the player's class, if found and Skills.charclass == player class; else -1
slot >= 0: playerdata+0x1D0[slot] = 1      else: playerdata+0x1CC = 1   (shared slot)
send packet 0x84 {u16 slot, u8 1} to the client
timer type 0xC at game frame + delay  ->  0x102C9960: clear the same slot, send 0x84 {slot, 0}
```
- The delay is plain game frames from the executor call.
  - Normal casts reach the executor from the player "skill do" timer handler `0x6FCC1E80` (timer type 8), i.e. on the cast's event frame.
  - **FCR and IAS do not change the delay.** They only change how soon after cast start that frame comes and how long the animation is.
  - No stat is read, except the delay calc's own `stat(...)` terms.

### 4.2 Checking — READ, VERIFIED (mode 8)
- D2Common #10787 `0x6FDA2340` is the "can this skill be used" test. It is used by 15 D2Game sites (skill start) and 6 D2Client sites (the client's own check before sending a cast).
  - Its stock state-121 test `0x6FDA2210` is replaced by PD `0x10268B80` at `0x6FDA2462`.
  - It returns error 8 when the skill is not ready.
```
usable = true                                           if the unit is not a player with playerdata
usable = true                                           if in a town room and skill not in {223,228,388}
slot as above:  usable = (slot >= 0 ? playerdata+0x1D0[slot] : playerdata+0x1CC) == 0
```
- The server also re-checks inside the executor front for flag-0 casts (§2.5).
- The client keeps its own copy of the slots, set by packet 0x84 through PD `0x102DF050` (slot 0xFFFF = shared). Client and server agree without the client running any timer.

### 4.3 Slots — shared cooldowns (READ, slot map VERIFIED through mode 8/9 with the extracted lists)
- PD's static init `0x10137B80` builds map `0x104E3584`: class → vector of skill ids.
  - The vectors are the 30 tree skills plus PD2's extra class skills (Sorceress +369 Ice Barrage, 376 Combustion, 383 Lesser Hydra; Necromancer +367 Blood Warp, 381 Dark Pact; Paladin +364 Holy Nova, 371 Holy Light, 378 Joust; Barbarian without 127/129/136, +368, 375; Druid +370 Gust; Assassin +366, 380).
  - The vector index is the slot.
- **Each of your own class skills has an independent cooldown.** Meteor and Blizzard don't block each other.
- **Every other skill shares one slot** (+0x1CC): other classes' skills from charges or oskills, and non-class skills such as Vampire Form (388).
  - Example: a Barbarian who uses Bone Prison charges (125 frames) cannot, for 5 s, cast Meteor charges, use any other non-class delayed skill, or get any flag-0 proc of a non-class skill.
- A charged skill of your **own** class uses the same slot as your hard-point version.

### 4.4 Cooldown reduction — DATA
The only reductions are stats read inside the delay calcs themselves:

| stat | read by | item | value | effect |
|---|---|---|---|---|
| 460 gustreduction | Gust | Quetzalcoatl | 50 | −50 frames (2 s), floor 13 |
| 465 joustreduction | Joust | Alma Negra | 13 | −13 frames (0.5 s) |
| 271 joustreduction_leorics | Joust | Leoric's Mithril Blade | 19 | −19 frames (0.75 s) |
| 209 joustreduction_zeraes | Joust | Zeraes Resolve (Amazon-only pike) | 38 | −38 frames (1.5 s) |
| 483 dragonflightreduction | **nothing** | none | – | "Dragon Flight Cooldown Reduced By 0.5 Seconds" exists as a stat and string, but the Dragon Flight calc `max(51 - lvl, 25)` does not read it and no PD code pushes 483 |

- The Joust reductions are subtracted **after** the 38-frame floor. A result ≤ 0 means no cooldown at all (`0x102C99E0` ignores it).
- No FCR-, IAS- or generic "cooldown reduction" stat exists.

### 4.5 PvP — DATA/READ
- Stats 482 `pvp_cd` and 492 `pvp_lld_cd` are given to every unit in PvP levels 157/159/166 (25 each; 482 in NM/Hell, 492 in Normal; `dmg_D_incoming.md` §5.2). They are **not** cooldowns.
- The only readers are two Skills.txt calcs:
  - Battle Cry length `ln12 + pvp_cd*5 + pvp_lld_cd*5` (+125 frames).
  - Teleport's `teleportdebuff` state length `max(0, 25 − pvp_lld_cd)`, which is 0 in Normal-difficulty PvP.
- `teleportdebuff` (25 frames, "base dmg reduction" par2 = 50) is a damage debuff after Teleport, not a cooldown.
- No PvP-specific cooldown code exists. PvP maps halve the charge-regeneration wait (§5.4).

### 4.6 Every skill with a cooldown (DATA; slot from §4.3; seconds = frames / 25)
| id | skill | class | anim | `delay` | seconds | slot |
|---|---|---|---|---|---|---|
| 25 | Plague Javelin | ama | TH | 25 | 1.00 | 19 |
| 32 | Valkyrie | ama | SC | 25 | 1.00 | 26 |
| 46 | Blaze | sor | SC | 50 | 2.00 | 10 |
| 51 | Fire Wall | sor | SC | 38 | 1.52 | 15 |
| 56 | Meteor | sor | SC | 13 | 0.52 | 20 |
| 59 | Blizzard | sor | SC | 23 | 0.92 | 23 |
| 376 | Combustion | sor | SC | 63 | 2.52 | 31 |
| 68 | Bone Armor | nec | SC | 50 | 2.00 | 2 |
| 75 | Clay Golem | nec | SC | 25 | 1.00 | 9 |
| 78 | Bone Wall | nec | SC | 25 | 1.00 | 12 |
| 85 | Blood Golem | nec | SC | 25 | 1.00 | 19 |
| 88 | Bone Prison | nec | SC | 125 | 5.00 | 22 |
| 90 | Iron Golem | nec | SC | 25 | 1.00 | 24 |
| 94 | Fire Golem | nec | SC | 25 | 1.00 | 28 |
| 367 | Blood Warp | nec | SC | `max(150 - lvl*5, 0)` | lvl 1: 5.80, lvl 20: 2.00, lvl ≥ 30: none | 30 |
| 364 | Holy Nova | pal | SC | 100 | 4.00 | 30 |
| 378 | Joust | pal | A1 | `max(104 - (lvl*5)/2, 38) - (465 + 271 + 209)` | lvl 1: 4.08, lvl 20: 2.16, lvl ≥ 27: 1.52 | 32 |
| 222 | Plague Poppy | dru | SC | 25 | 1.00 | 1 |
| 223 | Werewolf | dru | SC | 18 | 0.72 (also in town) | 2 |
| 228 | Werebear | dru | SC | 18 | 0.72 (also in town) | 7 |
| 231 | Cycle of Life | dru | SC | 25 | 1.00 | 10 |
| 234 | Eruption | dru | SC | 13 | 0.52 | 13 |
| 235 | Cyclone Armor | dru | SC | 50 | 2.00 | 14 |
| 241 | Vines | dru | SC | 25 | 1.00 | 20 |
| 244 | Volcano | dru | SC | 13 | 0.52 | 23 |
| 247 | Summon Grizzly | dru | SC | 50 | 2.00 | 26 |
| 370 | Gust | dru | SC | `max(163 - (lvl*5 + 460), 13)` | lvl 1: 6.32, lvl 20: 2.52, lvl ≥ 30: 0.52 | 30 |
| 256 | Shock Field | ass | S2 | 25 | 1.00 | 5 |
| 264 | Cloak of Shadows | ass | SC | 125 | 5.00 | 13 |
| 268 | Shadow Warrior | ass | SC | 50 | 2.00 | 17 |
| 275 | Dragon Flight | ass | KK | `max(51 - lvl, 25)` | lvl 1: 2.00, lvl 20: 1.24, lvl ≥ 26: 1.00 | 24 |
| 279 | Shadow Master | ass | SC | 50 | 2.00 | 28 |
| 388 | Vampire Form | – | SC | 18 | 0.72 (also in town) | shared |

- Monster, boss and mercenary rows also have a `delay`: 390, 403, 421 MonHydra 40, 449, 459 Merc Static Field 50, 461 A3 Merc Meteor 16, 463 A3 Merc Blizzard 25, 469, 524, 525, 527, 570, 577, 594, 597.
- The executor applies `delay` **only to players**, so these rows have no effect through this path. Whether monster AI enforces them elsewhere was not traced; the ItemStatCost rows 446–448 `mon_cooldown1..3` exist, but their use was not traced.

---

## 5. Charges (stat 204)

### 5.1 Storage — READ, consumption VERIFIED (mode 10)
- Layer = `skill << 6 | level`; value = `max_charges << 8 | current_charges` (both 8 bits).
  - The property `charged` is func 19 (min = charges, max = level). The negative min/max on magic staff suffixes ("of Teleportation" −30/−6) are the stock "scale with item level" form. The scaling rule was not traced.
- When the item is equipped, D2Common `0x6FD9EE00` (body at `0x6FD9EF2A`) gives the unit a separate skill entry for each charged skill:
  - +0x34 = item GUID,
  - +0x28 = the charge level,
  - +0x38 = charges,
  - **+0x3C = 1**, the "has charges" flag also set by #10863 `0x6FD9E1F0`.
- Using charges: the executor sees the selected skill entry is item-owned (#10304 GUID ≠ −1), so `0x6FCBF850` → `0x6FCBF4A0` spends a charge instead of mana:
```
v = GetStat(item, 204, skill<<6 | #10306(unit, entry, bonus=0))
m = v >> 8; if cur <= 0 or v == 0 or m <= 0 or m > 255: fail
new = (m << 8) | clamp(cur - 1, 0, m)                 // #11037 SetStat, then item update to the client
```

### 5.2 Cast level: no +skills — VERIFIED (mode 5)
- The executor's level for the selected entry is #10306 `0x6FDA05C0`(unit, entry, bonus = 1):
```
lvl = entry+0x28;  if bonus and entry+0x34 == -1: lvl += +skills (0x6FD9FCB0)
return clamp(lvl, 0, MaxLvl)
```
- An item-owned entry never gets +skills, all-skills, class skills or tab skills. Charges always cast at the printed level.
- The same level drives the cooldown calc (Dragon Flight, Blood Warp, … from charges use the charge level).

### 5.3 Synergies from your hard points — VERIFIED (mode 6)
- Every `skill('X'.blvl)` and `skill('X'.lvl)` calc and #10630 GetSkillFromId use `0x6FD9E480`:
```
among the unit's entries with skill id X and +0x3C == 0 (charged entries are skipped):
    prefer an entry with owner == -1 (your own hard/soft-point entry); otherwise the highest +0x28
```
- Charges therefore **use your hard points for synergies**.
  - Example: Demonlimb's level 23 Enchant charges on a Sorceress with 20 points in Warmth get the `Warmth.blvl × par8` synergy.
- Charges **never count as synergy points** themselves, since charged entries are skipped.
- Masteries and pierce are the caster's, as for procs.

### 5.4 Recharge (PD2 stat 422) — VERIFIED (mode 7)
- Stock D2 charges only come back through NPC repair. That path was not traced here.
- PD2's item timer 3:
  - It is started when the item gets stat 422 (`0x102AE080`).
  - It re-arms itself in PD `0x102AE170` (via `0x102AE2F0`, timer table `0x103C39E8`).
```
entry = the FIRST stat-204 entry of the item's base stat list (0x10277EF0(item, 0x40))    // one skill only
if (value / 256) > (value % 256): value += 1 ; update the client       // ItemStatCost[204].Multiply = 256
s = stat 422 on the item;  if s == 0: stop
t = trunc(2500 / s)                                                     // 64-bit CRT divide 0x10302E20
if the owner is a player standing in level 157, 159 or 166: t >>= 1
next tick at game frame + 1 + t
```
- s is "charges per 100 seconds". The period is `(1 + ⌊2500/s⌋)` frames:

| item | s | period |
|---|---|---|
| Enigma, Stone, Oath, Naj's Puzzler, Demonlimb | 33 | 3.04 s |
| Gavel of Pain | 16 | 6.28 s |
| Bloodraven's Charge | 12 | 8.36 s |

- The replenish function does not check whether the item is equipped. Whether item timers tick for stash or inventory items was not traced.

### 5.5 PD2 changes to charges
Replenishing (422), twice as fast in PvP maps. Per-skill cooldown slots also apply to charges (§4.3). The cast level
rule, +skills rule and synergy rule are stock (no PD patch in `0x6FDA05C0`, `0x6FD9E480`, `0x6FCBF4A0` or the executor).

---

## 6. What was not verified
- **The miss path's +0 = 0.** The zeroed struct was READ in the attack routines. Mode 11 verifies the consequence given that value.
- **Event raise sites.**
  - The Missiles `GetHit` / SrcDamage conditions.
  - The town and PvP-rectangle exclusions (on-cast and on-block scheduling in `0x102C63C0` and `0x1026EDD0`).
  - PD sets A and B: decoded from the static constants; the std::set code was not run.
- **Cast targeting** by `ItemTarget` inside `0x6FD114F0`, and the executor itself.
- **On-death casts while dead**, and the monster-side proc path.
- **Synergy and mastery attribution for proc and charge damage.** It follows from `dmg_C_spells.md` plus modes 5/6; no damage number was run.
- **Charges from repair**, the func-19 negative charge/level rule, and where monsters enforce their `delay`.
- **The class-skill index 0x102D18C0.** Replaced in the harness by the vectors decoded from `0x10137B80`; the vectors themselves are READ.

## Address index
- Events: dispatcher `0x6FC57030`; table `0x6FD277A8` (PD `0x102BEA10`); func 20 `0x6FCCDDC0`, 21 `0x6FCCDCE0`, 30 `0x6FCCDBF0`, 33 PD `0x102AFFF0`; registration `0x6FCF9470` (PD `0x102C1120`).
- Hit, impact and execute: melee impact `0x6FCFED30`; execute `0x6FCFE0C0`; missile PD `0x1026E7D0`; missile flag PD `0x1026B090`.
- Procs, cast and cooldown timers: post-hit and on-block PD `0x1026EDD0`; timer callback PD `0x102BECF0`.
- Cast wrappers `0x6FD117E0` / `0x6FD11730` / `0x6FD114F0`; executor front PD `0x102C63C0` (via `0x102ED4E0`); executor `0x6FCC19A0`; do timer `0x6FCC1E80`.
- Cooldowns: set PD `0x102C99E0`, clear `0x102C9960`, check `0x10268B80` (in #10787 `0x6FDA2340`), client packet 0x84 PD `0x102DF050`, class map `0x104E3584` (init `0x10137B80`), index `0x102D18C0`; PD sets A `0x104E3118` / B `0x104E30AC` (init `0x10126C20` / `0x10126BB0`).
- Charges: use `0x6FCBF850` → `0x6FCBF4A0`; entry creation D2Common `0x6FD9EE00`, charges #10863 `0x6FD9E1F0`; level #10306 `0x6FDA05C0`; lookup `0x6FD9E480` (#10630 `0x6FD9F070`); replenish PD `0x102AE170` / `0x102AE080`.
