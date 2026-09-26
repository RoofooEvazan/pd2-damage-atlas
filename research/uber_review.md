# Uber bosses in PD2: data and code review

Scope: Lilith, Uber Duriel, Uber Izual, Uber Mephisto/Diablo/Baal (Uber Tristram), Diablo Clone (`uberdiablonew`, stale `diabloclone`), Rathma/Mendeln with clones, golems and totem, Lucion, uber Ancients, and trapped souls.

Sources:
- Game files: `excel_live/*.txt`, which is byte-identical to `data.zip` (`MonStats.txt` md5 f333a0a3…). The remaining tables come from the extracted `data.zip`.
- Code: stock `D2Game.dll` and `ProjectDiablo.dll` (PD, base 0x10000000).

No web or community information was used. Realm servers may run different server code; this review covers the client install only.

**Confidence**
- **VERIFIED**: executed natively in an earlier report.
- **READ**: read from the disassembly for this review.
- **DATA**: a table value.
- **PLAUSIBLE**: follows from READ facts, but a guard somewhere else could not be ruled out.

**Cross-references (not repeated here)**
- Rathma/Mendeln damage sharing, including poison and the uncleared slots: `adv/re/bug_rathma_share.md`.
- Instant-death / HP-overflow investigation: separate deep dive. See §H for the numbers that feed it.

**Shared mechanics** (sources: damage.md, FINDINGS.md, block_regen.md)
- **Crushing blow**: PD 0x102AF610. The divisor is 8 for a normal monster. For a MonStats `primeevil` it is 70+10·(100−hp%), or 30 if the monster is on PD's map-boss list (0x102C7C00). The map-boss list is six `std::set`s built at 0x10126E9A–0x1012728A; **it contains no uber**.
- **−resist on immunes**: a negative aura or curse contribution is halved when the monster's base resist is > 99 (PD 0x102C0540). None of the Hell ubers has a base resist ≥ 100; only the trapped souls (200) do.
- **Monster life regen**: stat 74 = (maxHP<<8)·DamageRegen >> 12 (D2Game 0x6FCD00B5–0x6FCD010F, READ). One point of DamageRegen is therefore 25/4096 ≈ **0.61 % of max life per second**.
- **Curse resistance**: stat 109, from MonProp `curse-res`, handled at PD 0x102BFFB2–0x102C0104 (READ).
  - Curse duration is multiplied by (100 − cr)/100, and the curse fails when the result is 0.
  - Stat 504 scales the curse's value in the same way.
  - The value is capped at 75 **only for players**; monsters are uncapped, so 100 means immune.
- **MonLvl**: rows stop at level 110. The 110 row is special: HP ×100, XP ×1600, TH 6532.
  - Levels 111 and 120 clamp to row 110 for stats, but the real level (111/120) is used for to-hit, XP penalty and item level.
- **Hell HP**: HP = MonStats HP(H) × MonLvl HP(H)/100.
  - Player-count bonus: the PD table gives +70 % per player from 2 players (FINDINGS).
  - Stock clamps the result to **0x7FFFFF points** before the <<8 shift (D2Game 0x6FCD005F `cmp esi,0x800000` / `mov esi,0x7fffff`, READ).

---

## Per-boss key numbers (Hell)

Notation: "HP 1p" is non-ladder. The L- column is identical at level 110. `ci` = curse-res from MonProp, `Dr` = Drain %, `DR` = DamageRegen.

| boss (hcIdx) | lvl | HP(H) base → 1p | res Ph/Ma/Fi/Li/Co/Po | Dr | DR | ci | flags | TC(H) / drop |
|---|---|---|---|---|---|---|---|---|
| Lilith `uberandariel` (707) | 110 | 6500–6600 → 650–660k | 50/50/90/90/90/90 | 33 | 1 | – | boss, prime | `Uber Andariel` + code drops Diablo's Horn `dhn` |
| Uber Duriel (708) | 110 | 6500–6600 → 650–660k | 50/50/90/90/90/95 | 100 | 1 | – | boss, prime | `Uber Duriel` + Baal's Eye `bey` |
| Uber Izual (706) | 110 | 6500–6600 → 650–660k | 50/50/90/90/90/95 | 50 | 1 | – | boss, prime | `Uber Izual` + Mephisto's Brain `mbr` |
| Uber Mephisto (704) | 120 (→row 110) | 5695 → 569,500 | 40/40/60/60/60/60 | **blank → 0** | 0 | – | boss, prime, **no `flying`** | `Uber Soul` (see F5) |
| Uber Diablo (705) | 120 | 6427 → 642,700 | 40/40/60/60/60/60 | 15 | 0 | – | boss, prime | `Uber Soul` |
| Uber Baal (709) | 120 | 6336 → 633,600 | 40/40/60/60/60/60 | 20 | 0 | – | boss, prime | `Uber Soul` |
| Diablo Clone `uberdiablonew` (789) | 110 | 10530 → 1,053,000 | 30/15/30/30/30/30 | 5 | 0 | 95 (+pois-len 50, 8 % fire absorb) | boss, prime | none; code drops Annihilus `cm1` (+1/200 bonus) |
| `diabloclone` (333, stale) | 110 | 6427 | 50/50/95/95/95/95 | 15 | 2 | – | boss, **not prime** | – |
| Rathma `rathmaBone` (933) | 110 | 6975 → 697,500 | 30/20/75/50/50/0 | 5 | 0 | 95 | boss, prime | none |
| Mendeln `rathmaPoison` (934) | 110 | 8550 → 855,000 | 30/20/65/40/40/0 | 5 | 0 | 95 | boss, prime | none |
| clones (935/936) | 110 | 3900 → 390,000 each | same as originals | 5 | 0 | 95 | boss, prime | none |
| Void Walker `rathmaVoidGolem` (937) | 100 | 400 → 23,720 | 30/20/30/30/30/50 | 100 | 3 | **none** | prime, **not boss** | Exp **0** |
| Sanguine Bearer `rathmaBloodGolem` (938) | 100 | 450 → 26,685 | 50/20/50/45/50/−20 | 10 | 3 | 100 | boss, prime | Exp 9000 → 4.29 M base |
| Conjured Soul `rathmaTotem` (939) | 100 | 400 → 23,720 | 50/20/75/75/50/20 | – | 0 | 100 | boss, prime | Exp 3000 |
| Talic / Madawc / Korlic (989–991) | **90** | 7560/6650/6020 → 383k/337k/305k (ladder L-HP: 511k/449k/407k) | 20/20/50/50/50/50 | **blank → 0** | 0 | – | boss, prime | code: `UberAncients` TC + N × `std` |
| Lucion (1112) | 110 | **24000 → 2,400,000** | 30/20/30/30/30/30 | 5 | 0 | 95 | boss, prime | chests `LucionChestLow/High` |
| trapped souls (790–794) | 110 | 50–70 → 5–7k | 200 all | 100 | 0 | 100 | boss, prime, killable | – |

### Lilith / Duriel / Izual (Pandemonium)
- **Portals** (PD 0x102BD370, cube output type 0x1C, Hell only, from town):
  - The first key sets game type `game+0x1DF4 = 0x28` and `game+0x2628 = 1`.
  - Each portal is picked at random: 133 → bit 2, 134 → bit 4, 135 → bit 8. Bit 0x10 is set once all three are open.
  - The pick is not uniform. `rand%3` falls through to the next free slot, so after the first portal the remaining two are not equally likely.
- **Death** (PD dispatcher 0x102C11F0, class switch 0x102C51D0/0x102C519C): 706 → `mbr`, 707 → `dhn`, 708 → `bey` (0x102C20A8/B4/C0). No difficulty or game-type check.
- **Regen**: DR 1 ≈ 0.61 %/s, about 4,000 life/s at 1p.
  - Stock D2Game 0x6FCFCEA4 exempts classes 704–709 from the `item_preventheal` (stat 117) path, so Prevent Monster Heal does not stop this regen.
  - The exemption does not cover PD2's newer ubers.
- **Lilith** uses Andariel's Hell poison, 33/33 over 225 frames, which is tiny at level 110. Her Normal column has El1MinD(N)=60 > El1MaxD(N)=33, inherited from Andariel.
- **Uber Izual**: Skill1 is the player skill `Frost Nova` at level 28 and Skill3 is `MonTeleport`, with Skill2 empty (stock-style row). Poison resist 95 vs Lilith's 90.
- **Uber Duriel**: A1TH(H) 110 → about 7.2k AR, a third of Lilith's (inherited from Duriel). Holy Freeze is level 1. aip1(H) is 24 vs 6 on base Duriel.

### Uber Tristram
**Minion spawning** is PD code in the AI wrappers, which then chain to the stock AI: Mephisto → D2Game 0x6FCA5B60, Diablo → 0x6FCC9610, Baal → 0x6FCD8610.

| boss | wrapper | chance per AI tick | cap (counted in own + adjacent rooms, alive only) | spawn list |
|---|---|---|---|---|
| Mephisto | PD 0x102B16A0 | rand%100 < 2·(60−n) | n(725–730) < 40 | skeleton8, sk_archer11, skmage_fire7/ltng7/cold6/pois7 (0x10380230 + 0x2D9/0x2DA) |
| Diablo | 0x102B1850 | 30 % | n(megademon6) < 15 | megademon6 (0x2C8) |
| Baal | 0x102B1A00 | 20 % | n(vampire9+wraith9) < 20 | vampire9, wraith9 (0x2DB/0x2DC) |

- The counter is PD 0x102B1520: switch on class−712 (table 0x102B1680/0x102B1670). Each cap counts the same classes that boss spawns, so the caps are consistent.
- Minions are not bosses, so their level comes from the area: Levels 185/136 MonLvl3Ex = **83** (Pandemonium 1–3 are 85).
- **Completion** (0x102C18DB / 0x102C1FF9): Mephisto sets `game+0x2628` |= 0x10, Diablo 0x20, Baal 0x40. When `(flags & 0x70) == 0x70`, the last kill drops `cm2` (Hellfire Torch) and `dcho`.
  - The portal handler sets game type 0x29 and clears `+0x2628` to 0 when the Tristram portal opens (0x102D4C65).
- **Levels**: 111 (Normal/NM) and 120 (Hell). Stats clamp to MonLvl row 110, so they match level-110 ubers in HP/AC/TH, but the attacker's hit chance and XP penalty use 120.
- **Uber Mephisto** has no Hell leech (Drain(N)/(H) blank, same as base Mephisto) and no `flying` flag (base Mephisto has it). AI delay is 6 in every column.

### Diablo Clone (`uberdiablonew`, 789)
- The arena is level 137 (Hell MonLvl3Ex 90). AI slot `game+0x2600` = self (PD 0x102B4F2C).
- The **trapped souls** (790–794) find their dclone through `game+0x2600` and must see class 0x315 there (PD 0x102B5A84). They pick the nearest player via a scan of the player hash (see F1).
- **Death** (0x102C190C): drops `cm1`. When `game+0x2629` (the portal item's stat 185 `uber_difficulty`) > 0, there is also a 1/200 bonus roll: `utb`/`uth`/`7rq`/`7bs`, or an extra spawn.
  - Kill credit (quest words +0x381/+0x3FD) is given only when the byte `game+0x2628 == 1`.
- **Static Field** (srvdofunc 160 → PD 0x102FC760) uses the target filter PD 0x1026F150, which **rejects 789, 933–936 and 1112**. Static therefore does nothing to the Clone, Rathma, Mendeln, both clones or Lucion. It still works on Lilith, Duriel, Izual, the Tristram three and the Ancients.
- `diabloclone` (333) is the stock Clone row. It is not `primeevil` (crushing blow 1/8), has Drain 15 and DamageRegen 2. Nothing in PD code spawns it.

### Rathma / Mendeln (Necropolis 161 → 162 → 163)
- **AI**: PD 0x102B2010, used for 933–936. It reads the MonStats aip columns as **a flat parameter array**: aip1 (0x56), aip1(H) (0x5A), aip2(N) (0x5E), aip3 (0x62), and others.
  - So "Normal-only" or "blank (H)" values such as Rathma aip1=650, aip1(N) blank, aip4(H) blank are **not difficulty copy-paste errors**.
  - AI delay = aidel(H), doubled before phase 6, minus 4·uber_difficulty, minimum 3 (0x102B25A9–0x102B25CF).
- **Phase byte** `game+0x262A`:
  - Mendeln's death opens 162 (→2, 0x102C1BCF).
  - Rathma's death opens 163 (→4, 0x102C1C3F).
  - The first clone death (5→6) casts 485 `RathmaDeath` on the surviving clone (0x102C1C86). This is an aura state `monfrenzy`: +20 % poison and cold mastery, +10 % magic mastery, about 2.8 h.
  - The second clone death (6→7) kills the golems, totems and knights, then drops `cwss`-type items.
- **MonProp**:
  - `rathma` has death-skill 485.
  - `mendeln` has death-skill **487 `NihlathakMapDeath`**, a client-only visual (cltdofunc 105, no srvdofunc). Mendeln's death buffs nobody (F12).
  - The clones share these MonProp rows.
- **Clone vs original**: HP(H) is 3900 vs 6975/8550 (intended weaker). The Rathma clone has `RathmaPrison` at level 1 vs 3 (F14). Everything else matches.
- **Damage share**: see `bug_rathma_share.md`. The 50 % split is at PD 0x1026F5E9 and open wounds at 0x102AF347.

### Uber Ancients (level 168)
- **Level 90** (the other ubers are 110). Talic and Korlic have A1TH(H) 20000 (always hit); Madawc has 1000.
- MonProp:
  - `ubertalic`: aura 479 `Mon Vigor` level 22.
  - `uberkorlic`: leap speed 20 and aura 532 `Holy Freeze Korlic`.
  - **Madawc has no MonProp** (F13).
  - None of the three has curse resistance.
- Korlic is untargetable while leaping (PD 0x102686F0/0x10268750: class 0x3DF with skill 143 `Leap Attack` flag 0x100).
- **Death** (0x102C2880, needs game type 0x2C and `+0x2628 & 0x110 == 0x10`):
  - Each death sets bit 0x20, 0x40 or 0x80. When the killed count ≥ the spawned count (bits 2/4/8), bit 0x100 is set.
  - It then searches the object table for class 0x250 (F1): if found, a `UberAncients` TC is dropped from it (0x10281BA0) plus N × `std`. It then opens the portal; kill credit is given only when N = 3.
- The Ancients' Hell leech is **0** (Drain(H) blank, while Normal/NM are 100) (F7).

### Lucion (levels 188/189)
- **HP**: 24000 × 100 = **2.4 M at 1p**. With the PD per-player table, the stock clamp at 8,388,607 points is reached at 5 players (2.4 M × 3.8 = 9.1 M). HP is capped from there, so 5–8 players all face the same HP (F6).
- The phase byte `game+0x262B` (0–3) scales two skill values of Lucion ×2 +3 at phase 2 and ×3 +6 at phase 3 (PD 0x10300141–0x1030016B).
- **Block against Lucion**:
  - Shield block ÷ 3 (PD 0x1026FBAE, in the shield roll 0x1026FB60).
  - Weapon block min(v, 75) ÷ 3 (PD 0x1026FD78, in 0x1026FCF0).
  - Neither applies to his spawns.
- **Spawns**: `LucionSpawnTank` has Exp(H) **130** while Spawn, Ranged and Spawner have 3000/3000/300 (F15). LucionControl has no HP (a controller).

### Trapped souls (790–794)
- Resist 200 to everything and curse-res 100. Base resist ≥ 100 means −res is halved, so they are effectively unkillable despite `killable=1`. This looks like intended "turret" behaviour.
- Soul 1 has A1 500–600 and A2 500–600; souls 2–5 have A1 30–60 (unused slot?) and A2 230–260.

---

## Findings, ranked

### A. Looks like a bug

**F1. PD2 uber code walks only the first unit of each hash bucket** — READ, High (it affects rewards and fight logic)
- **Stock walks the whole chain.** Game unit tables are `game+0x1120 + type·0x200`, 128 buckets with a `+0xE4` next link. Stock code follows the chain, e.g. D2Game 0x6FC4AFD7 `mov ecx,[ecx+0xe4]` and 0x6FC3E456.
- **These PD loops read only `[table+i*4]` for i < 128:**

| PD addr | table | purpose | failure when the unit is not first in its bucket |
|---|---|---|---|
| 0x102C293C–0x102C2968 | objects (+0x1520) | uber Ancients reward: find object 0x250 | **no `UberAncients` TC and no `std` drop**; the portal still opens (jumps to 0x102C2A01) |
| 0x102C1CA1–0x102C1D9F | monsters (+0x1320) | first clone death: cast 485 on the surviving clone | the survivor is not enraged |
| 0x102C1DC1–0x102C1E32 | monsters | second clone death: kill golems/totems/knights (0x3A9–0x3AB, 0x3C8/0x3C9/0x452/0x453) | leftovers survive the finished fight |
| 0x102B2781–0x102B288E, 0x102B2D2E–0x102B2D8E | monsters | Rathma AI: detect or clean up its golems and knights | undercounts, so extra summons and missed cleanup |
| 0x102B2653–0x102B26E8, 0x102B5AF2–0x102B5C60 | players (+0x1120) | Rathma / trapped-soul target search | a player is skipped only after more than 128 player units in the game (rare) |

```
102c2942: mov eax,[ecx]           ; bucket head only
102c294e: cmp dword [eax],2 / cmp dword [eax+4],0x250 / je found
102c295c: inc edx ; add ecx,4 ; cmp edx,0x80 ; jl 102c2942   ; no [eax+0xe4] walk
102c2968: jmp 102c2a01            ; not found → skip reward, open portal
```

- **How likely a miss is:** D2 prepends new units to their bucket. Any object created after the altar whose GUID ≡ altar GUID mod 128 (town portals, other objects in the game) hides the altar. The Necropolis Void holds 2000+ density monsters, so monster collisions there are near-certain.

**F2. Every uber event shares the same per-game scratch fields** — PLAUSIBLE, Medium
- The shared fields are `game+0x2600/+0x2604` (boss slots), `+0x2628` (flags or counter), `+0x2629` (uber_difficulty), `+0x262A` (Rathma phase), `+0x262B` (Rathma/Lucion phase) and `+0x1DF4` (game type).
- **Uses of `+0x2600`:**
  - Rathma AI (0x102B2571).
  - Diablo Clone AI (0x102B4F2C); the souls read it (0x102B5A84).
  - WarlordOfBlood (0x102B4489) and willowispboss (0x102C20CC).
  - The OW and damage share.
- **Uses of `+0x2628`:**
  - Pandemonium portal bits 1/2/4/8/0x10.
  - Tristram 0x10/0x20/0x40.
  - Ancients 0x10–0x100.
  - The Diablo Clone credit test `==1`.
  - nihlathakMap's phase counter (0x102B5F75…0x102B65FB).
- **The portal handler has no guard against this** (PD 0x102D48C2–0x102D4C7E). The only check is "if the game type is 0x28, only Tristram 0x88/0xB9 may be opened".
  - Opening the Clone, Rathma or Lucion portal (0x89/0xA1/0xBC) rewrites `+0x2628`, `+0x2629`, `+0x1DF4` and `+0x262A`=0.
  - Opening Tristram or the Ancients zeroes `+0x2628`.
- **Examples:**
  - Kill Uber Mephisto, then open a Clone portal in the same game: bit 0x10 is lost, so killing Diablo and Baal never reaches 0x70 and **the Torch does not drop**.
  - Tristram flags left in `+0x2628` make the Clone's `==1` credit test fail.
  - The souls go idle whenever another AI has written `+0x2600`.
- I did not find a server-side guard. If PD2 blocks multiple uber portals per game elsewhere (item restrictions or the realm), this becomes moot.

**F3. Rathma slots are never cleared when a clone dies** — READ (details in `bug_rathma_share.md`), Medium
- The survivor keeps taking only 50 % of each hit; the rest goes to the dead partner.
- The partner pointer can also dangle once the corpse unit is freed. The only guard is the state-8 check (#10494), and there is no alive or pointer-validity check (0x1026F603–0x1026F66E).

**F4. Diablo and Baal minions spawn next to the boss because their `aidist` is blank** — READ + DATA, Low
- The PD wrappers compute the spawn radius as clamp(`aidist[diff]` (record +0x52) − `[aiparam+0x14]`, 3, 20) (0x102B178D, 0x102B192D, 0x102B1AD0).
- Uber Mephisto aidist(H)=46 gives a radius of up to 20. Uber Diablo and Uber Baal have aidist blank = 0, so their radius is always 3 around the boss.

**F5. `Uber Soul` is not a treasure class** — DATA, Low–Medium
- Uber Mephisto/Diablo/Baal set TreasureClass1–3(H) = `Uber Soul`. No TreasureClassEx row has that name; it is the Misc.txt *item name* of `dcso`.
- The MonStats TC column is a link type (0x14 → link table 0x6FDF0890), so this name does not resolve. How D2 handles an unresolved TC link was not traced.
- In practice the Tristram rewards come only from PD code (Torch plus `dcho`). The data field is dead or misleading.

**F6. Lucion's HP hits the stock monster-HP clamp** — READ + DATA, Medium (balance)
- 24000 × MonLvl 100.00 = 2.4 M at 1p. The +70 %/player table gives ×3.8 at 5p, which is 9.1 M and over the clamp.
- D2Game 0x6FCD005F clamps the value to 0x7FFFFF (8,388,607) before the <<8 shift. Lucion's life therefore stops scaling at 5+ players.
- No other uber reaches the clamp: the Diablo Clone peaks at 6.2 M and Mendeln at 5.0 M at 8p.
- For the instant-death deep dive: the stock spawn path cannot overflow. Any overflow must come from a later life change, not from `0x6FCD0058`.

**F7. The uber Ancients have zero leech in Hell** — DATA, Medium
- Drain = 100 and Drain(N) = 100, but Drain(H) is blank on all three. Base ancients have Drain(H) 100.
- Blank reads as 0, which disables leech entirely (damage.md §6). This looks like an unfilled column. If "no leech" was intended, it is unlike the other PD2 ubers, which use 5.

**F8. The Rathma summons disagree with each other** — DATA, Low–Medium

| | Void Walker (937) | Sanguine Bearer (938) |
|---|---|---|
| `boss` | no | yes |
| MonProp | none (cursable) | `mendelnbloodgolem`, curse-res 100 |
| Drain(H) | 100 | 10 |
| Exp(H) | blank → 0 XP | 9000 → 9000 × 477.2 = **4.29 M base XP** |

- Because the Void Walker is `primeevil` but not a `boss`, `ignoretargetac` works on it, yet crushing blow uses the prime-evil divisor.
- The Blood Golem XP is surprising for a summon the AI can re-create, especially together with F1's undercount. The totem gives 3000 → 1.43 M.

**F9. Lucion spawn XP outlier** — DATA, Low
- `LucionSpawnTank` Exp(H) = 130 (base Exp 142, Exp(N) 130) at level 110 → 208k XP.
- Its siblings have 3000 (Spawn and Ranged) → 4.8 M, and the Spawner has 300. Looks like the NM value copied into Hell.

**F10. Uber Tristram minions are one area level lower than Pandemonium 1–3** — DATA, Low
- Levels 136 and 185 have MonLvl3/MonLvl3Ex(H) = 83/83. Pandemonium 1–3 have 83/**85**. The spawned minions are not bosses, so they get the area level of 83.

### B. Intended, but surprising

**F11. Static Field is excluded for exactly six classes** — READ
- Filter PD 0x1026F150:

```
cmp eax,0x315 / 0x3a6 / 0x3a8 / 0x3a5 / 0x3a7 / 0x458 → return 0; else stock 0x6FC62EC0
```

- It still works on Lilith, Duriel, Izual, the Tristram three, the Ancients and every uber minion.

**F12. Mendeln's death skill does nothing on the server** — DATA
- MonProp `mendeln` death-skill 487 `NihlathakMapDeath` has no srvdofunc. Rathma's 485 enrages nearby allies.
- In the two-at-once clone phase, PD code applies 485 to the survivor either way, so only the MonProp death skill is asymmetric.

**F13. The three Ancients are set up differently** — DATA
- Madawc has no MonProp. Talic has the Vigor aura and Korlic has Holy Freeze and leap speed.
- Crit is blank on all three (the other ubers have 5).
- Their level is 90, versus 110 for the other ubers.

**F14. The Rathma clone's Bone Prison is weaker than the original's** — DATA, Low
- `rathmaBoneClone` Skill3 `RathmaPrison` Sk3lvl = 1; `rathmaBone` has 3. Every other skill level matches.

**F15. Curse resistance differs by uber generation** — DATA
- Rathma, Mendeln, the clones, the Diablo Clone and Lucion have 95 (curses last 5 % of their duration). Their minions have 100.
- Lilith, Duriel, Izual, the Tristram three and the Ancients have none, so full curse duration.

**F16. No leech on Uber Mephisto, little on the new ubers** — DATA
- Uber Mephisto Drain(H) blank → 0, as on base Mephisto.
- The Diablo Clone, Rathma and Lucion have 5. Uber Diablo has 15, Baal 20, Lilith 33, Izual 50, Duriel 100.

**F17. Ladder life bonus skips level-110 ubers** — DATA
- The MonLvl L- (ladder/online) columns give about +33 % HP at levels ≤ 109 (e.g. 90: 5068 vs 6757).
- Row 110 has HP = L-HP = 10000.
- On ladder or online, the Ancients (90) and the Rathma summons (100) get +33 % HP while every level 110/111/120 uber does not.

**F18. Prevent Monster Heal only affects PD2's newer ubers** — READ
- Stock exempts 704–709 from `item_preventheal` (D2Game 0x6FCFCEA4 `cmp eax,0x2c0 / jl / cmp eax,0x2c5 / jle skip`).
- The regenerating ubers (Lilith, Duriel, Izual with DR 1) are inside that range. PD2's newer ubers are outside it but have DR 0.

**F19. `diabloclone` (333) is a stale row** — DATA
- It is not `primeevil` (crushing blow 1/8), has Drain 15, DamageRegen 2, and resists 50/50/95×4.
- Nothing in PD spawns it: PD code refers only to 789. If some stock path ever spawns it, crushing blow is 9× stronger than on the real Clone.

**F20. Pandemonium portal order is biased** — READ
- In PD 0x102BD386–0x102BD3DB, `rand%3` falls through to bit 8 (Furnace, 135) when the rolled slot is taken.
- Example: after Matron's Den opens first, Furnace comes next with probability 2/3.

**F21. Cosmetic: name strings** — DATA
- Uber Baal uses NameStr `Baal Crab` → "Baal", with no orange colour code.
- Lilith's string has no colour code, while the other ubers use `ÿc4`.

---

## Evidence index (PD = ProjectDiablo.dll, stock = D2Game.dll)

- **Static Field filter**: PD 0x1026F150. It is pushed at 0x102FC836 in srvdofunc 160 = PD 0x102FC760, via table `0x104E3114` = D2Game 0x6FD274A8 (lazy table 0x103D321C → rva 0x1074A8).
- **Tristram**:
  - minion counter 0x102B1520;
  - Mephisto / Diablo / Baal wrappers 0x102B16A0 / 0x102B1850 / 0x102B1A00, installed at 0x102BAB39 / 0x102BAB43 / 0x102BABF7;
  - spawn lists 0x104E2CC0 / 0x104E2C78 / 0x104E2DA4, initialised at 0x10117680 / 0x101176F0 / 0x10117740.
- **Death dispatcher**: 0x102C11F0, with the class switch at 0x102C18ED (idx 0x102C51D0, targets 0x102C519C).
  - 704: 0x102C18DB. 705/709: 0x102C1FF9. 706–708: 0x102C20A8–C0. 789: 0x102C190C. 933: 0x102C1C3F. 934: 0x102C1BCF. 935/936: 0x102C1C86. Ancients: 0x102C2880.
- **Portal handler**: 0x102D48C2–0x102D4C7E. Game types: 0x28 Pandemonium, 0x29 Tristram, 0x2A Diablo Clone, 0x2B Rathma, 0x2C Ancients, 0x2D Lucion.
- **Rathma AI**: 0x102B2010 (slot write 0x102B2560).
- **Diablo Clone and souls AI**: 0x102B4F00…0x102B5C80.
- **Lucion**: block 0x1026FBAE and 0x1026FD78; phase damage 0x10300141.
- **Crushing-blow map-boss sets**: 0x102C7C10, contents built at 0x10126E9A–0x1012728A (none are ubers).
- **Curse resistance**: 0x102BFFB2–0x102C0104.
- **Stock monster HP**: 0x6FCD002B–0x6FCD0080 (clamp), regen 0x6FCD00B5–0x6FCD0117.

## Not done / open
- None of the code in this review was executed natively; it is all READ. The Rathma share is VERIFIED in the separate report.
- Not traced:
  - how D2 resolves the unresolved `Uber Soul` TC link;
  - whether the stock AIs that the Tristram wrappers chain to also spawn minions (no direct caller of the stock uber-minion counter 0x6FCD0C40 was found);
  - the level of skill-summoned Rathma minions.
- Lucion's summon and phase mechanics, and the Clone's phase logic, were only skimmed.
