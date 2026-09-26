# Character screen: attack rating, defense, damage, hit chance

Sources are the stock 1.13c D2Client, D2Common and D2Game, plus ProjectDiablo.dll (PD) patch records from `adv/pd2_patch_records.json`. The JS versions are in `adv/re/charscreen.js`.

**Status key**
- **VERIFIED**: the game code was run natively and matched the JS on every case.
- **READ**: read from the disassembly but not executed.

The native test is `harness/charscreen.c` with `charscreen_driver.js`; `charscreen_build.py` builds `D2Client.img`. It loads the stock D2Client image at 0x6FAB0000 alongside the real D2Common. It then calls the character-screen functions directly and compares them with `charscreen.js`.
- Result: **30,000 random cases, 0 mismatches** for the physical damage, elemental/poison, attack rating and defense functions.
- Real D2Common code used: #10973/#10910 stat getter, #10621 base AR, #10672 defense, #10858 skill txt.
- Stubbed data sources, which return the case's own values: weapon lookup, grip, mastery, Str/DexBonus, item type, skill ToHit and the colour helpers.

## Where the screen gets its numbers (D2Client)
- **Panel draw:** 0x6FB6CEA0.
- **Skill boxes:** 0x6FADFD30(unit, skill, slot) is called for the left skill (#10828) and the right skill (#10507).
  - It dispatches on `SkillDesc.descdam` (+0x12) through table 0x6FBA5210.
  - It dispatches on `SkillDesc.descatt` (+0x14) through table 0x6FBA52A0.
  - The column offsets come from the SkillDesc loader descriptor at D2Common 0x6FDB2131.
- **"Attack" skill:** descdam 1, descatt 2. "Throw" is descdam 3, descatt 3.
- **AR tooltip value:** 0x6FADE350(unit, left/right) goes through the same descatt table.
  - 0x6FB6CB90 turns that AR into the "chance to hit" tooltip against the last monster. It uses the same ratio formula as the server, clamped to 5–95.
- **Defense line:** 0x6FB6D97B, which is plain D2Common #10672(unit).
- **Mercenary panel:** 0x6FB3EA20 is a separate function. Its damage adds the merc's Str (Dex for class 271) as a percent, and PD sets its resist cap to 90.

### PD2 patches in these paths (D2Client)
| site | effect |
|---|---|
| +0x2C348, +0x2C56E, +0x30A10, +0x3136F, merc +0x8ECB3 | the mastery call (#10804) is redirected to **PD 0x102728B0** |
| 0x6FAE313B (161 bytes) → PD 0x102ED140 → 0x102F91B0 | throw damage (descdam 3/4/22, 0x6FAE2F30) is replaced |
| 0x6FAE0393 → PD 0x102ED1E0 | descdam 15 only; not used by Attack |

- **PD 0x102728B0(unit, weapon, skill, mode):**
  - mode 0 (AR) reads stat 342 `passive_mastery_melee_th`.
  - mode 1 (damage) reads 343 `passive_mastery_melee_dmg`.
  - mode 2 (crit) reads 344.
  - It switches to 345/346/347 (`passive_mastery_throw_*`) when the weapon is throwable (PD 0x102773B0) and the skill is a throw skill (skill record +0x18 test via 0x10273F50).
  - The value is the sum of that stat's entries whose param (an item type) the weapon matches (PD 0x102D30B0). Item types 114 `2han` and 116 `1han` get special handling there.
- **Throw damage (PD 0x102F91B0):** computed in floating point. `min = trunc(min × (100 + T17 + P)/100)`, `max = trunc(max × (100 + T17 + X + P)/100)`, with P = str/dex bonus + T25 + skill% + mastery. The replaced stock code did this in two steps.
  - The throw formula is **READ only**. The stack mapping of X (caller arg3) is uncertain, so the throw path is not in the JS.
- No PD patch touches #10621, #10672, 0x6FAE1220, 0x6FAE0E30 or 0x6FADC4F0 (apart from the mastery call).

## 1. Attack rating (VERIFIED: 0x6FADC4F0 + #10621)
```js
base = T(19) + 5*T(2) - 35 + cls.ToHitFactor          // D2Common #10621 0x6FD81EA0 (dex-7 + 4*dex-28); ToHitFactor = CharStats +0x3C
if (weapon is item type 38 'tpot') AR = 0              // #11088 check
pct  = mastery.ar + T(119) + skillToHit (+ T(325) progressive_tohit if the skill has the progressive flag)
AR   = base + trunc(base*pct/100)
```
- **skillToHit** = D2Common #10653 (0x6FD9EA50): `ToHit + (lvl-1)*LevToHit`, or `ToHitCalc` when that is set. It is 0 when lvl ≤ 0. Plain Attack gives 0.
- **Paths:** Attack uses descatt 2 → 0x6FAE0DB0 → 0x6FADC4F0 with a single weapon. Dual wield goes through 0x6FADD500, which detaches the other weapon with #10164 and computes each hand in turn.
- **Throw (descatt 3, 0x6FADC2D0):** the formula is the same, but mastery counts only while the unit has state 78.
- **Not in the screen value:** target-only bonuses (see §4).

## 2. Defense (VERIFIED: #10672 0x6FD82890, called with real code)
```js
base = T(31) + trunc(T(2)/4)
pct  = T(16) + T(171) (+ Holy Shield calc while state 101: 0x6FDA1BF0 on the state's skill)
def  = base > 0 ? base + trunc(base*pct/100) : base - trunc(base*pct/100)
if (T(182)) def += trunc(def*T(182)/100)               // armor_override_percent
```
- The screen shows exactly this number.
- The server hit roll uses this plus the target's stat 33 (melee) or 32 (missile).
- **Rounding:** each step truncates toward zero, including dex/4 for negative values.

## 3. Weapon damage (VERIFIED: 0x6FAE1220 + 0x6FAE0E30)
Attack goes descdam 1 → 0x6FAE5470 → descdam 7 0x6FAE4C00 → 0x6FAE3240. That call runs physical damage (0x6FAE1220), then elemental (0x6FAE0E30).
```js
// physical, 0x6FAE1220(ecx=unit, eax=skillPct; &min,&max,&color, flat, skill, srcOverride, weapon, flag)
[mn,mx] = weapon ? (grip==2 ? [T(23),T(24)] : [T(21),T(22)]) : [T(21)+1, T(22)+2]
if (src != 128) { mn = trunc(mn*src/128); mx = trunc(mx*src/128) }  // descdam 7 passes 128 here
if (src != 0)   { mn = max(mn,1); mx = max(mx,2) }  else mn = mx = 0
pct = skillPct + T(25) + mastery.dmg
      + (weapon ? trunc(T(0)*StrBonus/100) + trunc(T(2)*DexBonus/100) : T(0))
pct = max(pct, -90)
min = trunc((100 + T(18) + pct) * mn / 100) + weapon.T(111) + flat
max = trunc((100 + T(17) + pct) * mx / 100) + weapon.T(111) + flat
// elemental, 0x6FAE0E30, table 0x6FB869B4
for [lo,hi,mast] of [[48,49,329],[50,51,330],[54,55,331],[52,53,none]]:
   a=T(lo); b=T(hi); if (T(mast)) { a += trunc(a*T(mast)/100); b += trunc(b*T(mast)/100) }
   a = min(a,b); min += a; max += b
if (T(58)) { pm=T(332) applied the same way to T(57)/T(58);
   len = T(101) > 0 ? T(101) : trunc(T(59)/max(T(326),1));
   min += (len*T(57))>>8; max += (len*T(58))>>8 }
if (max > 0) { min = max(min,1); max = max(max, min+1) }
// back in 0x6FAE4C00: total scaled by Skills.SrcDam (+0x1A5) when != 128, then + skill damage >>8 (#10567/#10297/#10121/#11091)
```
**Two-handed grip** is decided by D2Common #10051 (0x6FD6FB80), which returns 2 in two cases:
- the only item in the hands is `2handed`;
- a Barbarian (class 4) holds a `1or2handed` item with the other hand empty.

With a shield or a second weapon, or when the rule above doesn't apply, the 1-hand stats 21/22 are used. Bows and crossbows only have 2-hand damage.

**Flat damage:**
- **+min/+max from anywhere** (stats 21–24 on rings, boots, jewels…) is already inside T(21..24). It is therefore multiplied by the full percent.
- **Stat 111 `item_normaldamage`** is read from the **weapon unit only** (#10910(weapon,111)). It is added after the percent. Stat 111 on other items is ignored by the screen.
- **Unarmed** uses 1–2 base plus Str as the percent (StrBonus effectively 100).

**Kick** (descdam 2, 0x6FAE0AE0) shows one number: `(Skills.MinDam << (HitShift-8)) + T(137)`.

### How ED counts: the op-13 rule (READ, D2Common stat lists)
Stats 16/17/18 have ItemStatCost `op 13`. On an item's stat list (owner type 4), op 13 adds `trunc(base × pct/100)` to the target stat. The base comes from the item's **base** list via 0x6FD88C20, so it covers weapon base damage and armor base defense.

In the list update 0x6FD89CE0, after each target is recomputed, the op switch at 0x6FD89DC8 decides whether the percent stat itself is kept:
- For op 13 (case 0x6FD89E5F): if the list's owner type (+0x08) is 4 and the recomputed target is **nonzero**, the percent stat itself is **not stored** in the item's full stats.
- The owner's sum (0x6FD88CD0) reads a child item's full stats. A percent that was not stored therefore never reaches the player.

Consequences:
- **The weapon's own ED is used once.** It is in the weapon's 21–24 and never in the player's T(17)/T(18). This includes jewels socketed in the weapon, because the weapon's base damage is nonzero when they merge.
- **Armor ED (16) never reaches the player's T(16).** It lives in the armor's 31.
- **Off-weapon ED reaches T(17)/T(18)** and multiplies the whole base damage, for example Fortitude, Phoenix, Steelrend or rings. This holds for items whose 21–24 are 0 when their ED is applied.
- **Order-dependent case:** a non-weapon item that carries both ED and +min/+max damage, such as ED/max jewels in a helm.
  - Whether its ED is kept depends on whether its 21/22 were already nonzero when the ED arrived. Stats are merged in id order: 17/18 come before 21/22.
  - Typically the first jewel's ED passes and later changes are dropped.
  - Not emulated. Treat as uncertain.
- **Damage-related stats of a deactivated item list are skipped** (0x6FD88D34). Examples: the inactive hand while dual wielding, attached and detached with #10164 → #10734/#10077 setting list flag 0x40000000. The skipped stats are those with ItemStatCost `damagerelated`: 17, 18, 21, 22, 25, 111….

**engine.js** adds every item stat to the player, so its T(16/17/18) double-counts. `charscreen.fromEngine(C)` subtracts two things:
- 17/18 of weapons that have base damage;
- 16 of items with base defense.

Without that correction, the displayed damage would apply weapon ED twice.

## 4. Melee hit chance (VERIFIED earlier: D2Game 0x6FCFDE90, 200k cases, see FINDINGS.md)
```js
function hitChance(ar, alvl, def, dlvl) {
  if (def < 0) { ar -= def; def = 0 } if (ar < 0) { def -= ar; ar = 0 }
  const ratio = ar+def === 0 ? 100 : trunc(ar*100/(ar+def));
  return clamp(trunc(ratio*alvl*2/(alvl+dlvl)), 5, 95);
}
```
Server-side AR against a target (READ for the mastery and 179 terms):
- **Base:** start from #10621, then add +T(123) against a demon or +T(124) against undead.
- **Percent:** `pct = mastery (stock #10804, stat 342 melee) + T(119) + skill ToHit + T(179 matching montype)`, then `AR += trunc(AR*pct/100)`.
- **Target defense:** #10672 + stat 33.
  - T(115) sets defense to 0 against normal and champion monsters. It does not apply to uniques, superuniques, act bosses or mercs.
  - T(116) is halved against players, bosses, superuniques and mercs, capped at 100, then `def -= trunc(def*v/100)`.
- **Differences from the screen:** the character screen's AR leaves out 123/124/179 and the 115/116 effects.
- In JS this is `meleeHitChance(ctx, mon, {serverMastery, attackVsMontype})`.

## ctx for charscreen.js
`{ T, clvl, cls:{ToHitFactor}, weapon:{StrBonus,DexBonus,'2handed','1or2handed',isMissilePotion}|null, weaponT, twoHandGrip | (otherHand,isBarbarian), mastery:{ar,dmg}, skill:{toHit,edPct,flat,srcDam,minDmg,maxDmg}, holyShieldPct }`

- **T:** must be the game's player totals, meaning after the op-13 rule above.
- **fromEngine(C, {mastery, skill}):** builds ctx from `engine.compute()`.
- **Not handled:** dual wield. Build one ctx per hand, with the other weapon's damage-related stats removed.
