# Depths of Kharzul

A text-based fantasy dungeon adventure, playable entirely in Python's standard
library (no installs required).

## How to run in IDLE

1. Open `game.py` in IDLE.
2. Press **F5** (or Run > Run Module).
3. The game runs in the Shell window — type commands and press Enter.

You can also run it from a terminal with `python3 game.py`.

## Goal

Explore the ruined dungeon of Kharzul, find the Heart of Kharzul, and escape
through the throne room to the tunnel outside. A Guardian blocks the only
way out — defeat it to win. Escaping with the Heart gives the true ending.

## Commands

- `north` / `south` / `east` / `west` / `up` / `down` (or `n`/`s`/`e`/`w`/`u`/`d`) — move
- `look` — describe the room again
- `inventory` (`i`) — show items, HP, and equipment
- `take <item>` / `drop <item>`
- `use <item>` — drink a potion, eat food, or equip a weapon/shield
- `equip <item>` — wield a weapon or raise a shield
- `examine <thing>` — look closely at an item or feature
- `talk <person>` — speak to someone in the room
- `answer <text>` — answer a riddle
- `attack` — fight an enemy in the room
- `save` / `load` — save or load your progress (`kharzul_save.json`)
- `help` — show the command list
- `quit` — exit the game

Progress is saved to `kharzul_save.json` in this folder; delete it to start
fresh, or choose "New Game" from the main menu.
