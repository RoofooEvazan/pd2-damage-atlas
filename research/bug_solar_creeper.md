# Bug: Solar Creeper sometimes eats a corpse but gives no mana

Scope: PD2 client install (1.13c D2Game/D2Common plus ProjectDiablo.dll, base 0x10000000), with data from `data.zip`/`excel_live`.
Status keys follow FINDINGS.md:
- **VERIFIED** means the real code was executed in the harness.
- **READ** means the result comes from disassembly or data only.
- **INFERRED** is a reasoned link that the code does not show directly.

Caveat: PD2 realm servers may run different server code. This report covers only the shipped DLLs.

## 0. Which rows are involved (the prompt had them swapped)

| In-game name | Skills.txt row | summon (MonStats) | Eating skill (MonStats Skill1) | What it grants |
|---|---|---|---|---|
| **Solar Creeper** | `Vines` Id 241 | `vinecreature` (hcIdx 427) | **VineCycler** Id 325 | calc3 = **8 (mana)** |
| Carrion Vine | `Cycle of Life` Id 231 | `cycleoflife` (426) | CorpseCycler Id 307 | calc3 = 6 (life) |
| Poison Creeper | `Plague Poppy` Id 222 | `plaguepoppy` (425) | — (does not eat) | — |

- Both eaters use MonStats `AI = CycleOfLife`, which is MonAI id 111.
- Both have aip1 = 35, aip2 = 20, aip3 = 15, aip4 = 10, aip5 = 35 and aidel 8.
- VineCycler and CorpseCycler both have:
  - `srvstfunc 67`, `cltstfunc 49`, `TargetCorpse 1`
  - `aurafilter 73731` (= 0x12003), `aurarangecalc 40`
  - calc1 = min, calc2 = max, calc3 = the stat to heal.

## 1. Server code path

### 1a. AI (MonAI 111): D2Game table 0x6FD2E788 + 16·ai → {type 2, param 0x6FCC3980, **AI 0x6FCCB070**}
The table is indexed by MonStats +0x1E via 0x6FCBD13B. Offsets below are MonStats offsets verified with the column table (`aip1` +0x56 … `aip5` +0x6E).

`0x6FCCB070(ecx=game, edx=vine, arg=AI ctx)` works as follows (READ):
1. **Owner lookup.** MonsterData(+0x14)+0x28 → ctrl, then `0x6FD003A0(game=ctrl+0x28, type=ctrl+0x30, guid=ctrl+0x2C)`.
   - If the owner is not found, the AI does nothing (0x6FCCB0C5–0x6FCCB0D0). So a missing owner never produces an eat.
2. **Leash.** `#10522(vine, owner)` is the edge-to-edge distance max + min/2 minus half sizes (D2Common 0x6FDCFBD0).
   - When it is ≥ aip5 (35), the AI calls `0x6FC3CAE0(... mode 3 ...)`:
     ```
     6fccb10d: cmp eax,ecx            ; dist >= aip5(35)?
     6fccb11f: call 0x6fc3cae0        ; mode 3 = warp next to the owner
     6fccb124: test eax,eax
     6fccb126: jne 0x6fccb34c         ; return only if the warp succeeded
     ```
   - Mode 3 (0x6FC3D17E) finds a free spot next to the owner with the random spot finder 0x6FC67450, then teleports there (0x6FD019C0).
   - **If no spot is found it returns 0 (0x6FC3D38E) and the AI falls through to the eat logic while still ≥ 35 away.**
3. **Corpse search.** `0x6FC2C1F0(game, radius)`. The radius is VineCycler `aurarangecalc` (40), clamped to 5..50 at 0x6FCCB1B0.
   - It lists units around the vine (#10591 / #10052, flags 0x1002).
   - It returns the **first** unit that passes `#10965` (D2Common 0x6FD811B0) and `0x6FD00A60` (hostile to the vine's owner).
   - #10965 requires a monster in mode 12 (DD), MonStats2 `corpseSel` = 1 and `deadCol` = 0, and no state with the `udead` flag.
     - The udead mask is DataTables+0x150, using mask base +0xCC and flag bit 0x21. This mapping was checked against the `meleeonly` → wolf/bear mask.
     - udead states: freeze, dopplezon, revive, redeemed, corpse_nodraw, shatter, **corpse_noselect (118)**, shadowwarrior, attached, recycled, restinpeace, corpseexp, uberminion, dopplezonnew, black.
   - **The corpse's owner fields (unit +0x94 type / +0x98 GUID) are not checked here.**
4. **Corpse distance.** If the corpse's #10522 distance is ≥ aip2 (20), the corpse is dropped (0x6FCCB1ED).
5. **Melee range.** `#11138(vine, corpse, 0)`, which PD2 redirects at 0x6FCCB200 to **PD 0x10270FF0**.
   - For player-owned pets PD adds half the vine's size plus the corpse's #10143 value, so the vine may eat from slightly further away.
6. **"Owner already full" gate.**
   - For class 426 it compares life (stat 6) with max life; for class 427 it compares **mana (stat 8) with max mana (#10186)** (0x6FCCB225–0x6FCCB26A).
   - Stock code sets `ok = 0` when the owner is full.
   - **PD2 static patch record: D2Game rva 0xAB26E (0x6FCCB26E) := 0x01**, byte-verified. The instruction `c7 44 24 1c 00 00 00 00` becomes `mov [esp+1c],1`.
   - **So the vine now eats even when the Druid's mana is full.**
7. If a corpse is found, in melee range, and the cooldown aip1 (35 frames) since ctrl+0x18 has passed:
   - `0x6FD12380(skill = MonStats Skill1, mode 8 = S1, target = corpse)` builds a command.
   - `0x6FC95E10` sets the path target to the corpse (#10336) and **calls the mode handler at once** (0x6FC949A0 / 0x6FC95F5C). The skill start therefore runs in the same AI tick.
   - Otherwise the vine walks to the corpse (0x6FD12540) or idles for aip3 frames.

### 1b. Skill start: srvstfunc 67 = **PD 0x10300CA0**
- PD writes it at init 0x102BE6C6: `[srvstfunc table 0x6FD27338]+0x10C`.
- The stock table has 0 in slot 67.
- The stock cycler (srvstfunc 63, D2Game 0x6FC2D170) used a different design:
  - It resolved the **owner** (0x6FC41260) and failed if the owner was missing.
  - It then fired the `recycler delay` missile. The missile's hit healed the owner.
- PD2 replaced that design with an **area heal around the vine**.

Walkthrough of `0x10300CA0(ecx=game, edx=vine, skillId, lvl)` (READ, and VERIFIED in §3):
```
10300cb8: cmp [ebx],1                 ; vine must be a monster
10300d02: call 0x102bfa90             ; target = path target (0x6FD007A0 refresh, #10392), must not be the vine
10300d14: push 0x76 / call #10494     ; corpse already has corpse_noselect(118)?  -> return 0
10300d24: cmp [esi+0x98],0 ; je ok    ; corpse owner GUID == 0 -> ok
10300d2c: cmp [esi+0x94],0 ; je FAIL  ; owner GUID != 0 and owner TYPE == 0 (player) -> return 0
10300d3d: call #10945(corpse,0x76,1)  ; mark the corpse consumed (corpse_noselect)
10300d54: range = calc(aurarangecalc +0x64)   ; = 40
10300d68: min = calc1, max = calc2, stat = calc3 (+0x138/+0x13C/+0x140), all evaluated WITH THE VINE as the calc unit
10300d98: stat must be 6 or 8 else return 0
10300dad: amount = min + RNG(vine seed, max-min)      ; D2Game 0x6FC211D0, range [min, max)
10300dd7: 0x102EC960 -> D2Game 0x6FCC0C70(game, vine, x=0,y=0, range, aurafilter 0x12003, cb 0x102BFBF0, &{amount,stat})
10300de9: 0x102ED170 -> D2Game via ptr 0x104E5104 (visual "recycler" missile / skill event on the corpse)
          return 1
```

### 1c. Who receives the mana: D2Game 0x6FCC0C70, filter 0x6FCC0370, callback PD 0x102BFBF0
- **Centre and rooms.** The centre is the vine's position. Only units in the **vine's room plus that room's near-list** are scanned: `#10331` GetRoom, then `#10383` room+0 / room+0x24.
  - Filter bit 0x2000 skips town rooms (#10057).
  - The per-room rectangle test 0x6FCBE610 is effectively always true.
- **Distance.** Squared centre distance ≤ range² (#10769 compared with `range*range` at 0x6FCC0E07). This is Euclidean on subtile coordinates.
- **Caster excluded.** The caster (the vine) is skipped.
- **Filter 0x12003:**
  - players (bit 1): mode 0 (death) and 0x11 (dead) are rejected
  - monsters (bit 2): mode 12 and 0 are rejected
  - **0x10000 ally**: `0x6FD00570(vine, unit)` resolves the vine to its owner through MonsterData+0x28 and compares that with the unit or the unit's party.
- **Callback 0x102BFBF0.**
  - `max = stat(9 or 7)`, `new = min(amount<<8 + stat(8 or 6), max)`, then `#10887 SetStat`.
  - It does not send a packet directly. The player's mana syncs through the normal stat update.
  - Every allied unit in range is healed, including party players and the Druid's other pets.
- **The Druid is never looked up directly.** The Druid gets mana only if at that moment it is:
  - inside the vine's room set,
  - within 40 subtiles (centre to centre) of the vine,
  - alive, and allied.

### 1d. Client
- cltstfunc 49 is D2Client 0x6FB75C70 (stock; PD2 overrides only cltstfunc 54–58).
- It only spawns the `cltmissilea`/`cltmissileb` visuals ("vine beast attack", "recycler delay").
- The client does not predict mana. The eat animation comes from the monster's skill/mode packet, which is sent whether or not srvstfunc 67 grants anything (INFERRED from the dispatch order: mode handler first, then 0x6FCBFD00 → srvstfunc).

## 2. Every way an "eat" can happen without mana

| # | Condition | What the player sees | Where | Status |
|---|---|---|---|---|
| A | **Druid outside the vine's room + near-rooms**, even if only 5 subtiles away | Eat succeeds, the corpse is consumed (corpse_noselect set), the visual missile appears, **no mana** | 0x6FCC0C70 room loop | **VERIFIED** |
| B | **Druid more than 40 subtiles (centre) from the vine** | Same as A | 0x6FCC0E07 | **VERIFIED** (40 gets mana, 41 and (29,29)=41.0 do not, (28,28)=39.6 does) |
| C | **Leash warp failed.** The vine is ≥35 (edge metric) from the Druid, the free-spot search next to the owner failed, and the AI goes on to eat a corpse near itself | Eat far from the Druid, no mana (falls into A/B) | AI 0x6FCCB124 → 0x6FC3D1DD / 0x6FC3D1F8 / 0x6FC3D20F → 0x6FC3D38E returns 0 | READ |
| D | **Corpse has owner type 0 (player) and GUID ≠ 0**, e.g. **Desecrate corpses**: PD srvdofunc 161 (Skill `Desecrate` 83) spawns level monsters in DD mode and writes `[+0x94]=0, [+0x98]=caster GUID` (PD 0x102FCAC4) | Eat animation plays (the AI's finder accepts the corpse) but st67 returns 0: **no mana and the corpse is not consumed**, so the vine targets the same corpse again every aip1 = 35 frames until it despawns (PD timer +400 frames) | finder 0x6FC2C1F0 does not check owner fields; st67 0x10300D24 does | st67 side **VERIFIED**; finder side READ |
| E | Corpse already has corpse_noselect | st67 returns 0 | 0x10300D14 | VERIFIED. Cannot come from the vine's own AI in the same tick, because the finder already excludes udead states and the start is synchronous (READ) |
| F | Druid at or near full mana | Eat happens, mana is clamped to max (495/500 → 500; 500 → 500) and the corpse is wasted | PD patch 0x6FCCB26E disables the stock "owner full → don't eat" gate | **VERIFIED** (clamp) and byte-verified (patch) |
| G | Druid dead (mode 0 / 0x11) | Eat succeeds, no mana | filter 0x6FCC0398 / 0x6FCC03A9 | VERIFIED |
| — | Integer math | Not a cause: amount = min + rand(max−min) ≥ calc1 = 20+4·(lvl−1) > 0. Over 300 runs with min 20 / max 40 the amounts fell in [20,39] (observed 21–38) | 0x10300DAD | VERIFIED |
| — | calc3 (stat) | Fixed at 8, so it passes the 6/8 check | — | READ |

## 3. Harness verification (real PD2 + D2Game + D2Common code)
Source: `/tmp/claude-0/sc/vine.c` and the driver `/tmp/claude-0/sc/vq.py` (scratch). Run with cwd `harness/`.

**Executed natively:**
- PD 0x10300CA0 (st67), 0x102BFA90, 0x102EC960 and callback 0x102BFBF0
- D2Game 0x6FCC0C70, 0x6FCC0370, 0x6FD00570, 0x6FD003A0, 0x6FCBE610 and RNG 0x6FC211D0
- D2Common #10331, #10383, #10057, #10561, #10750, #10867, #10769, #10392 and #10973

**Stubbed:**
- #10494, which returns a per-test corpse_noselect flag
- #10945, #10158 and #10786, the calc evaluator, which returns 40 / min / max / stat
- #10887 SetStat, which writes into the fake stat list
- the target refresh 0x6FD007A0
- the final missile call

**Results** (mana 100/500, VineCycler min 20 / max 40):
```
same room, 10 away                   ret=1  100 -> 136   corpse consumed=1
near full (495/500)                  ret=1  495 -> 500   (clamped)
adjacent (near-list) room, 30 away   ret=1  100 -> 135
NON-adjacent room, 30 away           ret=1  100 -> 100   corpse consumed=1, missile=1, setstat=0   <-- eaten, no mana
NON-adjacent room, 5 away            ret=1  100 -> 100   corpse consumed=1, setstat=0               <-- eaten, no mana
dist 40 / 41                         136 / no mana
(29,29)=41.0 / (28,28)=39.6          no mana / +37
corpse owner type 0, guid 7          ret=0  no mana, corpse NOT consumed, no missile
corpse owner type 1 (monster)        ret=1  +31
corpse already corpse_noselect       ret=0  no mana
druid mode 0 / 0x11                  ret=1  no mana (corpse consumed)
```

## 4. Most likely cause, ranked

1. **(Top) PD2's srvstfunc 67 grants mana only by area around the vine: radius 40, vine's room set only. Nothing ties the grant to the owner, while the AI decides when to eat using its own, different distances.** Confidence: mechanism **VERIFIED**; link to T4 **INFERRED**.
   - Code facts:
     - In the stock design (srvstfunc 63) the owner was resolved and healed by a missile.
     - The PD version heals "allies within 40 of the vine in its rooms" (§1c).
     - The AI eats whenever a corpse is within 20 of the vine. It only checks the owner's distance through the leash at 35, and that leash does nothing when the warp fails (§1a step 2).
   - A stranded vine (warp failed) therefore eats at ≥35 from the Druid. It gives mana only if the Druid is still ≤40 centre-distance away and in a near-room. As the Druid moves on, every eat is wasted, and each one consumes the corpse.
   - The room restriction (A) can also drop the Druid at short distances when rooms are small or irregular.
   - Why T4 makes it more frequent (INFERRED from data, not executed):
     - The t4m maps are Mesa, Realm of Terror/Pit of Despair (164/165), Sanctuary of Sin (171) and Kanemith (186/187).
     - Most of them are `IsInside=1` presets with LvlPrest `Outdoors=0` (164, 165, 186, 187). Most T1–T3 maps are `Outdoors=1`.
     - They have the highest `MonDen(H)` = 2475, and t3m/t4m affixes add +23…+30 % `map-glob-density`.
     - Dense, walled terrain makes the random free-spot search next to the Druid (0x6FC67450) fail more often, and teleporting Druids leave vines behind walls. Both lead to C → B/A.
     - Indoor preset room partitioning may also make near-lists shorter. **Not verified:** the preset room splitter was not reversed.
2. **Mismatch between the AI corpse finder and st67's owner check (D).**
   - Corpses whose unit owner is a player (type 0, GUID ≠ 0), notably PD2 **Desecrate** corpses, are picked by the stock finder. PD's st67 then rejects them.
   - The result is a repeated eat animation with no mana, and the vine keeps choosing that corpse because it is never consumed. The finder returns the first match.
   - Confidence: st67 **VERIFIED**; finder READ. It only happens with a Desecrate caster (or other player-owned hostile corpses) nearby, so the T4 link is weak (party composition).
3. **PD2's static patch at 0x6FCCB26E removes the "owner mana full" gate (F).**
   - The vine eats at full or near-full mana: the gain is clamped to 0 or near 0, and nearby corpses are used up, so fewer are left when mana is actually low.
   - Confidence: VERIFIED for the byte patch and the clamp. It explains "ate but got nothing" at high mana on any map.
4. Low or ruled out:
   - A same-tick corpse_noselect race (E): the start is synchronous (READ).
   - Owner lookup failure: the AI does nothing without an owner.
   - Integer truncation.
   - Client prediction: none exists.
   - Map affixes: the t4m affixes change monster/player stats and density only. None touches corpses, states 118/104, or owner fields (Properties `map-*` → stats 369–449).

Side finding (READ): VineCycler calc1/calc2 add `skill('Cycle of Life'.blvl)`. st67 evaluates calcs with the **vine** as the unit (0x10300D53: `push ebx`). `vinecreature` receives only Vine Attack, VineCycler and Oak Sage as sumskills (Skills row 241, sumskill1–3). So the Carrion Vine synergy on Solar Creeper mana is probably always 0. Oak Sage works because it is passed as sumskill3.

## 5. Suggested fixes (for PD2 devs)
1. **Grant to the owner, not by area.**
   - In PD 0x10300CA0, resolve the owner the way stock 0x6FC41260 does: MonsterData+0x28 → type/GUID → 0x6FD003A0.
   - Call 0x102BFBF0 on it directly, or on the owner and then party members.
   - If the area behaviour is intended for party members, keep 0x6FCC0C70 as well, but centre it on the owner (pass the owner's X/Y as args 2/3). Note that 0x6FCC0C70 still gathers rooms from the *centre unit* (arg 1), so pass the owner as arg 1 and skip it manually.
2. **Keep the AI and the grant consistent.**
   - In the CycleOfLife AI (0x6FCCB070, around 0x6FCCB124), do not eat when the leash warp failed.
   - Or require `#10522(vine, owner)` < aip5 before calling 0x6FD12380, or check the grant radius (40, centre distance) there.
3. **Make the finder respect st67's rule.**
   - Either reject corpses with `[+0x98]!=0 && [+0x94]==0` in the vine's corpse search (hook 0x6FC2C1F0 or its #10965 call at 0x6FC2C2BB for AI 111).
   - Or drop the owner check in st67 if Desecrate corpses are meant to be edible.
   - At minimum, when st67 rejects a corpse, mark it corpse_noselect so the vine does not loop on it.
4. **Revisit the static patch D2Game+0xAB26E (=1).**
   - Restoring 0 brings back "don't eat while the owner is full", which saves corpses.
   - If eating at full mana is intended, say so in the tooltip.
5. **Optional synergy fix.** Add `Cycle of Life` as a sumskill of `Vines` (241). Or evaluate VineCycler calcs with the owner as the calc unit.
