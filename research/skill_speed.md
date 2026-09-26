# Skills that change attack speed while their animation plays (PD2 / 1.13c)

Status: everything below is READ from the disassembly (D2Game, D2Client, D2Common, PD.asm) plus Skills.txt. Nothing new was run in the native harness.
Data: `excel_live/Skills.txt` and the uploaded Skills.txt agree for every row used here.

## Result

| id | skill | server | client | value added to stat 68 | removed |
|---|---|---|---|---|---|
| 270 | Dragon Tail | srvstfunc 27 0x6FCB2940 → 0x6FCBE7B0 | cltstfunc 9 0x6FAF0400 → 0x6FB50880 | Param4 (raw, −20) | next mode change |
| 133 | Double Swing | **none** (srvstfunc 0) | cltstfunc 27 0x6FB76290 → 0x6FB50880 | cltcalc1 = `par5` = **+50** | next mode change (leaving SQ) |
| 152 | Berserk | srvstfunc 32 0x6FC473C0 → 0x6FCBE7B0 | **none** (cltstfunc 0) | calc3 = `ln56` = 5 + (lvl−1) | next mode change |
| 126 | Bash | srvstfunc 32 | none | calc3 (blank in PD2 → 0) | — |
| 144 | Concentrate | srvstfunc 32 | none | calc3 (blank in PD2 → 0) | — |

Monster-only users of srvstfunc 32 (no client function): BearSmite 326 and UberTalicBash 511. Both have calc3 blank.

**No other player skill** changes attackrate or IAS just for its animation. That includes all the PD2 overrides: Frenzy, Zeal, Fend, Fury, Whirlwind, Blade Dance, Strafe, Dragon Claw, Dragon Talon, Leap Attack, Double Throw, Charge, the martial-arts charge-ups and Tiger Strike. See "Checked, no effect" below.

`calc` (JSON `addToCalc`): held attacks are paced by the client (FINDINGS). A calculator should therefore add only the client-side values:
- Dragon Tail: add Param4.
- Double Swing: add +50.
- Berserk: do not add it to the displayed frames. The server's copy runs faster, which only moves the server hit frame earlier.

## Mechanism

**Server helper 0x6FCBE7B0**(edi = unit, [esp+4] = value):
- #11013 allocates a stat list with **flags 4** (owner = unit type/id).
- #10807 attaches it (mode 1).
- #10188 sets stat 0x44 attackrate = value.
- #10819 (0x6FD83110) recomputes the animation rate.

The client helper **0x6FB50880** is the same code using the client's import thunks.

- **Callers.** The only callers are:
  - D2Game: 0x6FCB2996 (srvstfunc 27) and 0x6FC47449 (srvstfunc 32).
  - D2Client: 0x6FAF042B/0x6FAF0442 (cltstfunc 9) and 0x6FB762D0 (cltstfunc 27).
  - PD.dll never calls either helper.
    - Its resolver 0x1029A410 resolves D2Client+0xA0880, but the result is never used; the only references are unreferenced init stubs.
    - PD has no patch record inside either helper or in the call sites.
- **Other places that write stat 0x44 in D2Game/D2Client:**
  - the aurastat loop 0x6FC647D0 (also writes 0x45 other_animrate), used by persistent states;
  - chill 0x6FCFC780 (state 0xB) and slow 0x6FC6EFAC/0x6FCCD2DE, both applied to the *target*;
  - base-stat init at 0x6FAC9252 and 0x6FAFD206/73B (attackrate = 100);
  - there are no per-animation users besides the ones above.
- **Arguments.**
  - srvstfunc(ecx = game, edx = unit, arg1 = skillId, arg2 = skill level).
  - st32 evaluates `#10786(unit, SkillsTxt+0x140, skillId, level)`.
  - The client cltstfunc(ecx = unit, edx = skillId, arg1 = level). cltst27 evaluates `#10786(unit, SkillsTxt+0x114, skillId, level)`.
  - `level` is the skill level the skill is being used at (with +skills).
  - Dragon Tail passes Param4 raw, so there is no level dependence.
- **Field offsets** (VERIFIED from the D2Common Skills.txt field table at 0x6FDB44D7…/0x6FDB53F6…):

  | field | offset |
  |---|---|
  | calc1 | +0x138 |
  | calc3 | **+0x140** |
  | cltcalc1 | **+0x114** |
  | cltcalc2 | +0x118 |
  | cltcalc3 | +0x11C |
  | Param4 | +0x154 |
  | aurastate | +0x80 |
  | aurastat1..6 | +0x54.. |
  | aurastatcalc1..6 | +0x68.. |

**Order and removal**
- **Order.** The server attack start is 0x6FC987C0/0x6FC98850. It calls #11090 SetUnitMode, then 0x6FCFFF10 (rate), then 0x6FCC0830 → 0x6FCBFD00, which dispatches srvstfunc (0x6FCBFE3E). So the list is attached *after* the mode is set.
- **Removal.** D2Common #11090 (0x6FD83920) runs only when the new mode differs from the current one. It sets the mode, then calls **#10196 (0x6FD8AE60)**, which frees every flag-4 list, then 0x6FD835F0 (rate recompute). So the bonus lasts until the next mode change: end of the skill → neutral, or another command.
- **Caveat (READ).** #11090 with the *same* mode returns at once without freeing anything. If the server restarts the same mode mid-animation, the start function would attach a second list and the value would stack (e.g. Dragon Tail −40).
  - This happens when the server accepts a new command before END (FINDINGS: KK accepted while cur ≤ END+5).
  - The client always goes through neutral between local attacks, so the client never stacks.
  - This was not traced further.

## Client vs server
- **Dragon Tail:** identical on both sides.
- **Double Swing:** the client gets +50 and the server gets nothing.
  - PD2's Skills.txt has `calc3 = par5` described as "attack rate bonus", but Double Swing has no srvstfunc, so calc3 is never read. srvdofunc 70 (0x6FC47B90) reads no calc.
  - The server SQ animation is therefore slower than the client's.
  - Command acceptance during SQ goes through 0x6FC98100, which PD2 patches (FINDINGS: not run). Whether the server can delay the client's next Double Swing is **open**.
  - BH also models Double Swing as +Param5 (the same number, 50).
- **Berserk:** the server gets +(4+lvl) and the client gets nothing.
  - PD2 relabelled calc3 "phys pierce"; the aura stat `passive_phys_pierce` uses `min(ln56,45)` directly. Stock srvstfunc 32 still adds calc3 as attackrate, and PD2 does not override st32.
  - Because the client paces held attacks, the visible speed is the client's unmodified speed.
- **Stat sync.** A flag-4 list without a state is not sent to the client. The client learns about stat lists only through state packets (0x6FB226EB/0x6FB2374D state code), so a server-only value never reaches the client (READ).

## PD2 overrides checked (no attack speed change)
PD override tables, recovered from PD init 0x102BE2F0…0x102BE743 plus resolvers:

| global | target |
|---|---|
| 0x104E30E0 | srvstfunc (D2Game+0x107338) |
| 0x104E3114 | srvdofunc (+0x1074A8) |
| 0x104E626C | cltstfunc (D2Client+0xDE928) — writes entries 54–58 |
| 0x104E2FC8 | cltdofunc (D2Client+0xDEA48) — writes entries 8, 83, 94, 97–118; the stock table has 130 slots, count at 0x6FB8EC50 |

Module ids used by PD resolvers: 0 = D2Client, 1 = D2Common, 6 = D2Game.

Findings for each override:
- **srvstfunc 37 → PD 0x10300A90** (Zeal, Fend, Fury): builds a flag-4 list with `aurastate` (+0x80) and aurastats, through PD 0x102EF500 → D2Game 0x6FC648E0. The aurastats are only `inc_splash_radius` (state temp_splash), so there is no speed change.
- **srvstfunc 78 → PD 0x103014B0** (Frenzy, Tiger Strike): flag-4 list with state 0xE0 and stat 0x1DE (478 inc_splash_radius) from calc2/calc4. There is no speed change.
  - Frenzy's +attackrate (`dm56`) is the persistent `frenzy` state from srvdofunc 9 (0x6FC49600) aurastats. It is not per animation.
- **srvstfunc 28/29/66–77:** Blade Shield, Sacrifice, Blood Warp, Find Item, Charge, Holy Sword, Smite, golems and so on. None sets 0x44.
- **srvdofunc overrides that make temp lists** (51, 164, 170, 174, 180) set stats 0x3C, 0x3E, 0x19, 0x150 and 0x1DE only.
- **cltstfunc 54–58 and cltdofunc 97–118:** only 57 (Charge, PD 0x102F7BC0) and 58 make stat lists. Charge reads velocity 0x43; there is no 0x44.
- **PD's own calls to #10819** (0x10276850) are only 0x1026921E (aura-path wrapper) and 0x10272BA6 (PD's aurastat loop). Both are for states.
- **Stock functions:**
  - srvstfunc 38 (Whirlwind, Blade Dance) applies aurastats `velocitypercent` and `item_attackrate=100`. `item_attackrate` is not an ItemStatCost stat, so it is ignored.
  - srvstfunc 23 (charge-ups), srvstfunc 24 and srvdofunc 42 (Dragon Talon: Param4 50 is unused for speed), srvstfunc 40 (Leap Attack), srvdofunc 12/13 (Strafe, Zeal, Fend, Fury follow-ups) and Dragon Claw: none sets 0x44.

## Persistent states (not per-animation; already in FINDINGS "Speed skills")
These are aura/state stat lists that last beyond the animation, all READ from Skills.txt:
- Frenzy `attackrate dm56`
- Fanaticism `dm34`
- Werewolf `dm34`
- Quickness / Burst `((110*blvl)*(par4-par3))/(100*(blvl+6))+par3`
- Increased Speed passive `item_fasterattackrate blvl*2` (through EIAS)
- negative: Decrepify `attackrate −min(par6+lvl+CurMas,60)`, Holy Freeze `−dm34`, Confuse `attackrate par5+par6*lvl+…`

When an aurastat is attackrate, 0x6FC647D0 also writes other_animrate (0x45) with the same value.
