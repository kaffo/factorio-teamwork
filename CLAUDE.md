# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## About this mod

**Teamwork** (v0.5.1) is a Factorio 1.1 multiplayer mod published at https://mods.factorio.com/mod/teamwork. It divides the map into 2 or 4 team territories separated by an expanding neutral "No Man's Land" zone, with shared technology research and strict placement enforcement across team boundaries.

## Development workflow

There are no build scripts or test frameworks. To test changes, install the mod folder into your Factorio mods directory and run the game. Enable `teamwork-debug = true` in startup settings to use faster expansion intervals (15 seconds) during testing.

Factorio mod structure: Lua files are loaded in this order — `data.lua` → `data-updates.lua` (prototype phase), then `settings.lua` (settings phase), then `control.lua` + `teamwork-utils.lua` at runtime.

## Architecture

### File responsibilities

- **[control.lua](control.lua)** — All runtime game logic (~575 lines). Registers every event handler and contains the main systems below.
- **[teamwork-utils.lua](teamwork-utils.lua)** — Shared helpers: tech name predicates (`isSharedTech`, `isBackfillTech`, `backtechName`), messaging (`printAllPlayers`, `printForce`, `debugPrint`), expansion timing (`ExpandPeriod`), and table utilities.
- **[data-updates.lua](data-updates.lua)** — Prototype phase: iterates all technologies and creates expensive "backtech" duplicates for shared techs, costing `tech-cost-factor` × the original.
- **[prototypes/divider-technologies.lua](prototypes/divider-technologies.lua)** — Defines 6 divider-specific technologies that gate which item types can be placed in No Man's Land.
- **[settings.lua](settings.lua)** — Declares `tech-cost-factor` (startup), `num-teams` (runtime-global), `expand` (runtime-global), and `teamwork-debug` (startup).
- **[locale/en/teamwork.cfg](locale/en/teamwork.cfg)** — All player-facing strings.

### Core systems in control.lua

**Team initialization** (`on_init`, `on_player_created`): Creates peaceful Factorio forces named "left"/"right" (2-player) or "top-left"/"top-right"/"bottom-left"/"bottom-right" (4-player). Players are assigned round-robin and cannot switch teams.

**Divider zone**: A strip of hazard-concrete tiles along the map center axis (vertical for 2-player, cross-shaped for 4-player). Width starts at 6 tiles and expands by 1 tile per expansion event. `on_chunk_generated` fills newly-generated chunks with hazard concrete in divider coordinates.

**Expansion timing** (`on_tick` + `ExpandPeriod` in utils): Exponential decay — normal mode starts at 30-minute intervals with a 6-hour half-life down to a 2-minute minimum; extreme mode starts at 20 minutes with a 4-hour half-life to 1-minute minimum; debug mode uses 15-second intervals.

**Technology sharing** (`on_research_finished`): When any team researches a "shared" tech, it is automatically enabled for all other teams and a backtech (expensive duplicate, prefix `backtech-`) is added to the tech tree. Divider techs unlock specific item categories for placement in No Man's Land.

**Placement enforcement** (`on_built_entity`, `on_robot_built_entity`, `on_player_built_tile`, `on_robot_built_tile`): Entities placed on the wrong side of the divider are destroyed and returned to inventory. Global items (locomotives, tanks, cars) are exempt. Divider-allowed items (after tech research or rocket launch) may be placed in No Man's Land. Tiles placed in the divider zone are reverted to hazard concrete.

**Rocket progression** (`on_rocket_launched`): Items launched in a rocket are added to `divider_allowed_items`, unlocking them for No Man's Land placement as a mid/late-game reward.

### Key data structures (global table)

```lua
global.divider_size        -- current half-width of neutral zone in tiles
global.divider_allowed_items  -- set of item names allowed in No Man's Land
global.last_expand_tick    -- tick of last divider expansion
global.teams               -- list of force objects
global.player_team         -- map from player index → force
```
