# PD2 Damage Atlas

Every **Project Diablo 2** damage calculation, skill by skill, as flow charts: formulas and interactions traced from the game's reverse-engineered code, with a pipeline map, a hit lab and a developer reference.

**Live site:** https://roofooevazan.github.io/pd2-damage-atlas/

Part of [PD2 Calculators](https://roofooevazan.github.io/).

## What's here

| Path | What it is |
|---|---|
| `index.html` | The site: Skill flow, Pipeline map, Hit lab, Full reference and Developer reference. |
| `research/dmg_A_attack.md` … `dmg_D_incoming.md` | The four damage write-ups behind the atlas: your attack damage, the defender pipeline, spells/over time/summons, and damage to you and PvP. |
| `research/` (other files) | The wider research log behind all the PD2 Calculators tools: the hit roll, crit and crushing blow, open wounds and pierce, auras, skill damage and speed, Whirlwind, drops, experience, items, maps, mercenaries, monster AI, vendors, shrines and movement, the cube, uber bosses and reviews of the in-game Advanced Stats window. |
| `FINDINGS.md` | The research log shared by all the PD2 Calculators tools. |

The rules were recovered from the Diablo II 1.13c DLLs that PD2 uses (`D2Common.dll`, `D2Game.dll`, `D2Client.dll`), with PD2's changes from `ProjectDiablo.dll` applied. The page is one file with no server, build step or tracking. It loads the [marked](https://github.com/markedjs/marked) Markdown renderer from cdnjs, pinned with an integrity hash.

## Updating the site

Replace `index.html` with a new version, then commit. GitHub Pages republishes automatically within a minute or two.

## Disclaimer

This is a fan-made tool. It is not affiliated with or endorsed by Blizzard Entertainment or the Project Diablo 2 team. Diablo is a trademark of Blizzard Entertainment. No game assets are included.
