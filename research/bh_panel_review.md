# In-game Advanced Stats panel (BH.dll) vs the game's real math vs our page

The PD2 in-game Advanced Stats panel is drawn by BH.dll, function `0x1007C040`. This review checks every line it shows.
- Each line is compared with what the game server actually does: D2Game, D2Common and ProjectDiablo.dll.
- It is also compared with our Advanced Stats page, whose copy of the panel is `adv/engine/engine.js` `panel()`. The page's other cards come from `combat.js` / `damage.js`.

**Evidence labels**
- **TESTED**: the game's own code was run on thousands of random inputs and matched.
- **READ**: read from the disassembly.
- **DATA**: from PD2's .txt tables or AnimData.

## Summary

- **Most panel lines are raw stat totals.** They show exactly what you have, and they are fine.
- **Where the panel calculates something, it has 6 mistakes.** They are in crushing blow, critical strike, crit multiplier, open wounds, hit-recovery breakpoints, and the cold/poison/curse length and pierce lines.
- **Our page is more correct overall**, because its combat numbers follow the server code.
- **Our page is not free of these mistakes.** In a few places it copies the panel's formulas and inherits them. There are 5 issues to fix.

## 1. Where the in-game panel is clearly wrong

### 1.1 Crushing blow leaves out Two-Hand Mastery (READ, server side TESTED)
- **What BH does** (`0x10079670`): it assumes the mastery's crushing blow and efficiency are already inside your crushing-blow stat. It adds nothing when your weapon qualifies. It *subtracts* the mastery when the "other hand empty" rule fails.
- **What the game does:** masteries store their stats on a separate layer keyed by item type (D2Common `0x6FDA25DF`: SetStat layer = passiveitype). The plain stat read (`#10973`, exact layer 0) never sees them. The server adds them back only for a qualifying weapon, through PD `0x102D30B0`. For `2han` that means a two-handed melee weapon with the other hand empty.

| Barbarian, Two-Hand Mastery 20, no CB gear | Game | BH panel |
|---|---|---|
| Two-handed sword, other hand empty | 29% CB, 125% efficiency | 0%, 100% |
| Two-handed sword held one-handed with a shield | 0% | **−29%, 75%** |

### 1.2 Critical strike leaves out Sword, Mace and Spear Mastery (READ)
- **What BH does:** crit = Claw Mastery + One-Hand Mastery + Throwing Mastery + Critical Strike (337) + item crit (258), capped at 75.
- **What the game does** (PD `0x10270E00`): crit = weapon-matched `passive_mastery_melee_crit` (344), or 347 for throw skills, + 337 + 258, capped at 75.
- **Consequences:**
  - A Barbarian with Sword Mastery 20 and a sword really has 29% crit. The panel shows 0%. Mace and Spear Mastery are missed the same way.
  - Throwing Mastery crit is shown for melee attacks. The game only applies it to throw skills with a throwable weapon.

### 1.3 Crit multiplier leaves out Javelin and Spear Mastery (READ)
- **BH:** 200 + item stat 256.
- **Game:** 200 + stat 256 + the weapon-matched mastery entry. Javelin and Spear Mastery puts 256 on the `spea` layer.
- **Result:** an Amazon with a spear sees 200% when the game uses more.

### 1.4 Open wounds damage counts deep wounds about 2× (server TESTED)
- **BH:** `((owBase(clvl) + 25) × 25 >> 8) + stat 501`. This treats deep wounds as flat damage per second.
- **Game** (PD `0x102AF060`): per-frame drain `(owBase(clvl) + 25 + 5 × stat501) / 256`, over 25 frames per second. So each point of deep wounds is worth **125/256 ≈ 0.49 per second**.
- **Example:** level 90 with 200 deep wounds: the panel shows **465/s**; the game deals **363/s**, before the monster's physical resist and pierce. It is also ÷4 against player-owned targets. Up to 3 stacks add together, and the panel doesn't show that.

### 1.5 Hit-recovery breakpoints: right lists, wrong choice for some weapons (DATA)
BH's lists are static tables built at `0x10013B50`. Every list matches the frame counts computed from PD2's AnimData. The mistake is which list the panel picks, which is decided by weapon family at `0x1007D1B6`:

| Class and weapon | Game (AnimData) | BH shows |
|---|---|---|
| Paladin with a polearm, 2H axe, maul or war scythe (all use the staff animation) | fast list 3/7/13/20/32/48/75/129/280 | normal list 7/15/27/48/86/200 |
| Druid with a dagger, throwing knife, javelin, 2H sword, bow, crossbow or no weapon | normal list 5/10/16/26/39/56/86/152/377 | fast one-hand swing list 3/7/13/19/29/42/63/99/174/456 |

The panel is right for Paladins with spears and staves, and for Druids with one-handed swinging weapons.

### 1.6 Smaller errors

| Line | BH | Game | Example |
|---|---|---|---|
| Cold length | half-freeze / cannot-be-frozen only | chill and freeze length are also reduced by cold resist, including the Hell penalty and max-resist cap (PD 0x1026F410 rows 5/6, READ) | 75% cold res: BH 100%, real 25% |
| Poison length | 100 − PLR − penalty, no floor | 100 − min(PLR + penalty, 75), so the floor is 25% (READ) | 200 PLR in Hell: BH 0%, real 25% |
| Curse length | max(100 − stat109, 25) with an unsigned compare | stat 109 capped at 75 for players, so the floor is 25% (PD 0x102BFE20, READ) | more than 100 curse length reduction shows a negative % |
| Projectile pierce | stat 156 + stat 166, layer 0 only | 166 also gets weapon-matched layered entries (PD 0x102724DD → 0x102D30B0, READ) | Throwing Mastery's pierce is missing while holding a throwing weapon |

## 2. Where our page is wrong

| # | Where | Problem | Fix |
|---|---|---|---|
| 1 | Offense tab, "Critical hits" card | Crit chance comes from `panel()`, so it has error 1.2. The damage-against-a-monster card uses the right formula, so the page contradicts itself. | Use `combat.js` crit (TP(344/347) + T0(337) + T0(258)). |
| 2 | Offense tab and BH-panel tab, open wounds damage | Copies error 1.4. | Use `damage.js openWounds` per second. |
| 3 | BH-panel tab, cold and poison length | Copies both errors in 1.6. | Apply the cold-resist reduction and the 25% poison floor, or label the line "BH shows". |
| 4 | BH-panel tab, projectile pierce | Uses T(166), which counts Throwing Mastery pierce with *any* weapon. That's the opposite of BH's mistake, and also wrong. | T0(166) + TP(166) + T(156). |
| 5 | BH-panel tab, crushing blow and crit multiplier | These show the game's real values, not what BH shows. That's more useful, but the tab is labelled as a copy of the panel. | Label them "real value (BH shows X)". |

**Label note:** the Overview "Damage" card shows the in-game character screen's number. That screen adds flat "+damage" (stat 111) from the weapon only, after the % bonus. The server adds it from all items, before the % bonus. The damage-against-a-monster card already uses the server rule. The formula is faithful to the screen, but the card should say "character screen".

## 3. Correct in both

| Line | Rule (source) |
|---|---|
| Resistances | penalty 0/−40/−100 in expansion; max = min(75 + max-res stat, 90). PD2 hard cap 90 (defense.md, TESTED) |
| Curse resistance (stat 504) | capped at 75 for players (PD 0x102BFE20) |
| Base AR | ToHitFactor + 5·(dex − 7) + stat 19 = D2Common #10621 (TESTED) |
| Base defense | dex/4 + stat 31 = #10672 (TESTED) |
| Base damage | stats 21–24 |
| Deadly strike | chance capped at min(75 + stat210, 100), multiplier 150 + stat 257 (the game rolls it only if crit misses) |
| Added elemental damage | item min/max × (1 + mastery/100), with magic using 357: matches D2Game 0x6FCFCD80 (TESTED). Item poison ignores poison_count (326), which is essentially never on a player. |
| Cast-rate breakpoints | all class lists, plus the Chain Lightning / Frozen Orb list, match AnimData |
| Attack-speed breakpoints | match the game (bh_ias.md). Our Dragon Tail error was fixed earlier. |
| Raw stat lines | FCR, FHR, FBR, FRW, IAS, attack rate, thorns, masteries, elemental pierce, per-kill, MF, GF |

## 4. Shown raw in both, which can mislead

- **Life and mana leech:** the game divides by 1/2/3 by difficulty (÷3 in Hell). It halves leech for missiles and scales it by the monster's Drain.
- **Damage reduction % and absorb %:** anything above 50% (DR) or 40% (absorb) does nothing.
- **Magic find:** shown raw. The effective values for uniques, sets and rares are lower (250/500/600 diminishing returns).
- **Deadly strike:** shown as its own chance. It only rolls when crit fails.

## 5. Not checked

- The panel's **XP %** line. BH computes it from its own level × act/difficulty table at 0x1011DE28. It was not compared with the server's experience formula.

> **Correction (audit):** this has since been checked in `experience.md` §6 (verdict: the level factor is one number per act/difficulty; 2–12× too low in Hell Acts 1–2 at clvl 80–90 and 0.00 % in maps and Uber zones).


## Addresses

- **BH panel:** `0x1007C040`
- **BH mastery helper:** `0x10079670`
- **BH breakpoint tables:** FHR `0x10013B50` → map `0x1014D4F4`; FCR `0x100142E0` → map `0x1014D528`
- **BH weapon family switch:** `0x10080E90`
- **Server:** stat layer set D2Common `0x6FDA25DF`; GetUnitStat #10973 `0x6FD88B70`; layered-stat sum PD `0x102D30B0`; crit PD `0x10270E00`; crushing blow PD `0x102AF610`; open wounds PD `0x102AF060`; resist rows PD `0x1026F410`; curses PD `0x102BFE20`; missile pierce PD `0x102724CB`.

## Update: our page fixed
- The BH-panel tab now copies the in-game panel's own math exactly, including its mistakes: crushing blow without Two-Hand Mastery (negative with the other hand occupied), the crit multiplier without Javelin and Spear Mastery, pierce from layer 0 only, and the unsigned curse-length compare.
- Wherever the game uses a different number, it now appears next to the panel's in green as "game: …". The code is `engine.js gameValues()`.
- **Offense tab:** crit, crushing blow (plus the Smite bonus when you have Smite), open wounds (including the 3-stack maximum) and projectile pierce (as a fixed pierce count, see `pierce_ow.md`) now use the game's values.
- **Against-a-monster:** a new "Players in game" control feeds the monster's life and the crushing blow divisor. Whirlwind now uses its fixed hit rate (`whirlwind.md`).
- **Overview:** the "Damage" card is labelled "character screen".
