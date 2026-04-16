# Vostok Mods

A collection of mods for **[Road to Vostok](https://store.steampowered.com/app/1963610/Road_to_Vostok/)** - a hardcore survival FPS set on the Finnish-Russian border.

All mods use the [Metro / Community Mod Loader](https://modworkshop.net/mod/55914) and are compatible with each other.

---

## Mods

### [Quick Stack & Sort](https://github.com/Dominicode-s/vostok-quick-stack)
One-click inventory sorting, smart item transfer, take-all / store-all, and configurable hotkey support.
- Sort any grid (inventory or container) with auto-stacking
- Transfer matching items between inventory and container
- Take All / Store All buttons
- MCM hotkey support (keyboard + mouse buttons)

### [Cash System](https://github.com/Dominicode-s/vostok-cash)
Adds a physical cash economy to the game - lootable cash bundles, wallet UI, and buy/sell mechanics with traders.
- Physical cash items that spawn in containers and on enemies
- Wallet UI integrated into the trader interface
- Buy and sell items for cash
- Custom 3D cash bundle model and textures

### [XP & Skills System](https://github.com/Dominicode-s/vostok-skills)
An RPG-style progression system with 13 upgradeable skills and a Prestige tier.
- Earn XP from looting, killing, trading, and completing tasks
- 13 skills: Vitality, Endurance, Pack Mule, Hunger/Thirst/Cold Resist, Iron Will, Regeneration, Stealth, Recoil Control, Athleticism, Scavenger, and Composure (reduces camera shake on hits)
- Prestige system — max every skill to unlock permanent stat bonuses that survive death
- Skill UI tab integrated into the inventory screen
- Fully configurable via MCM

### [Run Summary](https://github.com/Dominicode-s/vostok-run-summary)
After-action report for every raid — shows a detailed breakdown when you die or return to the shelter.
- Combat, loot, survival, economy, XP, and time stats
- Tracks survival drain and restoration separately (eating/drinking shows up)
- Browse your last 10 runs in the history tab
- F6 hotkey to reopen the last summary anytime
- Optional integration with Cash System and XP & Skills mods

### [Secure Container](https://github.com/Dominicode-s/vostok-secure-containers)
Equipable pouches with a dedicated slot that protect a handful of items from loot-on-death and let you move them between runs.
- Three tiers: Field Pouch (2 slots), Secure Pouch (4 slots), Secure Case (6 slots)
- Custom 3D models and icons per tier
- Spawns naturally in civilian / industrial / military loot pools
- Middle-click to open the secure panel; shift-click or drag-and-drop to secure items
- Contents survive death, scene changes, and main-menu reloads
- Configurable slot counts, keybind, and loot spawn toggle via MCM

---

## Modding Guide

New to modding Road to Vostok? Check out the **[Modding Guide](MODDING_GUIDE.md)** - a comprehensive reference covering mod structure, override chaining, UI injection, MCM integration, hotkeys, debugging, and more.

---

## Installation

1. Install the [Metro / Community Mod Loader](https://modworkshop.net/mod/55914)
2. Download the `.vmz` file from each mod's releases (or from [ModWorkshop](https://modworkshop.net))
3. Drop the `.vmz` into your game's `mods/` folder
4. (Optional) Install [MCM](https://modworkshop.net/mod/55932) for in-game configuration

## Compatibility

All mods are designed to work together and with other mods:
- **Quick Stack & Sort** - Pure autoload, no script overrides
- **Cash System** - Minimal overrides (Interface.gd only), uses signals API
- **XP & Skills** - Two overrides (Character.gd, Interface.gd), both call `super()` for proper chaining; other behaviour uses polling so it composes cleanly with HellMAI, Injuries System Rework, Aim Punch Modifier, etc.
- **Run Summary** - Pure autoload, no script overrides
- **Secure Container** - Runtime Interface.gd override via `take_over_path`, scoped to pouch drop/place handling

## Links

- [ModWorkshop - Quick Stack & Sort](https://modworkshop.net/mod/55986)
- [ModWorkshop - Cash System](https://modworkshop.net/mod/55951)
- [ModWorkshop - XP & Skills](https://modworkshop.net/mod/55940)
- [ModWorkshop - Run Summary](https://modworkshop.net/mod/56019)
- [ModWorkshop - Secure Container](https://modworkshop.net/mod/56154)
