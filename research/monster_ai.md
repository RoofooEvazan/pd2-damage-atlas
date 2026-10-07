# Monster AI in PD2: targeting, decisions, PD2 changes

PD2 = Diablo II 1.13c `D2Game.dll` + `ProjectDiablo.dll` (PD). This covers how a monster picks its target, how it
chooses between attacks and skills, what PD2 changes, and which champion/unique mods touch the AI.

**Status labels.**
- **VERIFIED** = the real code was run natively in the harness and matched a model.
- **READ** = read from the disassembly only.
- **DATA** = taken from the PD2 tables (`data.zip` MonStats/MonAi/Skills/States/Levels).

**Files.**
- `adv/re/monster_ai.js` is the model (UMD, `PD2MonsterAI`).
- `harness/aipick/` holds the native check:
  - `aipick.c` and `aipick.py` run the check: `cd harness/aipick && python3 aipick.py [mutation]`.
  - `check_js.js` replays the native results through `monster_ai.js`.

**Units.**
- Distances are path sub-tiles, which this page calls "units". Skill "yards" are 2/3 of a unit.
- Times are game frames at 25 frames per second.
- `aipN` always means the MonStats column for the current difficulty: `aipN`, `aipN(N)` or `aipN(H)`. It is a signed
  16-bit value at MonStats `+0x56 + 6·(N−1) + 2·difficulty`. The column offsets were checked against the D2Common
  column table at 0x6FDAB76C.

---

## 1. The AI engine: when a monster thinks

### 1.1 The AI table (READ)
- **Where it is.** D2Game **0x6FD2E788** holds one 16-byte record per MonAi.txt row, 148 rows, indexed by MonStats `AI`
  (MonStats +0x1E). Each record is `{targetMode, initFn, thinkFn, extraFn}`.
- **Getter.** The getter is 0x6FCBD0F0.
- **Special states.** A second block at `0x6FD2E668 + 16·s` (s = 1…17) holds the *special AI states*. These are
  temporary AIs that override the normal one.
  - The three curse states 10, 11 and 12 are used only when the monster qualifies (§2.7). Otherwise the normal record
    is used.
- **Switching.** Switching to a special state is done by 0x6FCBD230. It runs the new state's init function and stores
  the think function in aiControl+4. aiControl is MonsterData+0x28.

| targetMode | search run before `think` (0x6FCBD160) | AIs |
|---|---|---|
| 0 | none; `think` finds its own targets | NPCs, Idle, Hireable, NecroPet, all assassin/druid summons, BoneWall, BaalCrab… (26) |
| 1 | melee finder via 0x6FD13220. **No target → think is skipped** and the monster idles (§2.6) | 104 AIs: almost every monster, including all act bosses and ubers |
| 2 | melee finder 0x6FCD1C90; think runs even with no target | Tentacle(Head), Hydra, traps, ShadowWarrior/Master, Raven, Vines, NpcBarb, Nihlathak, **Ancient**, BaalThrone (16) |
| 4 / 5 | finder via 0x6FD13150; none → wander or idle 20 | SandMaggot (4, think skipped) / FrogDemon (5, think runs) |

### 1.2 The tick (VERIFIED for the rescheduler, READ for the rest)
Monster AI runs as a timed event. Event type 2 is handled by **0x6FCBD3B0** (event table 0x6FD1BC14[2]).

**Rescheduling after a mode ends.** When a monster finishes a mode (attack, cast, walk step, hit recovery…),
0x6FC94E60 schedules the next think:

```
if an AI event (type 2) is already pending at a later frame: do nothing
if state 21 (stunned):            next think = now + 45
else  d = aidel[col]              col = difficulty if (game+0x6A != 0 or game+0x74 != 0) else Normal
      if d <= 0: d = 15
      next think = now + d
```
- **VERIFIED** on 10,000 native cases of 0x6FC94E60: 0 mismatches. The mutation "always use the difficulty column"
  gives 1,134 mismatches.
- The game+0x6A/+0x74 test is the same game-type/ladder test that picks the MonLvl "L-" columns (FINDINGS.md).
  - In a normal online game the difficulty column is used.
  - A game where both flags are 0 uses the Normal `aidel`.
- DATA: in Hell, 746 enabled monsters use `aidel(H)` = 13, 272 use 15, and 48 use 12.
  - A Hell monster therefore re-decides about **0.5 s after each action ends**.
  - In Normal most monsters use 15 (0.6 s).

**Stall / idle.** 0x6FD122C0 `idle(n)` handles stalls:
- It puts the monster in mode NU (neutral).
- It deletes pending AI events.
- It schedules the next think at `now + n`.
- n = 0 becomes 1. A negative n (bad data) fires on the next frame.

The stall values in MonAi ("stall time", "melee stall"…) are frames passed to this function.

**Order inside one think event (0x6FCBD3B0):**
1. **Stunned** (state 21): idle 3 frames, then stop (0x6FCBCD90).
2. A room check (0x6FCBCCF0). It searches type 8 within 9 units when the room's flag 0x800 is set. It was not analysed.
3. **Command / owner check** (stock 0x6FCBCB30). **PD replaces this call** at D2Game +0x9CDC6 with PD 0x102BA0C0
   (the pet leash, §3.3).
4. **Target search** by targetMode (0x6FCBD160). **PD wraps this call** at D2Game +0x9D495 with PD 0x102B9760
   (the arena freeze, §3.3).
5. **Checks that need a target** (0x6FCBCFE0):
   - the boss "notice" pause (§4.4)
   - the teleport umod (§4.1)
   - the Levels.txt `MonSpcWalk` retarget. In levels with `MonSpcWalk` > 0 (jails, basements, jungle, lava), a target
     further away than MonSpcWalk that cannot be reached is swapped for a reachable one, or the monster does a
     special walk.
6. `think(game, unit, tick)`.
   - The tick record is: `+0 aiControl, +8 target, +0x14 distance, +0x18 inMelee, +0x1C MonStats, +0x20 MonStats2`.

---

## 2. How a monster picks its target

### 2.1 The target lists (READ)
The game keeps ten linked lists at `game+0x10F8`. A unit's list number is stored in unit+0xD0 (0xB = none).

| slots | contents | added by |
|---|---|---|
| 0–7 | one list per player: **the player first, then all his pets** (summons, mercenary, revives, Decoy, Valkyrie, sentries…) | 0x6FD0FED0 (player), 0x6FD0FE40 (pet; PD's summon code calls it too) |
| 8 | allied NPC monsters (Align 1: A5 barbarians and similar) | spawner 0x6FC67450, Wussie AI 0x6FCC7A10 |
| 9 | **lures**: an **Attract** victim, a **Confused** monster, **Bone Wall** and **Bone Prison** segments | do 59 0x6FC71D42, Confuse 0x6FC70D32, do 60 0x6FC711CC, bonewallmaker 0x6FC610DD, Bone Prison 0x6FC70FC7 |

### 2.2 The melee finder, D2Game 0x6FCD1C90 (READ; distance VERIFIED)
This finder is used by every targetMode 1/2/4/5 AI, which means almost every monster.

```
if forcedTarget(monster) valid: return it                   (§2.7: Taunt, Attract, Confuse)
if monster alignment != hostile: search type 5 (allied / converted monsters), range 35
needLOS = !(aiControl.flags & 8)                             # true until the monster has acquired someone once
          (flag 0x40 "retarget" forces needLOS once; a room in a DRLG *maze* level never needs LOS — #10685;
           a still-unaware pack member skips LOS if its group record +0x24 is set)
best = none; bestD = aidist[diff] (0 -> 35)
for each player list (slots 0–7):
    P = player; skip whole list if P.act != act, P in town, or dist(P) >= 55
    d = dead(P) ? INF : dist(P);  if d < bestD and (!needLOS or LOS): best = P
    for each pet of P: if dist < bestD and (!needLOS or LOS): best = pet
for each unit in slot 8 (same act): same test
lure = nearest slot-9 unit (same LOS rule)
if lure and (no best, or lure within 5 and the path to best is blocked): best = lure
if best found: aiControl.flags |= 8 (now "aware");  group+0x24 = (old group flag == 0)
```

**Distance** is D2Game **0x6FCD08F0**:

```
dist = max(|dx|,|dy|) + floor(min(|dx|,|dy|) / 2)
```
- **VERIFIED** on 10,000 native cases, all unit types, 0 mismatches.
- The mutation `min/3` gives 9,977 mismatches.

**What this means in practice**
- **Nearest wins, full stop.** Players and pets compete on distance alone. `threat` is not used here.
  - Ties go to the first unit found. That is player slot order; inside a list it is the player first, then the newest
    pet.
- **The aggro radius** is `aidist` for the difficulty, or **35 when blank**.
  - DATA: 1,158 of 1,209 enabled monsters leave `aidist(H)` blank.
  - The exceptions:
    - bats 60
    - Mephisto 46
    - Dclone, trapped souls, Rathma/Mendeln and the Ancients 128
    - some PD map bosses 50–250
- **Players further than 55 are never targeted.** The test uses `>= 55` (0x6FCD1DF7) and happens before the aidist
  test, so an `aidist` above 55 does nothing against players or pets.
  - Dclone's 128 is effectively 55.
- **A player's pets are only considered if the player himself is within 55 of the monster, in the same act and
  outside town.** A dead player still "carries" his pets.
- Candidates must be strictly closer than `aidist` (`<`).

### 2.3 The ranged finder, D2Game 0x6FCD1BC0 with filter 0x6FCD15E0 (READ)
Shooters and casters do not keep the pre-found target. Their `think` calls this finder itself. Callers seen in the
code:
- SkeletonBow, SkeletonMage, CorruptArcher, FallenShaman, Bighead, GreaterMummy, Vampire, OblivionKnight,
  PantherJavelin, FetishBlowgun, Succubus(Witch), Imp, SandMaggot, TentacleHead
- the stock Summoner, BloodRaven
- mercenaries and NPC rangers

```
if forcedTarget valid: return it
scan every unit in the monster's room and the adjacent rooms:
    valid = alive, type player/monster, hostile, not in town, attackable (unit flag 0x4)
    invisible target (state 146): a player is seen only on a 20 % roll (rolled with the target's seed) unless adjacent;
                                  a monster only when adjacent
    d = dist(monster, unit); require d <= 48 and LOS (#10839 mask 4)
    t = threat(unit) = 14 for players, MonStats `threat` for monsters (0x6FCD08A0)
    t > 1  -> keep the nearest in bucket HIGH
    t <= 1 -> keep the nearest in bucket LOW
return HIGH, unless there is no HIGH, or the LOW unit is within 5 and the path to HIGH is blocked
```

**Threat values in PD2** (DATA, MonStats `threat`):

| units | threat | bucket |
|---|---|---|
| players | 14 (hard-coded) | high |
| Valkyrie, **Decoy** (`dopplezonnew`), Shadow Warrior/Master, golems, Grizzly (druidbear), mercenaries (act2/3/4/5hire) | 11 | high |
| skeletons, skeletal mages, Fenris, most normal monsters | 10 | high |
| Oak Sage, Heart of Wolverine, Spirit of Barbs | 8 | high |
| **Bone Wall / Bone Prison** | 1 | low |
| **Hydras, all assassin sentries and Blade Sentinel, Vines/Poppies/Cycle of Life, eagle/raven** | 0 (blank) | low |

- A revived monster keeps its own row's threat, usually 10.
- **Summons with threat 0/1 are never shot while a player, merc or normal summon is within 48.**

### 2.4 Retargeting and "aggro" (READ)
- **No memory, no aggro table.** The finder runs again on **every think**. The inputs are position, act, town, LOS,
  the lists and the forced-target field. No damage or "last attacker" value is stored anywhere in MonsterData or
  aiControl.
  - I checked every write to the forced-target fields (MonsterData+0x34/+0x38): only Attract, Confuse and the Terror
    init write them.
  - **Hitting a monster does not make it target you.** It retargets whenever something else becomes nearer at its next
    think, about 0.5 s after its current action in Hell.
- **Awareness.** Before its first acquisition a monster needs line of sight to a candidate (outdoor and preset
  levels). After that (aiControl flag 8) LOS is never checked by the melee finder, so it keeps finding you through
  walls.
  - Pack members share a group record (MonsterData+0x50).
- **Ranged AIs** always need LOS and range 48 (§2.3), independent of `aidist`.

### 2.5 Where each target type comes out (summary)

| candidate | melee-finder AIs | ranged-finder AIs |
|---|---|---|
| player | yes, by distance, < aidist (35) and < 55 | yes, ≤ 48, LOS, high bucket |
| his merc / summons (threat ≥ 2) | yes, by distance, only while the owner is within 55 | yes, high bucket |
| hydra, sentries, blade creeper, vines, raven (threat 0) | yes, by distance, same condition | only if no high-bucket unit, or ≤ 5 and blocking |
| Bone Wall/Prison, Attract victim, Confused monster | only if nothing else, or ≤ 5 and blocking the path | as low bucket |
| allied NPC monsters (A5 barbs) | yes, by distance | yes, by threat bucket |

### 2.6 No target found (VERIFIED for the idle rule)
For targetMode 1, 0x6FD13220 handles the case where the finder returns nothing.

```
if the monster stands on a spot with collision flag 0x40 (#10851) or its spawn mode is 3/0x13, and MonStats2 mWL:
    wander (0x6FD11B10, type 5)
else  n = distance of the nearest same-act, non-town player (INF if none)
      idle( n >= 35 ? 25 : n < 25 ? 10 : n - 10 )     # frames
```
- **VERIFIED** on 10,000 native cases (2,040 in the idle branch), 0 mismatches. The finder, collision, room and wander
  calls were stubbed.
- The mutation `clamp(n−10, 10, 25)` gives 363 mismatches.
- So an idle monster thinks every 10 frames (0.4 s) when a player is close but out of reach, and every 25 frames
  (1 s) when everyone is 35 or more away.

### 2.7 Forced targets and special AI states: Taunt, Terror, Confuse, Attract, Dim Vision (READ)
**Forced target** is MonsterData+0x38 (mode) plus +0x34 (GUID). It is read by 0x6FCD1820 at the start of both finders.

| mode | meaning | set by |
|---|---|---|
| 1 | player with GUID | Terror init (flee source) |
| 2 | monster with GUID | **Attract**: every eligible monster around the victim, and Terror init |
| 3 | "confused": the finder temporarily flips the monster's alignment and takes the nearest living unit within 35 (search type 5) | **Confuse** (0x6FC70D6D) |
| 4 | GUID looked up in hash game+0x1920, which is the item hash | Terror init when the source is a missile (type 3). The lookup fails and the field is cleared |

If the forced unit is gone, dead or invalid, the field is cleared and the normal search runs.

**Curse → AI state.** The curse applier 0x6FC70250 (also used by PD's Taunt) maps the curse state to a special AI
state:
- state 23 `dimvision` → **10**
- state 27 `taunt` → **12**
- state 56 `terror` → **11**

When the curse ends, callback 0x6FC70130 switches the monster back to state 0.

| state | think fn | behaviour |
|---|---|---|
| 12 Taunt | 0x6FCC7F60 | The target is the taunter, taken from the monster's path target (#11129 set by Taunt). The monster walks to the taunter. When it is in melee with the taunter it **melee attacks it**. If another enemy is adjacent while walking, it hits that one instead. If the taunter dies, leaves or goes to town, it returns to its normal AI. **Ranged monsters are forced into melee.** |
| 11 Terror / flee | init 0x6FCC5AC0, think 0x6FCC6FC0 | The init stores the curse source as forced target (mode 1/2/4). Think: while state 56 is on, the monster runs away (it uses the run mode if MonStats2 `mRN`, otherwise walk). When the state ends it goes back to normal AI. |
| 10 Dim Vision | 0x6FCC5B30 | Its own blind search/idle routine (not fully analysed). Special state 17 uses the same function. |

**Who can be Taunted, Terrored, Dim-Visioned, Confused, Attracted or made to flee** (0x6FCBF120 → 0x6FCD0B20 →
D2Common #11085, plus 0x6FCBC9A0):

```
target is a monster, alive and attackable, not in state 54 (uninterruptable)
MonStats `switchai` = 1 and `boss` = 0
not unique or superunique (MonsterData+0x16 & 0xA); champions and minions are allowed
```
- If an AI-changing curse fails this test, **the whole curse is not applied**. The stats and duration are skipped too
  (0x6FC702A7).
- Confuse (0x6FC70BC7) and Attract (0x6FC6EE97) use the same gate.

Every curse also passes 0x6FC6ECD0. **It fails when:**
- the monster is **Possessed** (champion mod, MonsterData+0x16 & 0x20)
- its MonStats2 has no walk mode (`mWL`)
- it is not selectable or attackable

DATA: 147 killable non-NPC monsters have `switchai` blank. Among them:
- **all 15 Oblivion Knights** (incl. PD Void Knights and Putrid Defiler dungeon copies)
- the Minions of Destruction (`BaalMinion` AI, 9 rows)
- all suicide minions
- Putrid Defilers
- the Summoner-AI map bosses
- Blood Raven, Tentacles, catapults, nests and eggs

These can be neither taunted nor terrored.

**Other effects:**
- **Confuse**:
  - the monster's alignment is flipped to 1
  - it is put in the lure list (slot 9), so other monsters attack it
  - it gets forced mode 3, so it attacks the nearest living thing
  - an end-of-confusion event (type 10) is queued at the curse's end frame
- **Attract**:
  - the victim goes into slot 9 with alignment 1
  - every eligible hostile monster in range gets forced mode 2 on the victim (0x6FC70DB0)
- **Howl** (missile hit 0x6FC5EB50) triggers the flee state 11 through 0x6FCD2120, the same gate as Terror.
  - It only works if caster level plus a skill-level term is greater than the monster's level. Duration comes from
    the missile.
- **"Hit causes monster to flee"** (item event 0x6FCCF2E0) flees for 20 frames when `rand(128) < value`.
  - It is not applied to champions or uniques (MonsterData+0x16 & 0xC). The Terror gate above also excludes
    superuniques and non-switchai monsters.
- **War Cry / stun** (state 21) is not a special state:
  - each think ends in `idle 3`
  - a finished mode reschedules at +45 (VERIFIED formula §1.2)
- **Battle Cry** (do 68, stock 0x6FC49720) has no AI effect. Only states 23/27/56 are mapped.
- **Decoy** (`dopplezonnew`), **Valkyrie** and **Revive** have no targeting pull of their own.
  - They are ordinary pets (threat 11, or 10 for revives). They take hits because they are the *nearest* unit (melee
    finder) or the nearest high-threat unit (ranged finder).
  - Decoy and Valkyrie use the ShadowMaster AI, whose init PD replaces (§3.3).

---

## 3. How a monster chooses its attack or skill

### 3.1 Structure
- Every `think` uses the tick record (target, distance, `inMelee`) and rolls the unit's own LCG. The LCG is seeded at
  unit+0x20/+0x24: `new = lo·0x6AC690C5 + hi`, and `roll(n) = lo % n`. Each call consumes one step.
- The `aipN` columns are **percent chances, frame counts or distances**, depending on the AI.
- The actions are a small set:
  - start mode A1/A2/S1… (0x6FC95E10)
  - use skill k (`MonStats SkillK / SkKmode`, e.g. PD 0x102ECD80)
  - walk or run to the target (0x6FD12540)
  - step away / circle (0x6FD126B0, 0x6FD11D90)
  - wander (0x6FD11B10)
  - idle n (0x6FD122C0)
- Most AIs are "roll → action" trees. The MonAi.txt `*aipN` comments are the developers' own labels.

### 3.2 Common AIs (the aip values are one Hell example, DATA)

**Skeleton** (MonAI 2, 0x6FCA3AC0). **VERIFIED:** 10,000 native cases including RNG state, 0 mismatches.
Example: Bone Warrior `skeleton3`, aidel(H) 13.

| param | Hell | meaning |
|---|---|---|
| aip1 approach? | 90 | not in melee: % chance to walk at the target; otherwise stall aip2 |
| aip2 stall time | 7 | frames to stand still when it does not act |
| aip3 attack? | 95 | in melee: % chance to attack; otherwise stall aip2 |
| aip4 att1/att2? | 50 | % of attacks that use A1 (the rest use A2) |

```
inMelee:  roll<aip3 ? (roll<aip4 ? A1 : A2) : idle(aip2)
else:     roll<aip1 ? walk to target      : idle(aip2)
```
Worked example: a Bone Warrior in melee attacks 95 % of thinks. Each think is about (attack length + 13) frames apart.
In the other 5 % it waits 7 frames.

**Zombie** (MonAI 3, 0x6FCA38B0). **VERIFIED** (10,000 cases, 0 mismatches; the mutation "no Burial-Grounds rule"
gives 570). Example: Drowned Carcass `zombie4`.

| param | Hell | meaning |
|---|---|---|
| aip1 approach? | 100 | % to walk at a target closer than aip2 |
| aip2 aware dist | 30 | only targets nearer than this make it walk (the finder already limits to aidist 35) |
| aip4 att1/att2? | 45 | in melee it **always** attacks, A1 with this % |

- Farther targets make it wander.
- In **Burial Grounds (level 17)** it always walks at the target.
- A spawn mode of 3/0x13 (MonsterData+0x54) always walks at the target.

**SkeletonBow** (MonAI 37, 0x6FCA6280, READ). Uses the ranged finder. Example: Horror Archer `sk_archer10`.

| param | Hell | meaning |
|---|---|---|
| aip1 shoot? | 99 | target within 20: % chance to shoot (attack A1) |
| aip2 stall time | 11 | otherwise: 20 % chance to reposition, else stall this many frames |
| aip3 approach? | 50 | no target within 20: % chance to walk toward one; otherwise stall 20 |
| aip4 walk steps | 5 | steps per approach |
| aip5 tgt dist | 12 | the distance it tries to walk to |

**Oblivion Knight** (MonAI 74). **PD replaces it completely** with PD 0x102B4BB0, which never calls the stock
0x6FCA4DB0 (READ). Example: every PD OK and Void Knight has the values below.
- Skill1 DoomKnightMissile, Skill2 MonBoneArmor, Skill3 MonBoneSpirit, Skill4 MonCurseCast.

| param | Hell | meaning in PD's code |
|---|---|---|
| aip1 flee+curse range | 8 | target closer than this: curse and back off |
| aip2 engage range | 27 | a ranged-finder target closer than this is shot at |
| aip3 curse timer | 200 | **written to aiControl+0x14 but never read by PD's AI** (stock uses it as a cooldown) |
| aip4 curse? | 50 | % to cast Skill4 MonCurseCast when the target has none of Amp (9), Weaken (19), Decrepify (60), Lower Res (61) |
| aip5 shoot? | 90 | % to use a missile at an engaged target |
| aip6 bonespirit? | 30 | of those, % that are Bone Spirit (Skill3); the rest are Skill1 |
| aip7 approach? | 30 | beyond aip8: % to walk toward the target |
| aip8 approach dist | 11 | the distance where approaching starts |

```
d < aip1:  if Skill4 and target has no amp/weaken/LR/decrep and roll<aip4: cast Skill4 (random of MonAmplifyDamage,
                MonWeaken, MonLowerRes, MonDecrepify — PD do 112)
           else run away (10); if that fails and Skill3 exists: Skill3
else:      T = ranged finder; if T and dist<aip2 and roll<aip5: roll<aip6 ? Skill3 : Skill1
           else if dist>aip8 and roll<aip7: walk to target
           else 70 %: wander move, 30 %: idle 10
```

**AIs by meaning** (DATA: the parameter names are MonAi.txt's comments and the values are PD Hell examples).
These were not read line by line; the finder column is READ.

| AI (example) | finder | parameters (Hell) |
|---|---|---|
| Goatman (Hell Clan) | melee | approach? 95, stall 7, attack? 99 |
| Fallen (Devilkin) | melee | cmd:attack? 60, aip2 30, attack? 90, A1/A2 40. **PD branch for aip8(H)=1, see §3.3** |
| FallenShaman (Warped Shaman) | ranged | rez?/cmd? 90, shoot? 95, melee?/circle? 80, rez dist 30, shoot dist 15 |
| SkeletonMage (Bone Mage) | ranged | shoot? 85, approach dist 20, approach? 30, too close 7, walk away? 50, fire dist 20, circle? 20, stall 5 |
| Vampire (Dark Lord) | ranged | melee? 76, cast? 70, active dist 27, upgrade cast? 43, spell flags 7 |
| Bighead (Damned) | ranged | hurt% 33, circle? 30, fire healthy? 85, fire hurt? 80 |
| CorruptArcher (Flesh Archer) | ranged | approach? 99, shoot? 99, stall 6, run? 60, always run dist 20, walk tow dist 12 |
| HighPriest (Council) | own | engage? 65, heal at range? 25, heal/hydra timer 75, hydra? 50, ltng at range? 80, disengage? 9, ltng engaged? 12, range 30 |
| Minion (Greater Hell Spawn) | melee | attack? 80, melee stall 10, approach? 60, ranged stall 15, A1/A2 50 |
| Succubus (Blood Temptress) | ranged | attack? 95, approach? 10, curse? 50, curse range 25, melee stall 11, ranged stall 15, curse level 4 |
| SuccubusWitch (Hell Witch) | ranged | attack? 90, approach? 25, walk away? 50, comfort dist 12, shoot? 90, stall 13, amp if tgt hp% 80, weaken if my hp% 66 |
| Overseer (Blood Boss) | melee | rally timer 250, heal? 65, whip? 50, comfort 17/7, attack? 100, A1/A2 45 |
| ReanimatedHorde (Defiled Warrior) | melee | attack? 85, melee stall 12, charge range 18, charge? 45, follow? 35, walk fwd? 65, ranged stall 19, reanimate 15 |
| Megademon (Venom Lord) | melee | inferno ranged? 90, inferno melee? 65, swing? 97, approach? 70, circle? 70, inferno timer 45 |
| Summoner (The Summoner) | ranged | cast? 98, weaken? 5, pref element 63, nova timer 40, firewall timer 80, walk away? 10, nova dist 11, missile dist 40 |

Low-life behaviour lives in the AIs that have a `hurt%` / `weak%` / `wounded%` parameter:
- Bighead, BatDemon, Baboon, SandRaider (33 %)
- Fetish (weak 33 %)
- Vulture (50 %), ZakarumZealot (50 %)
- Arach (45 %)
- FingerMage (healthy 70 / hurt 50)

Below that life % they switch branch: fire more (Bighead), disengage or regenerate (BatDemon, Baboon), or run
(Zealot "run?", Arach "run dist"). There is **no generic flee at low life**. Generic fleeing comes only from
§2.7 (Terror, Howl, item flee), and from the teleport mod (§4.1) below 30 % life.

### 3.3 PD2 changes to monster AI
**How PD installs them.** PD writes new function pointers straight into D2Game's AI table at runtime. PD 0x102BAAE0
does the writes, through the lazy pointer 0x104E2CB4 → D2Game +0x10E788. There are **no static patch records on the
table**, which is why a patch-record scan misses them. Most overrides test a class or a data field and otherwise jump
back to the stock AI.

| MonAI slot | PD fn | applies to | what it does (READ, depth varies) |
|---|---|---|---|
| 6 Fallen | 0x102B3EB0 | rows with **aip8(H) = 1** → `treasurefallenMap` **Treasure Fallen** | runs away from its target; if it has not moved for 3 thinks and the target is within 10, **Blinks** (Skill1) 2–4 units away; each time life drops past another 10 % it **drops loot** (0x102D5B80 → D2Game drop). Others → stock 0x6FCA3130 |
| 7 Brute | 0x102B13F0 | class 1103 Naz the Cursed King | not in state 13: Skill1; else if dist < 20: 20 % Skill2, else 20 % Skill3, else idle 2×aidel(H) (C `rand()`) |
| 22 GreaterMummy | 0x102B91F0 | class 1104 Elmegaard | 30 % Skill1, 30 % Skill3, 30 % Skill4 at target, 10 % Skill2 at a random point ±20 |
| 25 WillOWisp | 0x102B0A30 | aip7(H) > 0: Canight, Madness, Hysteria | spawns its MonStats minion at ±8 when life ≤ aip4(H) with a roll ≤ aip5(H), cooldown aip6(H) (stat 446 `mon_cooldown1`); Skill4 toward a far boss partner (game+0x2600); Skill1 roll ≤ aip7(H); Skill3 roll ≤ aip8(H) |
| 32 Npc | 0x102B0E40 | class 147 Gheed, 1168 Emperor Hakan | custom behaviour for the two; others stock |
| 35 CorruptArcher | 0x102B0FF0 | class 894 Dark Commander Alma | timed phases: sets fire/light/cold/poison resist 99, magic 50, DR 75 and a state, then restores them from MonStats; alternates Skill2; others stock |
| 45 Sarcophagus | 0x102B0FE0 | all | jumps to stock 0x6FC9CFD0 (no change) |
| 47 FlyingScimitar | 0x102B0C80 | aip7(H) > 0: Waheed the Traitor | cooldown-driven skills; others stock |
| 49 ZakarumPriest | 0x102B1300 | aip7(H) > 0: Cantor Boss | when life ≤ aip7(H) % (66): casts Skill8 (MinionSpawner) once, then melee-only with Skill5 (walks in, uses it at distance ≤ 2) |
| 50 / 51 / 135 Mephisto, Diablo, BaalCrab | 0x102B16A0 / 0x102B1850 / 0x102B1A00 | the Uber Tristram bosses | minion spawning per AI tick, then stock (uber_review.md) |
| 53 Summoner | 0x102B2010 | Synthetic One, Onion the Accused, Rathma/Mendeln + clones, Sanguine Bearer | boss scripts (uber_review.md for Rathma); other Summoner-AI monsters → stock |
| 61 Hireable | 0x102B3D70 | all mercenaries | if the merc's Hireling row has a flagged self-cast skill it is high enough for and it lacks that skill's state, it casts it, then idles 10; hireling versions 0x19/0x1B/0x1D get an extra PD routine; then stock merc AI |
| 71 Regurgitator | 0x102B41C0 | 1064/1065 Guardian of Fate | boss script |
| 72 DoomKnight | 0x102B4440 | 1060 Warlord of Blood | boss script |
| **74 OblivionKnight** | 0x102B4BB0 | **every OK** | full rewrite (§3.2) |
| 87 Trap-Melee | 0x102B4F00 | aip3(H) > 0 → Diablo Clone | Dclone script (uber_review.md) |
| 89 Megademon | 0x102B1210 | aip7(H) > 0: Lieutenant of Sin (1350/1350) | Skill2 every aip7(H) AI ticks (stat 446 countdown), Skill3 every aip8(H) ticks (stat 447), otherwise stock |
| 93 ArcaneTower | 0x102B6640 | 966 Mendeln spire | boss script |
| 99 TrappedSoul | 0x102B5770 | Dclone souls, 1057 Demonic Sentinel, 922 Shadow of Mendeln | boss scripts |
| 102 BladeCreeper | 0x102B9820 | the assassin summon | rewritten (hit roll via PD 0x1026F8F0) |
| 105 / 106 ShadowWarrior / ShadowMaster | init 0x102BA3A0 + think 0x102B9D50 / init 0x102BA480 only (think stays stock 0x6FCCB580) | Shadow Warrior (105); Shadow Master, **Decoy, Valkyrie** (106) | PD init (and think for 105); per MonAi.txt (PD's own comments): max target dist, max boss dist, attack chance, skill decrement / approach dist, melee bonus, random pick, ignore range, **boss leash** |
| 114 ReanimatedHorde | 0x102B7570 | 998 King Leoric (map) | boss script |
| 115 SiegeBeast | 0x102B9330 | 1000 The Stygian Beast | boss script |
| 120 Overseer | 0x102B77B0 | 1112/1113 Lucion, 1141, 1105 Skullsplitter Cyrus | boss scripts |
| 133 Ancient | 0x102B66D0 | uber Ancients (game type 0x2C) | boss script |
| 141 BaalMinion | 0x102B1BA0 | 996 Urzael, 938 Sanguine Bearer | boss scripts |

**Other PD code in the AI path**
- **Pet leash** (replaces the stock "command target" check at D2Game +0x9CDC6; PD 0x102BA0C0). For a monster whose
  owner (aiControl+0x2C/+0x30) is a **player**:
  - A target that is more than about 27.9 units away (√780) is dropped. It must be a live monster.
  - The leash radius is 17 units, or about 27.9 while the pet has a target.
  - Inside the radius it idles or makes a random regroup step (1 in 12).
  - Outside the radius it walks back.
  - At 48 or more (dist² ≥ 0x900) it is pulled with a mode-3 move.
  - ShadowMaster-AI pets skip this and Blade Creeper uses the stock check.
- **Arena freeze** (wrapper around the target search, PD 0x102B9760):
  - The Rathma, Dclone and Lucion scripts decrement `game+0x26FE`. A PD hook sets it to 15 when one of the ten boss
    slots at `game+0x2600` is involved in an event (PD 0x10269070).
  - While it is > 0, monsters whose class is in one PD set and not in another skip their search and idle 20 frames.
- **Special-state switch** (D2Game 0x6FCBD230):
  - PD NOPs the assert at +0x9D247 (33 bytes) that fired when a monster in state 54 changed AI state.
  - Code-init NOPs at +0x9D2E2/+0x9D2E4/+0x9D2E9 remove the zeroing of the AI scratch words aiControl+0x14/+0x18/+0x1C.
    **The scratch now survives a Taunt, Terror or other state change.** PD boss scripts keep counters there.
- **Taunt** (do 71 → PD 0x102FAD00, callback 0x102C5C60) still goes through the stock curse applier 0x6FC70250. The
  AI side (special state 12, gates) is unchanged.
- **New per-monster cooldown stats** `mon_cooldown1..3` (ItemStatCost 446–448) are counted in AI ticks by the PD
  scripts. **The PD scripts read the `(H)` aip columns in every difficulty.**

---

## 4. Champion and unique mods, and bosses

### 4.1 Teleport (MonUMod 26) — READ, 0x6FC441B0 + 0x6FCBCDE0
The mod gives the monster skill 184 `MonTeleport` (do 98) and sets aiControl flag 0x20. Then, on every think that
has a target:

```
if roll(100) >= 40: nothing
melee = MonStats isMelee, or base class 10 (Bighead), 345 or 557 (Council Members)
if life% >= 30 and (melee or target distance >= 10): nothing
if roll(100) >= 15: nothing
pick a random free spot nearby (0x6FC67450), not in town
if life% < 30 and roll(100) < 25 and not in state 52: life += monster level  (whole points, capped at max)
cast MonTeleport there
```

So each think with a target has these chances:
- **6 %** to teleport when below 30 % life
- **6 %** for a *ranged* monster whose target is closer than 10
- **0 %** otherwise

**The "teleport heal" is tiny:** 25 % of low-life teleports add *monster level* life. That is +85 HP for a level-85
unique.

### 4.2 Other mods (READ)
| mod | AI effect |
|---|---|
| 38 possessed | MonsterData+0x16 \|= 0x20, plus a +100 scaling through 0x6FC41EC0 (stats: hit_pd2.md). **Cannot be cursed at all** (0x6FC6ED07), so no Taunt, Terror, Attract, Confuse or Dim Vision |
| 36 ghostly, 37 fanatic, 39 berserker, 16 champion | stat changes only (hit_pd2.md §4). No AI code reads them |
| 30 aura | chooses an aura by monster level (0x6FC44CD0). The aura runs as a normal aura (auras.md) and does not change the AI |
| 41 always_run_ai | queues event 7 at +75 frames (0x6FC469F0) |
| unique / superunique flag | immune to Taunt, Terror, Dim Vision, Confuse, Attract, Howl and item-flee (§2.7). Their **minions and champions are not** |

### 4.3 Fleeing
There are only four ways a monster flees:
- **Terror curse**
- **Howl**, when the level test passes
- **"hit causes monster to flee"** items, not on champions or uniques
- AI-specific `hurt%` branches (§3.2)

The teleport mod repositions below 30 % life but does not flee. All the curse-based flees share the §2.7 gate.

### 4.4 Boss "notice" pause — READ, 0x6FCBCA60
A unique, a MonStats `boss`, or The Summoner (class 250) pauses **20 frames** the first time a *player* target is
within 20:
- it plays its notice sound (0x6FCFFFD0, event 0x10)
- it sets aiControl flag 0x10, so this happens once

---

## 5. Worked examples (PD2 Hell numbers)

1. **Target choice.** A Drowned Carcass (`zombie4`, aidist blank → 35) stands where the necromancer is 30 away, his Revive 12
   away and his merc 20 away.
   - It takes the Revive, because 12 < 20 < 30.
   - If the necromancer steps back to 56, the whole list is skipped. With no other player near, the zombie gets no
     target and idles 25 frames, even though the Revive is still 12 away.
2. **Ranged choice.** A Horror Archer sees a player at 30, an Oak Sage at 15 and a Hydra at 8, all in LOS.
   - The Hydra has threat 0 and goes in the low bucket. The Oak Sage (threat 8) and the player (14) go in the high
     bucket.
   - It shoots the **Oak Sage**, the nearest high unit. The Hydra is only shot if nothing in the high bucket is within
     48.
3. **Distance.** dx = 10, dy = 4 → 10 + 2 = 12. dx = dy = 20 → 30, not 28.3.
4. **Think rate.** A Hell Bone Warrior (aidel(H) 13) finishing a 16-frame attack thinks again 13 frames later, so it
   decides about once every 29 frames (1.16 s) in melee.
   - If it is out of reach and a player is 30 away, it idles 20 frames per think (§2.6).
5. **Teleport unique.** A level-88 teleporting ranged unique at 60 % life, with its target 8 away, thinks every
   ~0.5–1 s. It has a 6 % chance per think to blink. Below 30 % life the chance is still 6 %, and 1 in 4 of those
   blinks heals 88 life.

---

## 6. Verification record and gaps

**VERIFIED** (`harness/aipick`, 50,000 random cases, 0 mismatches, RNG state compared too):
- 0x6FCD08F0 distance
- 0x6FCA3AC0 Skeleton think
- 0x6FCA38B0 Zombie think
- 0x6FC94E60 mode-end rescheduler
- the no-target idle branch of 0x6FD13220

Harness setup:
- Actions (start mode, walk, idle, wander, event) and the D2Common imports were stubbed and recorded.
- The finder and collision helpers were stubbed for the 0x6FD13220 case.

Checks:
- Mutations: distance `min/3` gives 9,977 mismatches; aidel column choice gives 1,134; the Burial-Grounds rule gives
  570; the idle clamp gives 363.
- `node harness/aipick/check_js.js` replays the 40,000 applicable cases through `monster_ai.js`: 0 mismatches.

**READ, not run:**
- the full finders 0x6FCD1C90 and 0x6FCD1BC0 (lists, threat buckets, lure rule)
- the forced-target modes
- the special-state gates
- the Taunt, Terror and Confuse behaviour
- the teleport mod
- the boss notice pause
- all PD overrides. They were skimmed for gating and main actions; Treasure Fallen, Megademon, ZakarumPriest,
  OblivionKnight and Brute were read fully.

**Not analysed:**
- Dim Vision's think (0x6FCC5B30)
- special states 2–9 and 13–15
- the room check 0x6FCBCCF0
- stock SkeletonMage, Vampire, HighPriest and the other §3.2 "by meaning" AIs beyond their finder
- the PD ShadowWarrior/ShadowMaster and Blade Creeper internals
- what exactly the MonsterData+0x50 group record groups (pack or boss + minions)

PD2 realm servers could run different server code. This covers the client install only.
