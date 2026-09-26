# Part B — the damage application pipeline (resolved hit → life/mana/stamina → hit effects)

Scope: what happens to a hit **after** the hit roll and the attack/skill damage build-up (parts A and C), up to
the defender's life, mana and stamina changes, the hit reaction (get-hit, knockback, block), the on-hit/on-struck
effects, and the kill/death events. Reverse-engineered from stock 1.13c D2Game/D2Common and ProjectDiablo.dll
(PD) runtime patches only. Flow-chart data: `adv/re/flow/flow_B.json` (stage ids `B.*`, schema `flow/SCHEMA.md`).

Status: **VERIFIED** = the real code ran natively in a harness and matched the formula on every case;
**READ** = disassembly only; **DATA** = PD2 txt files. Units: life, damage, leech and poison are 8.8 fixed point
(×256); `tdiv` = C division (toward zero). Stat numbers are ItemStatCost ids.

New native checks for this part: `harness/pipeB.c` + `harness/pipeB_check.py` (build line in the .c file).
9 modes × 20,000 random cases, **0 mismatches**; a 3-mutation run (hit-class divisor, half-freeze rule,
knockback compare) was detected by modes 1, 4 and 7, so the check is sensitive. Earlier VERIFIED pieces are
cited from `damage.md`, `crit_cb.md`, `defense.md`, `bug_rathma_share.md`, `bug_boss_instadeath.md`.

| mode | real code run | what was confirmed |
|---|---|---|
| 1 | D2Game 0x6FCFB910 (+ real RNG 0x6FC21450) | get-hit (hit recovery) thresholds by hit class, random bands, RNG draws |
| 2 | PD 0x102AEE30 (event func 6) | thorns: element placement, missile DamageRate scaling, player exemption, OW trigger |
| 3 | PD 0x1026F190 | stun length caps, unique 90 % resist, boss/immobile immunity, refresh rule |
| 4 | PD 0x102713C0 | cannot-be-frozen / half-freeze / shrine length zeroing |
| 5 | PD 0x10271A70 | poison state per player, replace-if-stronger rule, expiry |
| 6 | D2Game 0x6FCCD580 (event func 14, real RNG) | freeze-target chance and freeze length |
| 7 | PD 0x102AEFE0 → D2Game 0x6FCCCE60 (real RNG) | knockback chance by MonStats2 size, PvP block |
| 8 | PD 0x102AFA40 → D2Game 0x6FCCD2A0 | slow caps by monster kind, prime-evil/PvP block |
| 9 | D2Game 0x6FCB8A70 | Iron Maiden return amount (1/8 vs players/mercs) |

---------------------------------------------------------------------------------------------------------

## 0. The damage struct (0x70 bytes, D2Game)

| off | meaning | off | meaning |
|---|---|---|---|
| +0x00 | input flags: 0x02 preset damage (skip fill), 0x04 no life leech, 0x08 no mana leech, **0x20 "attack damage" (item events allowed)**, 0x80 no attacker missile event, 0x100/0x200/0x400 bypass undead/demon/beast (stats 103/104/106) | +0x04 | result flags: 0x01 hit, 0x02 will die, 0x04 get-hit, 0x08 knockback, 0x10 block, 0x20 suppress events, 0x2000 crit/DS, 0x4000 no reaction, 0x8000 weapon block; 0x80/0x100/0x200 avoid/dodge/evade |
| +0x08 | physical | +0x0C | enhanced-damage % of the fill |
| +0x10 | fire | +0x14 / +0x18 | burn rate / burn length |
| +0x1C | lightning | +0x20 | magic |
| +0x24 | cold | +0x28 / +0x2C | poison rate per frame / length |
| +0x30 / +0x34 | cold (chill) length / freeze length | +0x38 / +0x3C / +0x40 | life / mana / stamina leech |
| +0x44 | stun length | +0x48 | absorbed life (healed) |
| +0x4C | total (post-resist) | +0x54 | DR scale = missile stat 327 `damage_framerate` (/1024) |
| +0x5C | source missile / thorns owner | +0x60 | hit class (low nibble = HitClass.txt row) |
| +0x65 / +0x68 | physical conversion type / % (A's fill) | | |

The fill (D2Game 0x6FCFD450, part A) always sets +0x00 |= 0x20. Physical conversion (+0x65/+0x68, READ):
`c = muldiv(phys, pct, 100); phys = max(phys − c, 0)`; type 1 fire, 2 light, 3 magic, 4 cold (+cold length ≥ 50),
5 poison += c/8 (length ≥ 50), 10 random 1–5, 11 fire (+burn length ≥ 50), 12 cold (+freeze length ≥ 50).

---------------------------------------------------------------------------------------------------------

## 1. Order of operations

### 1a. Melee hit on a monster (player or merc attacker) — two phases

**Swing start** (resolver PD 0x10270EB0, which replaces stock 0x6FCFE5A0 at 31 call sites; hit roll/block: A and D):

1. Hit roll + block/avoid (PD 0x10271740, 0x1026FB60; `hit_pd2.md`). A hit gets result flag 0x01 and, unless the
   defender has state 54 `uninterruptable`, **0x04 get-hit request** (PD 0x10270FD0).
2. `item_preventheal` 117 → state 52 on monsters (not classes 704–709): PD 0x102EFAF0 → 0x6FCFCE70.
3. **Damage store** D2Game 0x6FCFDDE0: skipped entirely if result flags & 0x8380 or no hit (the node is still
   queued as a miss).
   1. Fill 0x6FCFD450 (part A): physical roll, **B.crit / B.deadly** (PD 0x10270E00), elemental rolls, leech %,
      bypass flags, poison, cold length, stun, burn, conversion, then **event 3 `attackedinmelee` on the
      defender** (0x6FCFD97B) — only for player or merc attackers.
   2. **Resist pass** D2Game 0x6FCFC0B0 (§2) — at swing start.
   3. Kill prediction: `(total & ~0xFF) > (life & ~0xFF)` → +4 |= 0x02 (monster attackers add +0x38).
   4. Queue the 0x70-byte struct on attacker+0xAC (0x6FCFAB90).

**Impact frame** (the skill's attack event → 0x6FCFED30):

4. Find the queued node for (attacker, defender). #11138 range/validity check; if it fails the node is dropped.
5. If hit (+4 & 1) and the attacker is alive: **execute** 0x6FCFE0C0 with `isMissile = 0` (no second resist pass):
   1. Defender dead/dying (0x6FCFFA90) or a monster without MonStats `killable` (flag bit 15) → no damage.
   2. **Events** (if +0 & 0x20 and not +4 & 0x20): event 5 `domeleedamage` on the attacker (**B.cb**, **B.ow**,
      knockback, freeze-target, slow, howl, stupidity, cast-on-hit 198, heal-after-hit 424, splash 359), then
      event 1 `damagedinmelee` on the defender (cast-when-struck 201, damage-to-mana 114, Life Tap, Blood/Clay Golem).
   3. **B.overkill_clamp**: `phys = min(phys, current life)` (physical only).
   4. **B.leech** (PD 0x102700F0 → 0x6FCFBA40).
   5. Monster attacker only: unique-monster modifier on-hit dispatch 0x6FC468E0 (PD 0x10253DE0 adds a 1-frame
      state 86 `justhit` throttle).
   6. **B.absorb_heal**: `life += absorbed(+0x48)` capped at max life (0x6FCFB570; PD stat-488 gate).
   7. **B.life_subtract**: if total > 0: `life = life − total; if life < 256 → 0`.
   8. Mana drain `mana −= +0x3C` (< 256 → 0), stamina drain `stamina −= +0x40` (0x6FCFA970 / 0x6FCFA930).
   9. **B.stun** (PD 0x1026F190), **B.chill** (0x6FCFC780), **B.freeze** (0x6FCFDAC0), **B.poison** (PD 0x10271A70),
      **B.burn** (0x6FCFC940) — in that order.
   10. Life > 0 → clear +4 & 2; a damaged monster with `hpregen` gets its regen timer restarted (frame+1);
       0x6FCB0F30 (monster AI hit notice). Life = 0 → +4 |= 2.
   11. Will die → event 10 `killed` on the defender (197 skill-on-death, 453), event 9 `kill` on the attacker
       (86 heal-after-kill, 138 mana-after-kill, 139 demon heal, 108 rest-in-peace, 155 reanimate, 196).
6. Bookkeeping calls #10689 / #10862 on the units (READ, not identified further).
7. **Always (hit, miss or block)**: event 7 `domeleeattack` on the attacker (skill-on-attack 195) and event 3
   `attackedinmelee` on the defender (**thorns 78/128**, Shiver/Chilling Armor).
8. **Post-hit**: stock 0x6FCB8A70 (Iron Maiden) is NOP'd at 0x6FCFEE85 (14 bytes); 0x6FCFCF20 is replaced at
   0x6FCFEE9D by **PD 0x1026EDD0**, which runs Iron Maiden (if +4 & 0x20 = 0 and +0 & 0x20), the PD block-delay
   and `doblock` timer, then stock 0x6FCFCF20 (hit class, get-hit / knockback / block / death animations), then
   the PD Sacrifice kill branch (§7).
9. 0x6FCFA6D0: free the node.

### 1b. Missile hit
Collision PD 0x102723E0 (hit roll for Missiles.txt ToHit = 1 only) → damage **PD 0x1026E7D0** (replaces
0x6FC5AE10 at 0x6FC5AFF7 and 0x6FCFE559):
hit flag; state 54 test; Missiles.txt hit class (+4) → get-hit (0x04) or no reaction (0x4000); **knockback
Missiles.txt `KnockBack` (+0xAE) % with pdRand**; block/avoid (PD 0x1026FB60); hit class (#10677); MonStats `Crit`
(monster owners); +0 |= 0x20 / 0x80 from the missile flags (#10623); +0x54 = missile stat 327; then execute
0x6FCFE0C0 with **isMissile = 1**: resist pass now (at impact), events 6 `domissiledamage` / 2 `damagedbymissile`,
leech halved; then PD post-hit 0x1026EDD0 with the missile's **owner** as attacker (so Iron Maiden also reflects
missiles in PD2, §5). Missile damage roll and crit: part C (`damage.md` §1b).

### 1c. Hit on a player (monster attacker)
Same code. Differences: per-type resist caps/penalty/absorb apply (`defense.md` §1.3), monster leech is a flat
drain (0x6FCFBA40 non-player branch: life/mana/stamina drain clamped to phys / defender mana / stamina), MonStats
`Crit` doubles everything, damage percent (PD 0x1026D1D0), freeze becomes chill, get-hit uses the same thresholds
(§4.1). Owned by part D.

---------------------------------------------------------------------------------------------------------

## 2. Resist pass D2Game 0x6FCFC0B0 (VERIFIED as a whole, `defense.md`)

Order inside the pass:
1. DR8 = stat 34 << 8, MDR8 = stat 35 << 8; if +0x54 > 0 each ×(+0x54)/1024 (missile `damage_framerate`).
2. **B.dmg_percent**: PD 0x1026D1D0 (replaces 0x6FCFAC10 at 0x6FCFC180) scales all fields itself and returns 100.
   Player/summon → monster: 100 %. Non-100 cases (merc, monster-owned monster 50 %, PvP 25/85 %, prime evil
   300 % vs flagged units) are part D.
3. **B.absorb_event**: event 11 `absorbdamage` on the defender — **before resistances**: Energy Shield (PD
   0x102AFC70), Bone Armor (func 22, PD 0x102AFAB0), Cyclone Armor (func 25, PD 0x102AFE90).
   ES (READ): for each of phys/fire/light/cold/magic (and life/mana/stamina leech when the attacker is a non-merc
   monster; table D2Game 0x6FD1C3E8): `a = min(trunc(v·calc1/100), trunc(mana·16/calc2))`, `v −= a`,
   `mana −= trunc(a·calc2/16)`.
4. **B.len_prestep** PD 0x102713C0 (replaces 0x6FCFA7B0 at 0x6FCFC478) — VERIFIED mode 4:
   ```
   if cold len (+0x30) or freeze len (+0x34) > 0:
       stat 153 cannot_be_frozen ≠ 0          → both = 0
       else stat 118 half_freeze == 1           → both >>= 1
       else stat 118 > 1                        → both = 0          (stock: any non-zero halves)
   poison len (+0x2C) > 0 and state 133 (poison shrine) → 0
   burn len (+0x18) > 0 and state 131 (fire shrine)     → 0
   ```
5. **B.bypass**: defender monster matching attacker stat 103/104/106 (undead/demon/beast) → bypass (positive
   resist, DR and absorb ignored; negative resist still applies).
6. Per damage type over the table 0x6FD22AB0 (12 rows; `damage.md` §2), PD 0x1026F410 (call at 0x6FCFC4E9):
   - **B.pierce** (effective resist, PD 0x1026F680 / 0x1026EA70, VERIFIED):
     `res = defender stat; if res < 100 or defender not a monster: res −= attacker pierce;`
     `if res < 0 and attacker (or its owner, 0x102CB180) is a player: res = tdiv(res, 2); if res < 1: res = max(res, −100)`.
     Pierce ids: phys 425 (PD), fire 333, light 334, cold 335, poison 336, magic 358 (PD). Players/mercs as
     defenders: difficulty penalty and caps (`defense.md`). Sanctuary (attacker state 47) → phys res 0 vs
     non-prime-evil undead (READ, `damage.md`).
   - **B.dr** / **B.mdr**: `v = max(v − DR8, 0)` for phys (34), `v = max(v − MDR8, 0)` for fire/light/cold/magic (35).
     PvP maps cap each at 25.
   - **B.resist**: `if v > 0 and res ≠ 0: v = trunc(v·(100 − min(res,100))/100)` (double math).
   - **B.absorb**: `a = trunc(v·min(abs%,40)/100); v −= a; F = absFlat<<8; a2 = min(v, F); v −= a2;`
     absorbed (+0x48) += a + a2; then `v = max(v, 0)` (PD clamps each type; stock let a negative phys eat other types).
   - Poison rate row 8 and length row 7 (stat 110 + pierce 336), cold/freeze length rows 5/6 use cold resist.
   - Player-owner damage meter: playerdata+0x1A8 += min(v>>8, life>>8).
   - **B.rathma_share** (bug, `bug_rathma_share.md`): defender class 0x3A7/0x3A8 with both slots set → `v = v/2`,
     partner life −= v.
7. **B.total**: +0x4C = phys + fire + light + magic + cold + poison(one frame). Burn (+0x14) is **not** in the
   table: burning ignores fire resistance, absorb and immunity (READ, stock).

Curses/auras never enter this pass directly: they change the defender's stats. PD 0x102C0540 halves each
negative contribution to 36/37/39/41/43/45 when the monster's **base** value is > 99 (**B.immune_halving**,
VERIFIED; stock 0x6FC6E230 used ÷5; the Confuse path still uses ÷5).

---------------------------------------------------------------------------------------------------------

## 3. Crit/DS, crushing blow, open wounds (cited, VERIFIED)
- **B.crit / B.deadly** PD 0x10270E00 / 0x10270D20 — physical only, before resist; sum of crit sources capped
  75, one roll, DS only if crit fails (`damage.md` §1b, `crit_cb.md` §1).
- **B.cb** PD 0x102AF610 (event func 16) — share of **current** life before this hit's damage; raw stat 36
  (negative counts in full, no −100 floor); immune ≥ 100 → none; ÷8 monsters, ÷10 players/mercs, prime evil
  70 + 10·(100−life%) or 30 (map bosses); ×1.5 divisor for missiles (`crit_cb.md` §2).
- **B.ow** PD 0x102AF060 (func 15) — 125-frame hpregen drain, stacks to 3 (`damage.md` §5). Vs players: ×8/6/4 % by
  difficulty (READ).

---------------------------------------------------------------------------------------------------------

## 4. Hit effects

### 4.1 Get-hit (hit recovery) D2Game 0x6FCFB910 — VERIFIED (mode 1, 20k)
Requested by result flag 0x04 (every melee hit unless state 54; missiles by their hit class). In post-hit:
state 21 `stunned` → get-hit **always** (threshold skipped); otherwise 0x6FCFB910 decides (returns 1 = no reaction):
```
if state 1 (frozen)                          → none
if poison(+0x28) ≠ 0 and poison == total     → none        (pure poison never interrupts)
if total < 256                               → none
M = max life (8.8), d = GH_DIV[hitclass]    // hth 16, 1hss 8, 1hsl 16, 2hss 32, 2hsl 64, 1ht 8, 2ht 16,
                                             // club 32, staf 16, bow 8, xbow 8, claw 16; other/flagged 16
if total < M/d                               → none
if total < 2M/d:  rng(defender) & 1 == 0     → none         (1/2 pass)
if total < 4M/d:  rng(defender) & 3 == 0     → none         (3/4 pass)
monster without MonStats2 mGH                → none
→ get-hit mode (player mode 4 GH, monster mode 3 GH) via 0x6FC98550 / 0x6FC95E10
```
So P(get-hit) = 0 below M/d, 3/8 in [M/d, 2M/d), 3/4 in [2M/d, 4M/d), 1 above. FHR only changes the animation
speed (part A speed code). Thorns/IM damage uses hit class 0x8D/0x4D → default divisor 16. Same rule for players
and monsters. Knockback (0x08) on a monster without MonStats2 mKB becomes get-hit (0x6FCFD00E).

### 4.2 Knockback (stat 81, event func 7) — VERIFIED (mode 7, 20k)
PD 0x102AEFE0: no knockback when the (missile owner / attacker) is a player **and** the defender is a player.
Stock 0x6FCCCE60: any positive stat 81 → chance by the defender's MonStats2 size: `small` 128/128, `large`
32/128, otherwise (and players) 64/128; `rng(attacker) & 0x7F < thr` → +4 |= 0x08. The stat value is only a switch.
Missiles use Missiles.txt `KnockBack` % instead (PD 0x1026E7D0).

### 4.3 Stun (dmg +0x44 from stat 66 / skills) PD 0x1026F190 — VERIFIED (mode 3, 20k)
```
Mind Blast (273) as the attacker's current skill with flag+0x28 = 1: len = 33 if the defender is in a PvP
level, else 50;   otherwise len < 1 → nothing
monster defender: unique (MonsterData+0x16 & 8) → 90 % resisted (pdRand % 100 < 90)
                  MonStats boss (#10064) → immune;  MonStats Velocity = 0 → immune
                  merc (#11104) and len > 12 → 13;  else min(len, 250)
players: min(len, 250); player→player: PvP cooldown check, Smite (97) capped at 63
expire = frame + len; existing stun state 21 → expire is REPLACED (can shorten); no refresh when the
attacker's owner and the defender are both players; new → state 21 list + timer
```

### 4.4 Chill D2Game 0x6FCFC780 and freeze 0x6FCFDAC0 (READ)
- Chill effect `e` = MonStats `coldeffect` for the difficulty (monsters), −50 for everything else. Nothing if e ≥ 0.
- Monster: `len = tdiv(len, MonsterColdDivisor)` (1/2/4); `len ≥ 1`. State 11 `cold` with velocity/attack/other
  rate −e; an existing chill is only extended (never shortened).
- Freeze (+0x34 > 0): players, MonStats bosses, uniques (MonsterData flag 8) and mercs are **never frozen** — the
  freeze length is applied as a **chill** instead. State 54 → nothing. Monster: `len = tdiv(len,
  MonsterFreezeDivisor)` (1/2/4; 0 divisor → error). State 1 `freeze`, extended only; monster regen timers are
  cleared and re-armed at frame + len + 1.
- Both lengths first went through the cold-resist rows (5/6) and the pre-step (§2.4).

### 4.5 Freeze-target proc (stat 134, event func 14) — VERIFIED (mode 6, 20k)
```
v = stat 134; aL = attacker level (−6 for domissiledamage), dL = defender level
x = 5·(4·max(v−1,0) − dL + aL + 10);  missile: x = tdiv(x, 3);  chance = clamp(x, 0, 100)
roll = rng(attacker) % 100; roll ≥ chance → nothing
len = clamp(2·(chance − roll) + 25, 25, 250) → a separate damage struct with freeze length only, applied through
execute(isMissile = 1): cold resist, pre-step, divisors, freeze→chill rules above
```

### 4.6 Slow (stat 150, event func 19) — VERIFIED (mode 8, 20k)
PD 0x102AFA40: none on players hit by player-owned attackers; none on prime evils (#10278).
Stock 0x6FCCD2A0: `v = stat 150` (any non-zero); cap 50 for players, champions/uniques (MonsterData & 0xC),
MonStats bosses and mercs; 75 for superuniques (flag 2); 90 otherwise. State 24 `slowed`, 750 frames, velocity
/ attack rate / other rate −v; replaces the previous slow. A negative value would speed the target up.

### 4.7 Poison PD 0x10271A70 — VERIFIED (mode 5, 20k)
```
needs len > 0 or rate > 0; monster defender: regen timer re-armed at frame+1
list = attacker is a player or player-owned ? the poison state (2) list OWNED BY THAT ATTACKER (0x10269230)
                                            : any poison list (#10871)
no list      → new state-2 list, expire = frame+len, hpregen(74) = −rate
list exists  → if −rate ≤ current hpregen (new rate ≥ old): expire = frame+len, hpregen = −rate
               else ignored (a weaker poison never extends a stronger one)
```
Poisons from **different players stack** (separate lists, stat 74 sums); the same player never stacks. Ticks:
monster regen timer 0x6FC96740 every frame `life += Σhpregen` (state 52 blocks positive regen); **life < 1 → 0**
(poison kills). One frame of poison is also in the hit total (+0x28 is in +0x4C).

### 4.8 Burn D2Game 0x6FCFC940 (READ)
Burn rate +0x14 and length +0x18 → state 115 `burning`, hpregen −rate; monster regen timer re-armed. Not
resisted (§2.7).

### 4.9 Other on-hit procs (READ)
| stat | func | rule |
|---|---|---|
| 112 howl | 8, 0x6FCCF2E0 | monsters only, not champions/uniques; chance `rng & 127 < v` (v/128); flee (0x6FCD2120, 20) |
| 113 stupidity | 9, 0x6FCCE0E0 | chance `clamp(5·(aL + 4v − dL + 6) [÷3 missile], 1, 99)`; blind level `clamp(tdiv(chance−roll,5)+1, 1, 20)` |
| 117 prevent heal | resolver post-hit 0x6FCFCE70 | state 52, not classes 704–709 |
| 198 / 195 / 201 | 20 / 20 / 21, 0x6FCCDDC0 / 0x6FCCDCE0 | one node per (stat, layer = skill/level); `rng % 100 < value`; 198 needs +0 & 0x20; 195 fires on every melee attempt (hit or miss) at impact; 201 when damaged |
| 196 / 197 / 453 | 30 | kill / killed events |
| 202 skill on block | 30 via PD timer | PD 0x1026EDD0 schedules `doblock` (event 15) 2 frames after a block (callback 0x102BECF0) |
| 200 skill on cast | 33, PD 0x102AFFF0 | PD event 14 `dospellcast` |
| 203 / 205 | 30 | events 16/17 are never raised (`strafe_procs.md`) |
| 424 heal after hit / 86 after kill | 28, 0x6FCCCB10 | `life = min(life + v<<8, max)`, SetStat directly (no 488 gate, not blocked by prevent-heal) |
| 138 mana after kill | 17 | players only |
| 139 heal after demon kill | 18 | target is a demon |
| 114 damage to mana | 13 | `mana += muldiv(stat114, total, 100)` capped (`defense.md` §1.4) |

> **Correction (audit):** event 7 is raised on every attempt, but "on attack" (195) only **fires on melee swings that hit**: func 20 refuses a struct without input flag 0x20, and the miss/block struct has +0 = 0. VERIFIED in `procs_cooldowns.md` §2.2 (`harness/procs.c` mode 11). Ranged attacks never fire it (event 8 is never raised).


Proc order: nodes on unit+0x90 in registration order (stat callback 0x6FCF9470; PD wraps the duplicate check
0x6FCF94F9 → 0x102C1120, special case stat 359). Each node rolls independently.

---------------------------------------------------------------------------------------------------------

## 5. Thorns, Iron Maiden and other reflect paths

### 5.1 Thorns / attacker takes damage (78, 128) — PD 0x102AEE30 (event func 6) — VERIFIED (mode 2, 20k)
Events: `attackedinmelee` (3) **and, new in PD2, `hitbymissile` (0)** (ItemStatCost `itemevent2`).
```
other is a missile: needs missile+0x98 and a damage struct; target = the missile's owner;
                    f = Missiles.txt DamageRate (+0x19C)
target must have unit flag 0x4; flag 0x80000000 targets whose owner is a player are exempt
v = owner's stat (78 or 128) for this node's layer; v ≤ 0 → nothing
f > 0: v = max((v·f) >> 10, 1)
target is a player → nothing                                   (PvP thorns disabled)
struct: +4 = 0x4021 (hit, get-hit request, 0x4000); 78 → physical v<<8, hit class 0x8D;
        128 → LIGHTNING v<<8, hit class 0x4D (stock put 128 into physical); +0x5C = owner
execute(target, attacker = owner, isMissile = 1): the target's resists/DR/absorb apply with the thorns owner's
pierce; +0 has no 0x20 → no events, so no chains; then PD post-hit (get-hit possible)
78 only: the thorns owner's Open Wounds (135) is rolled on the target (0x102AF060, event 5, stat 135 layer 0)
```
Timing: event 3 fires **at impact on every melee attempt — hit, miss or block** (0x6FCFEE62) and **also at swing
start** from the fill (0x6FCFD97B) for successful hits by players or mercs. Players are exempt, so only mercs
take melee thorns twice per connecting swing (see quirks).

### 5.2 Iron Maiden (stat 131 `thorns_percent`) — D2Game 0x6FCB8A70 — VERIFIED (mode 9, 20k)
```
s = defender stat 131; attacker is a player or merc (or NULL) → s = tdiv(s + 4, 8)
phys = the defender's physical damage after resist and the overkill clamp
r = phys > 0x100000 ? tdiv(phys,100)·s : s > 0x10000 ? tdiv(s,100)·phys : tdiv(phys·s, 100)
→ physical r, +4 = 0x4021, hit class 0x8D, execute on the attacker (its phys resist/DR apply), post-hit
```
PD2 moved it: the stock call in the melee execute is NOP'd; PD 0x1026EDD0 calls it when +4 & 0x20 = 0 and
+0 & 0x20, which also happens on **missile** hits (PD 0x1026E7D0 → 0x1026EDD0 with the missile owner).

### 5.3 Other return paths (READ)
- Life Tap (func 5 0x6FC6E760): heals the attacker by muldiv(damage, %, 100); the heal SetStat at 0x6FC6E8AD is
  hooked by PD 0x10268600 (stat 488 gate).
- Blood Golem (func 23, PD 0x102AFB70): owner heal with MonStats Drain. Clay Golem slow (func 27, PD 0x102AFF50):
  same PvP / prime-evil exclusions as item slow.
- Revive (func 32, new PD 0x102AFFC0) on `domeleedamage`. Skill on cast (func 33, PD 0x102AFFF0).
- Map mods: `map_mon_openwounds` 407 / `map_mon_crushingblow` 408 / `map_mon_splash` 427 / `map_mon_skillondeath`
  453 are ordinary event stats on monsters (funcs 15/16/20/30). `map_mon_phys_as_extra_*` 432–436 (Divide 3000)
  are converted at spawn by the map-mod applier PD 0x102DBA50 into elemental min/max stats (part D). No
  generic "reflect" map mod exists in this ItemStatCost.

---------------------------------------------------------------------------------------------------------

## 6. Leech, heals and clamps (cited, VERIFIED unless noted)
- **B.leech** (`damage.md` §6): `L64 = (s60<<6)/LifeStealDivisor`, `gain = trunc(Drain·muldiv(L64, phys, 100)/100/64)`,
  mana first; PvP levels → 0; **isMissile = 1 → both halved**. The PD wrapper at 0x6FCFE219 (0x102EE040) passes
  execute's own 3rd argument (`[esp+0x2C]` = isMissile, stack recounted: 5 saved regs + 1 push + ret + eax) to
  0x102700F0, which does `shr +0x38,1; shr +0x3C,1` only when it is non-zero. The melee attack frame calls execute
  with 0 (0x6FCFEDEA), so **normal melee leech is NOT halved**; halved are missiles (PD 0x1026E7D0 passes 1), PD
  area/splash hits (0x10271CB9 passes 1) and struct-only hits (thorns, IM, freeze proc — no leech fields anyway).
  The halving itself is VERIFIED (damage harness mode 4, missile flag 0/1); the argument wiring is READ. (missiles, and every PD area/splash hit applied
  through 0x10271BD0/0x10271CF0 with isMissile = 1 — READ). Drain = MonStats Drain(N/H); blank → 0.
- Monster attackers: flat life/mana/stamina drain (non-player branch of 0x6FCFBA40) — part D.
- **B.absorb_heal / leech heal** 0x6FCFB570: nothing if the unit is dead or has state 92 `death_delay`;
  `life = min(life + x, maxlife)`; the SetStat (0x6FCFB5B8) is PD 0x10268600: refused when old **and** new life
  are ≥ (100 − stat 488)% of max. This gate also applies to absorb healing and Life Tap (same SetStat).
- Monster regen tick 0x6FC97CB0 clamps to ≥ 1 HP and can overflow near 0x7FFFFFFF (`bug_boss_instadeath.md`);
  PD 0x102689B0 caps regen to 30 in PvP levels.

---------------------------------------------------------------------------------------------------------

## 7. Death and kill (READ)
- Execute sets +4 & 2 when life reaches 0 and fires events 10 (defender) and 9 (attacker) before the death
  animation.
- Post-hit 0x6FCFCF20: defender with state 54 and dying → flag 0x5C only. Monster dying → 0x6FCFEEE0 (monster
  death); player → mode DT (0x6FC98430). Mode DT start for monsters is the mode table row 0x6FD1A498, **patched
  to PD 0x102C11F0** (keys/sigils/maps, extra drops, Rathma/boss phase logic, then stock 0x6FC96270; `drops.md`,
  `bug_rathma_share.md`).
- **PD Sacrifice kill branch** (PD 0x1026EDD0 → timer 0x10270190): when the killing skill is 96 Sacrifice (or its
  missile), after the corpse animation (anim frames − 6, clamped 3–30) an area hit (0x102EC960, callback
  0x102703B0) is made. Its damage uses the skill's elemental min+max (<<7, ×Skills+0x154 %) plus the overkill
  `total − maxhp·stat352/128` (×Skills+0x158 %). Stat 352 is `last_sent_hp_pct`, only resent when the HP bar
  moves by more than 4/128 — a stale "life before the hit".

---------------------------------------------------------------------------------------------------------

## 8. PD2 hooks on this path (patch records in PD .rdata, module 3 = D2Game)

| D2Game site | → PD | effect |
|---|---|---|
| 0x6FCFD52C (+131 NOP) | 0x102ED930 → 0x10270E00 | crit/DS (VERIFIED) |
| 0x6FC5A730 (+16 NOP) | 0x102EEBD0 → 0x10270C50 | missile crit/DS |
| 0x6FCFC180 | 0x102ED040 → 0x1026D1D0 | damage percent (PvP/merc) |
| 0x6FCFC478 | 0x102EFA90 → 0x102713C0 | length pre-step (VERIFIED mode 4) |
| 0x6FCFC4EA | 0x102ED620 → 0x1026F410 | per-type resist/pierce/absorb, damage meter, Rathma share (VERIFIED) |
| 0x6FCFC88E | 0x102C0AD0 | inside chill (state creation helper) |
| 0x6FCFE219 | 0x102EE040 → 0x102700F0 | leech (VERIFIED) |
| 0x6FCFB5B8 | 0x10268600 | heal SetStat, stat 488 (VERIFIED) |
| 0x6FCFE233, 0x6FCFD13A, 0x6FCFD179 | 0x10253DE0 | umod on-hit dispatch with state 86 throttle |
| 0x6FCFE288 | 0x102ED5A0 → 0x1026F190 | stun (VERIFIED mode 3) |
| 0x6FCFE2AF | 0x10271A70 | poison (VERIFIED mode 5) |
| 0x6FCFEE85 (14 NOP) / 0x6FCFEE9D | 0x102ED340 → 0x1026EDD0 | Iron Maiden move + post-hit wrapper |
| 0x6FCB8B8A, 0x6FCCD792/83D/8ED/99D, 0x6FC5AFD9 | 0x102ED340 → 0x1026EDD0 | other post-hit calls through the wrapper |
| 0x6FC5AFF7, 0x6FCFE559 | 0x102EFA50 → 0x1026E7D0 | missile damage application |
| 0x6FC5AF05 | 0x102EEDE0 → 0x1026FB60 | missile block/avoid |
| 0x6FCFE661, 0x6FCFB7A0/7C2/846 | 0x102EEDE0 / 0x102EDAE0 → 0x1026FB60 / 0x1026FCF0 | melee block/avoid |
| 0x6FCFD41B | 0x102ED580 → 0x1026F370 | fill helper (part A) |
| 0x6FCFD746 | 0x102ED2A0 → 0x1026ED30 | fill helper (part A) |
| 0x6FCFE736, 0x6FCFE7BA/89D/95F/9DE, 0x6FCFEB8E | 0x102ED420 / 0x102EEB80 | old resolver (unreachable after 0x10270EB0) |
| 0x6FCF94F9, 0x6FCF9762, 0x6FCF9A41/5F, 0x6FCF9F66, 0x6FCF9527 | 0x102C1120 / 0x102C10E0 / 0x102C62A0 / 0x102C98A0 | stat-callback / event-node registration |
| 0x6FC6E8AD | 0x10268600 | Life Tap heal gate |
| 0x6FC97CBB | 0x102689B0 | PvP regen cap |
| 0x6FD1A498 (pointer) | 0x102C11F0 | monster death-mode start |
| event table 0x6FD277A8 (code PD 0x102BEA10) | funcs 6→0x102AEE30, 7→0x102AEFE0, 15→0x102AF060, 16→0x102AF610, 19→0x102AFA40, 22→0x102AFAB0, 23→0x102AFB70, 24→0x102AFC70, 25→0x102AFE90, 27→0x102AFF50, 32→0x102AFFC0, 33→0x102AFFF0 | event handlers |
| type table 0x6FD22AB0 (code PD 0x102BE9C0) | row 0 pierce 425, row 4 pierce 358 | phys/magic pierce |
| ItemStatCost (data) | 78/128 also `hitbymissile` | ranged thorns |

---------------------------------------------------------------------------------------------------------

## 9. PD2 vs stock 1.13c on this path (summary)
Crit/DS reworked (sum, cap 75, one of the two); negative resist halved for player-owned attackers; phys and magic
pierce; immune −res halved (stock ÷5); per-type clamp at 0 and double-precision resist; PvP DR/absorb rules;
leech halved for missiles and PD area hits; stat 488 heal gate; CB/OW rewritten (event 15/16); **thorns**:
ranged, lightning as lightning, players exempt, OW trigger; knockback and slow disabled in PvP, slow not on prime
evils; **stun** caps (uniques 90 % resist, bosses/immobile immune, merc 13 frames, Mind Blast fixed); **half
freeze ≥ 2 = unfreezable & unchillable**; **poison per player**; **Iron Maiden on missiles**; Sacrifice overkill
explosion; skill-on-block via timer; skill-on-cast event; Rathma share (bugged); damage meter.

---------------------------------------------------------------------------------------------------------

## 10. Discrepancies with our JS models
- `adv/engine/combat.js` — player → monster DPS: physical ignores monster DR (stat 34; `map_mon_normal_damage_reduction`
  400) and physical absorb; poison and burn: burn not modelled; leech ignores the overkill clamp. None of these
  are code bugs in the model's normal case (no DR on most monsters); not changed.
- `adv/re/damage.js` — matches the VERIFIED pieces used here (effectiveRes, applyResist, leech, CB, OW).
- `adv/re/defense.js` — the player-side get-hit rule is not modelled; §4.1 gives it (same rule as monsters).
- No JS was changed, so the existing tests are unaffected.

## 11. Confidence and open questions
- VERIFIED: §2 (earlier), §3 (earlier), §4.1, 4.2, 4.3, 4.5, 4.6, 4.7, 5.1, 5.2, 6 (earlier), pre-step.
- READ: execute order (§1), chill/freeze appliers, burn, events/proc order, Sacrifice branch, death path,
  missile application, the PD 0x1026EDD0 block/`0x4000` branches.
- Open: (1) which splash callers set +4 & 0x20 before 0x10271BD0 (decides whether splash fires normal
  `domissiledamage` events and the ×1.5 CB divisor); (2) PD 0x1026EDD0 sets 0x4000 for a player defender with
  unit+0x30 ≠ 0 and unit+0x40 = 22 on flags 0x8014 — meaning of those fields not identified; (3) exact meaning
  of the missile flag (#10623 bit 0) that sets +0 & 0x20; (4) PD 0x1026D1D0 non-100 cases (part D);
  (5) monster poison/burn sources and the burn fill arithmetic (part C/D).
