# Block chance, block speed, life/mana/stamina regeneration (PD2 = 1.13c + ProjectDiablo.dll)

All formulas come from the disassembly. Code: `adv/re/block_regen.js`. Native tests: `harness/regen.c` + `regen.py`, and `harness/fbr.c` + `fbr.py`.
Status: **VERIFIED** means the real DLL code ran in the harness and matched the JS formula on every case. **READ** means it was only read from the disassembly.
`tdiv` is C integer division (truncates toward zero). Stats 7/9/11 (max life/mana/stamina) and 6/8/10 (current values) are 8.8 fixed point (×256).
The server runs 25 frames per second.

## 1. Chance to block

### Base value: D2Common #10212 = 0x6FD81D20 (VERIFIED, 40,000 cases, 0 mismatches)
`int __stdcall(unit, bExpansion)`:
- **Player** (type 0):
  - Returns 0 unless #10854 (0x6FD6FF40) finds a shield-type item in a hand slot.
    - That means ItemTypes 0x33 and its subtypes, and the item must not be broken (item flag 0x100 or 0x4000 clear).
  - b = stat20 `toblock` (unit total) + CharStats `BlockFactor` (+0x49, byte).
  - If `bExpansion`: b = tdiv((dex − 15) × b, 2 × max(1, clvl)), using stat 2 and stat 12 totals.
  - Cap: b ≥ 75 → 75. There is no lower clamp, so a result ≤ 0 means no block. Classic games skip the dex/level scaling.
- **Monster** (type 1): toblock only (MonStats flag / 0x6FD81680 gate).
- **Shield's own block value**: the item's stat 20 = Armor.txt `block` (ItemsTxt +0x111, byte). It is set when the item is created (D2Common 0x6FD7ADBC, D2Game 0x6FC311AB). Class bonuses come only from BlockFactor.
  - PD2 BlockFactor: Ama 25, Sor 20, Nec 20, Pal 30, Bar 25, Dru 20, Asn 25.
- **Skill sources** reach the formula through stat 20 on the unit:
  - Holy Shield (aurastat `toblock` = dm56)
  - Chilling Armor (5+blvl)
  - Weapon Block and Holy Sword give stat 348 instead (see below)
- **PD2 changes to #10212**: only the ItemStatCost record size, 0x144 → 0x150 (code-init immediates at 0x6FD81DB2 and 0x6FD81E09). The formula is unchanged.

### Character screen
- Stock: D2Client 0x6FB6C7FE, the defense-box hover: `#10212(player, expansionFlag)`. When that is < 1, it shows weapon block instead (0x6FB6C430), but only with two claws (wclass 13).
- PD2 redraws this hover in 0x1021BC80 (call at 0x1021BD17) with the same logic and the strings `ClientDefenseBlockHover*`.
- **The panel shows the unmodified #10212 value.** It does not show any moving or attacker-specific reduction.

### The roll when hit
- Stock D2Game 0x6FCFB790 is called from melee 0x6FCFE660 and missiles 0x6FC5AF04, only after a successful hit:
  - chance = #10212(defender, game+0x70 = expansion).
  - If the defender is a player, #10143 says it is moving (mode 2, 3 or 6), and mode ≠ 2 (walk): chance = tdiv(chance, 3). Running and town-walking give 1/3; walking does not.
  - roll = LCG(defender seed) % 100. Blocked if roll < chance. Otherwise it goes to avoid/evade 0x6FCFB030.
- **PD2 replaces both calls** (patch records 0x6FCFE661 and 0x6FC5AF05 → 0x102EEDE0 → **0x1026FB60**):
  - chance = #10212(defender, game+0x70). PD resolves the ordinal lazily through 0x10274510, and its table entry is −10212.
  - **The moving/running 1/3 penalty is removed.** There is no mode check at all.
  - chance = tdiv(chance, 3) when the attacker is monster class 1112 (`wraithMapMod`).
  - Blocked if (RNG % 100) < chance, with RNG 0x102C5D10 (PD2's own generator: same output formula, different next-state; see hit_pd2.md). The roll uses the defender's seed. VERIFIED natively later (harness/hitpd.c).
  - Result flag 1: the caller clears the hit and sets block mode.
- Now VERIFIED: harness/hitpd.c runs 0x1026FB60/0x1026FCF0 natively with the imports stubbed through PD's lazy-import slots (8,000 cases, 0 mismatches).

### Weapon block (stat 348 `passive_weaponblock`)
- Tried as the first step of the avoid chain, which runs after a failed shield roll or when the shield chance is < 1 (stock 0x6FCFB030, PD2 0x1026FCF0).
- Value: 0x6FCFA540 / PD 0x1026FC00 takes the largest stat-348 value whose param (item type) matches the item in body location 4 or 5. A param of 0 is used directly.
- Stat 348 sources: Assassin Weapon Block (dm12 + toblock/5) and, in PD2, Paladin Holy Sword (dm56 + toblock/5).
- **PD2**:
  - cap 75
  - /3 against wraithMapMod
  - requires wclass 5 (2hs) or 13 (ht2, two claws)
  - halved in PvP maps (levels 157, 159, 166)
  - success returns flag 0x10
  - A player who dodges, avoids, evades or weapon-blocks gets a 4-frame lockout (playerdata+0x1C0 = frame+4).
- **Stock**: two claws only, no cap, no halving.
- READ.

## 2. Faster block rate (block animation speed)

### D2Common 0x6FD83110, block branch (VERIFIED, 50,000 cases, 0 mismatches, harness/fbr.c)
- Block mode is player mode 9 (BL) or monster mode 6 (0x6FD7EF90).
- base = 50, or 100 while the unit has state 101 `holyshield` (checked for unit types 0, 1, 3 when the states table has more than 101 entries).
- EFBR = tdiv(120·fbr, 120+fbr). This is 0x6FD823E0 with table 0x6FDE4608 row {1, 120, stat 102}.
  - PD2 routes this helper to 0x10267DA0. That copy uses the same stock table (PD resolves D2Common+0x94608) and only fixes the ISC record size.
- rate = floor((base + EFBR) × animSpeed / 100), unsigned, clamped to 1..0x7FFF.
  - **Block has no 175 cap**, unlike cast (min(100+EFCR, 175) at 0x6FD831A1). Your `min(base+EFBR,175)` never binds in practice, because EFBR < 120 and a Holy Shield base of 100 + 86 still gives the same frame counts, but the game code has no such cap.
- Frames: ceil(frames·256 / rate) − 1. This reproduces the known FBR tables exactly:

| AnimData key (PD2) | frames, speed | base 50: fbr → frames | Holy Shield (100) |
|---|---|---|---|
| AMBL1HS | 3, 88 | 0:17 4:16 6:15 11:14 15:13 23:12 29:11 40:10 56:9 80:8 120:7 200:6 | – |
| AMBLHTH/1HT, AIBL*, PABL(HTH/1HS/1HT/2HT) | 3, 256 | 0:5 13:4 32:3 86:2 | 0:2 86:1 |
| PABL2HS | 3, 168 | 0:9 3:8 9:7 19:6 35:5 65:4 142:3 | 0:4 18:3 95:2 |
| BABL* | 4, 256 | 0:7 9:6 20:5 42:4 86:3 280:2 | – |
| SOBL* | 5, 256 | 0:9 7:8 15:7 27:6 48:5 86:4 200:3 | – |
| NEBL*, DZBL* | 6, 256 | 0:11 6:10 13:9 20:8 32:7 52:6 86:5 174:4 | – |

- FBR sources: items (stat 102), Weapon Block (+lvl FBR), Shiver Armor (10+2·blvl).
- The animation key uses the weapon class (FINDINGS: 0x6FD93860).

## 3. Life regeneration

### D2Game 0x6FC97CB0 (VERIFIED, 30,000 cases)
- Called every frame by the unit timer 0x6FC99B10. That timer re-arms itself for game frame+1 and skips dead units (player mode 0/17, monster 0/12).
- If stat74 ≠ 0: life = clamp(life + stat74, 256, maxlife). Negative hpregen cannot kill; it stops at 1 life.
- **life/s = stat74 × 25 / 256**, so +N replenish life ≈ 0.0977·N per second. There is no base life regen.
- **PD2** (patch 0x6FC97CBB → 0x102689B0): stat 74 is read through a wrapper. In PvP maps (levels 157, 159, 166) a value > 30 becomes the `bloodwarp` state's own hpregen, or 30. READ.

## 4. Mana regeneration

### D2Game 0x6FC97950 (VERIFIED, 40,000 cases)
- Runs every frame. No PD2 patches.
- `perFrame` is in 1/256 mana:
  ```
  if (!state85 'nomanaregen') {
      d = CharStats.ManaRegen(+0x3A, byte) * 25  (0 -> 7500)
      e = max(1, tdiv(maxmanaFP, d))
      e = tdiv(e * (stat27 manarecoverybonus + 100), 100)   // 0x6FC214D0 muldiv
  } else e = 0
  e += stat26 (manarecovery, raw 1/256 per frame)
  e = clamp(e, -mana, maxmana - mana)
  ```
- **mana/s = 25·e/256 ≈ maxMana / ManaRegen × (1 + mrb/100)**.
- PD2 CharStats ManaRegen = 120 for every class, so a full bar takes 120 s at 0% bonus. Vanilla uses 300, which does not fit the byte field and wraps to 44, but PD2 is 120.

## 5. Stamina (optional)

### Regen: D2Game 0x6FC97A50 (VERIFIED, 30,000 cases)
- If stamina < max: a = maxstaminaFP >> sh; a += tdiv(a·stat28, 100); stamina = min(stamina + a, max).
- sh by player mode:
  - NU/TN: 8, a full bar in 10.24 s
  - WL/TW: 9, 20.48 s. When walking (WL), stamina must be ≥ 1 point.
  - Every other mode (run, get-hit, attacks, casts, block…): no regen unless stat28 ≥ 1000, then sh = 8.

### Drain: D2Game 0x6FC97BB0 (VERIFIED without armor, 10,000 cases)
- Runs in run mode and not in town (#10331 / #10057).
- d = RunDrain × 2
- d ×= tdiv(bodyArmor.speed, 10) + 1 (ItemsTxt +0xD8, READ)
- d −= tdiv(d·stat154, 100)
- d = max(d, 1)
- stamina −= d per frame. At 0 the stamina is set to 0.

## Caveats
- The shield-block roll (0x1026FB60), weapon block, the PvP rules and the hpregen wrapper are PD2 code. They were READ but not executed natively, because they depend on PD's lazy-import resolver.
- The stock paths they replace were read. #10212 and the anim-rate block branch that they call were executed.
- `T(348)` in engine.js sums every param. The game takes the largest value whose item-type param matches a held item.
- Holy Shield's toblock only counts if the engine applies the state's aurastats to stat 20.
- The PD2 client tooltip shows weapon block only with two claws (wclass 13), even though the server also allows 2hs (Holy Sword).
- As with the rest of this project, realm servers might run different server code. This only covers the client install.
