# Road to Vostok — Modding Guide

How we build and package mods for Road to Vostok (Godot 4.6.1 EA) using the Metro/Community Mod Loader.

---

## Table of Contents

1. [How the Game Works](#how-the-game-works)
2. [Mod Loader Overview](#mod-loader-overview)
3. [Project Structure](#project-structure)
4. [Two Modding Patterns](#two-modding-patterns)
5. [Script Override Pattern](#script-override-pattern)
6. [Override Chaining & Priority](#override-chaining--priority)
7. [Autoload-Only Pattern](#autoload-only-pattern)
8. [Node Discovery](#node-discovery)
9. [Scene Tree Reference](#scene-tree-reference)
10. [UI Injection](#ui-injection)
11. [Custom Hotkeys (InputMap)](#custom-hotkeys-inputmap)
12. [Inter-Mod Communication (Signals API)](#inter-mod-communication-signals-api)
13. [Data Persistence](#data-persistence)
14. [MCM Integration](#mcm-integration)
15. [mod.txt Format](#modtxt-format)
16. [Packaging as VMZ](#packaging-as-vmz)
17. [Compatibility Checklist](#compatibility-checklist)
18. [Known Gotchas](#known-gotchas)
19. [Debugging Tips](#debugging-tips)
20. [Surviving EA Updates](#surviving-ea-updates)
21. [File Paths Reference](#file-paths-reference)

---

## How the Game Works

- **Engine**: Godot 4.6.1 EA (bytecode revision `ebc36a7`, 4.5.0-stable)
- **Game data singleton**: `gameData` — a preloaded `.tres` resource (`GameData.tres`) accessible everywhere. Holds all player state: health, stamina, hunger, hydration, mental, temperature, oxygen, inventory flags, etc.
- **Key scripts**: `Character.gd` (survival stats/tick), `Interface.gd` (~3765 lines, entire UI), `AI.gd` (enemy logic), `Trader.gd` (trade logic), `LootContainer.gd` (container interaction), `Controller.gd` (movement/physics), `HUD.gd` (world-space tooltips), `Database.gd` (item/scene preloader)
- **Item database**: `Database.master.items` — array of `ItemData` resources with fields like name, icon, type, value, capacity, insulation, etc.
- **Dual tooltip system**: The game has two completely separate tooltip paths:
  - **Inventory tooltips**: `Pickup.UpdateTooltip()` sets `gameData.tooltip` — fires when hovering items in inventory
  - **World tooltips**: `HUD._physics_process()` reads data and sets `label.text` — fires when looking at items on the ground
  - If you only override one, you'll miss the other. The mod loader flags this as a `TOOLTIP WARNING`.

## Mod Loader Overview

We use the **Metro/Community Mod Loader** (`modloader.gd`, ~67KB, lives at `user://modloader.gd`).

- Enabled via `override.cfg` in the game directory (points autoload to the loader)
- Scans the `mods/` folder for `.vmz` files (ZIP archives with a `mod.txt`)
- Mounts VMZ contents into the virtual filesystem at `res://mods/<ModFolder>/`
- Instantiates autoloads declared in `mod.txt` as children of the ModLoader node
- Supports MCM (Mod Configuration Menu) for in-game settings
- Has an Updates tab that checks modworkshop.net for newer versions
- Writes a **conflict report** to `%AppData%\Road to Vostok Demo\modloader_conflicts.txt` every launch — always check this when testing

### Conflict Report Tags

| Tag | Severity | Meaning |
|-----|----------|---------|
| `CHAIN OK` | ✅ Good | Multiple mods override the same script and all call `super()` properly |
| `CHAIN BROKEN` | ⚠️ Warning | Someone in the chain skips `super()`, silently killing mods below them |
| `CONFLICT` | ❌ Critical | Two mods have the same file at the same `res://` path — last-loaded wins |
| `CONFLICT: take_over_path` | ❌ Critical | Two mods both call `take_over_path()` on the same script |
| `DATABASE OVERRIDE` | ❌ Critical | A mod replaced `Database.gd` — can break other mods' scene overrides |
| `OVERHAUL` | ℹ️ Info | A mod overrides 5+ core scripts |
| `TOOLTIP` | ⚠️ Warning | Mod overrides `UpdateTooltip()` but may miss world tooltip path |
| `NO SUPER` | ⚠️ Warning | Lifecycle method override without `super()` |
| `BAD ZIP` | ❌ Critical | Archive has Windows backslash paths — re-pack with 7-Zip or PowerShell |

## Project Structure

```
_modfiles/
├── MODDING_GUIDE.md              # This file
├── xp_vmz/                       # XP & Skills System source
│   ├── mod.txt                   # Mod metadata
│   └── mods/
│       └── XPSkillsSystem/
│           ├── Main.gd           # Autoload entry point
│           ├── Character.gd      # Override: survival stats
│           ├── Interface.gd      # Override: Skills UI tab
│           ├── AI.gd             # Override: kill XP
│           ├── LootContainer.gd  # Override: container search XP
│           └── Trader.gd         # Override: trade XP
├── cash_vmz/                     # Cash System source
│   ├── mod.txt
│   └── mods/
│       └── CashSystem/
│           └── Main.gd           # Autoload: wallet + UI injection + signals API
├── quickstack_vmz/               # Quick Stack & Sort source
│   ├── mod.txt
│   └── mods/
│       └── QuickStack/
│           └── Main.gd           # Autoload: sort/transfer buttons + hotkey
_extracted/                        # Decompiled vanilla scripts (reference only)
├── Scripts/
│   ├── Character.gd
│   ├── Interface.gd
│   ├── AI.gd
│   └── ...
├── UI/
│   └── Interface.tscn
mods/                              # Packaged VMZs go here (game reads this)
├── XP-Skills-System.vmz
├── Cash-System.vmz
└── Quick-Stack-Sort.vmz
```

## Two Modding Patterns

### 1. Script Override (`take_over_path`)

Replace a vanilla script with your modified version. Your script `extends` the original and then calls `take_over_path()` to hijack its resource path. All existing scene nodes using that script will run your code instead.

**Use when**: You need to modify existing game behavior (stat calculations, UI layout, enemy logic).

**Pros**: Full control over game logic, can modify any method.
**Cons**: Potential chain conflicts with other mods overriding the same script.

### 2. Autoload-Only (UI Injection)

A standalone Node added as an autoload. Doesn't replace any scripts — instead, it finds existing scene nodes at runtime and adds new UI elements as children.

**Use when**: You're adding new features without changing existing behavior (wallet system, HUD overlays, inventory tools, debug tools).

**Pros**: Zero conflict risk — no override chains to worry about, fully compatible with all other mods.
**Cons**: Can only read/call the game's public API, can't modify internal behavior.

**Prefer autoload-only when possible.** It's the safest pattern and survives game updates better.

## Script Override Pattern

This is the core technique. In your autoload's `_ready()`:

```gdscript
func overrideScript(path: String):
    var script = load(path)
    script.reload()
    var parent = script.get_base_script()
    script.take_over_path(parent.resource_path)
```

Your override script must extend the vanilla script **by file path** (not class name):

```gdscript
extends "res://Scripts/Character.gd"

# Override any function — call super() to keep vanilla behavior
func Health(delta):
    super(delta)  # ALWAYS call super() first
    # Your custom logic after vanilla runs
```

**Important**: `take_over_path()` replaces the script resource globally, but it does NOT update already-instantiated nodes that loaded the script before your mod ran. This is why `gameData` (a preloaded `.tres` singleton) keeps its original script — you can't override its script class, only read/write its properties.

### Calling overrideScript

From your autoload `_ready()`:

```gdscript
func _ready():
    overrideScript("res://mods/YourMod/Character.gd")
    overrideScript("res://mods/YourMod/Interface.gd")
```

The path is where the file lives inside the VMZ archive (mounted at `res://mods/`).

### Self-Destructing Autoloads

If your autoload only registers overrides and doesn't need to persist, free it immediately:

```gdscript
func _ready():
    overrideScript("res://mods/YourMod/HUD.gd")
    overrideScript("res://mods/YourMod/Pickup.gd")
    queue_free()  # Clean up — nothing to stick around for
```

This keeps the scene tree clean. Only stay alive if you need `_process()`, `_input()`, signals, or persistent state.

## Override Chaining & Priority

When multiple mods override the same script, they form a **chain**. The `priority` field in `mod.txt` controls the order:

```ini
[mod]
name="My Small Tweak"
id="my-tweak"
version="1.0.0"
priority=1
```

- **Lower priority loads first** (default is `0`)
- **Higher priority loads later** and sits on **top** of the chain
- The chain runs top → bottom when `super()` is called

### Example Chain

With ImmersiveXP (priority 0) and your mod (priority 1):

```
Your override._process(delta)
  → calls super(delta)
    → ImmersiveXP._process(delta)
      → calls super(delta)
        → Vanilla._process(delta)
```

**The golden rule: always call `super()` in lifecycle methods** (`_ready`, `_process`, `_physics_process`, etc.) and any game method you override. Without it, every mod below you in the chain silently loses its overrides. The mod loader reports this as `CHAIN BROKEN`.

### Priority Guidelines

| Priority | Use For |
|----------|---------|
| `0` (default) | Overhaul mods that others build on top of |
| `1-10` | Small mods that layer on top of overhauls |
| Higher | Only when you specifically need to be last in the chain |

## Autoload-Only Pattern

The Cash System and Quick Stack mods use this — no `take_over_path()` at all. Instead:

1. Wait for the game scene to load (check `get_tree().current_scene`)
2. Find the Interface node under `Core/UI`
3. Create new Controls/Buttons/Labels in code
4. Add them as children of existing nodes

```gdscript
var deal_panel = interface_node.get_node("Deal/Panel")
var my_button = Button.new()
my_button.text = "Sell"
deal_panel.add_child(my_button)
```

This is less invasive and more compatible with other mods since you're not replacing any scripts.

### Autoload Timing Gotcha

Autoloads run `_ready()` and `_process()` **before** the first scene is ready. You must guard against null:

```gdscript
func _process(_delta):
    var scene = get_tree().current_scene
    if scene == null:
        return  # Too early — scene not loaded yet
    
    # First scene is "Menu" (main menu), game maps are "Map"
    if scene.name != "Map":
        return
    
    # Safe to find game nodes now
```

### Scene Change Handling

When the player transitions between scenes (Menu → Map, or Map → Map), your cached node references become invalid. Track the scene name and reset:

```gdscript
var _interface = null
var _last_scene: String = ""

func _process(_delta):
    var scene = get_tree().current_scene
    if scene == null:
        return
    
    # Reset on scene change
    if scene.name != _last_scene:
        _last_scene = scene.name
        _interface = null  # Force re-discovery
    
    if _interface == null:
        # Find interface...
```

## Node Discovery

**Critical issue**: The Metro Mod Loader adds autoloads as children of itself:

```
/root/ModLoader/YourAutoload    ← actual path
/root/YourAutoload              ← what you might expect (WRONG)
```

So `get_node_or_null("/root/YourAutoload")` will return `null`.

**Solution**: Use `Engine.set_meta()` / `Engine.get_meta()` for cross-script communication:

```gdscript
# In your autoload _ready():
Engine.set_meta("MyMod", self)

# In any override script:
var my_mod = Engine.get_meta("MyMod", null)
if my_mod:
    my_mod.some_function()
```

This works regardless of where the node ends up in the tree.

### Finding the Interface Node

The Interface is **not** at the scene root. It's nested under `Core/UI`:

```gdscript
# DON'T do this — won't find it:
var iface = scene.get_node_or_null("Interface")

# DO this — search Core/UI children:
var core_ui = scene.get_node_or_null("Core/UI")
if core_ui:
    for child in core_ui.get_children():
        if child.get("containerGrid") != null:
            _interface = child
            break
```

Checking for `containerGrid` is a reliable way to identify the Interface node among its siblings.

## Scene Tree Reference

```
Map (root scene)
├── Content (NavigationRegion3D)
│   └── Cabin (Node3D)
├── World (Node3D)
│   └── Environment, Planet, VFX, Audio
├── Core (Node3D)
│   ├── UI (Control)              ← UIManager script lives here
│   │   └── Interface             ← Separate node! UIManager.interface = $Interface
│   │       ├── Inventory (Control)   ← offset_left=1216, offset_top=128
│   │       │   ├── Header (ColorRect) ← 320px wide, 32px tall, y=-32
│   │       │   └── Grid (TextureRect)
│   │       ├── Container (Control)   ← offset_left=192, offset_top=128
│   │       │   ├── Header (ColorRect) ← 256px wide, 32px tall, y=-32
│   │       │   └── Grid (TextureRect)
│   │       ├── Deal (Control)        ← Trade panel
│   │       ├── Tools (Control)       ← Tab buttons
│   │       └── ...
│   ├── Tools (Node)
│   ├── Audio (Node3D)
│   ├── Camera (Camera3D)
│   └── Controller (CharacterBody3D)
└── Killbox (Area3D)
```

### UIManager vs Interface — Critical Distinction

**UIManager** (`/root/Map/Core/UI`) and **Interface** (`UIManager.interface`) are **separate nodes** with different responsibilities:

| Node | Script | Key Properties | Key Methods |
|------|--------|---------------|-------------|
| **UIManager** | `UIManager.gd` | `interface`, `HUD`, `settings` | `OpenContainer()`, `ClickAudio()` |
| **Interface** | `Interface.gd` | `containerGrid`, `inventoryGrid`, `container` | `Create()`, `AutoStack()`, `AutoPlace()`, `Open()` |

- `UIManager.OpenContainer(container)` sets `interface.container` then calls `interface.Open()`
- `containerGrid`, `inventoryGrid`, and all item manipulation methods live on **Interface**, not UIManager
- If you find a UIManager reference, access Interface via `ui_manager.interface`
- `UIManager.ClickAudio()` plays UI click sound but may not work from mod context — create your own `AudioStreamPlayer` instead

### Key Interface Properties

| Property | Type | Description |
|----------|------|-------------|
| `inventoryGrid` | Grid | Player inventory grid |
| `containerGrid` | Grid | Open container's grid |
| `container` | Node | The open container reference (null when closed) |
| `hoverGrid` | Grid | Grid currently under the mouse (only set when hovering an item) |
| `hoverItem` | Item | Item currently under the mouse |
| `cellSize` | int | 64px per grid cell |

### Key Interface Methods

| Method | Description |
|--------|-------------|
| `Create(slotData, grid, useDrop)` | Create an item in a grid. `useDrop=true` drops on ground if no room |
| `AutoStack(slotData, targetGrid)` | Try to stack item into existing stacks. Returns bool |
| `AutoPlace(item, targetGrid, sourceGrid, useDrop)` | Find free space and place item. Returns bool |
| `GetHoverGrid()` | Returns the grid under mouse position (checks all visible grids) |
| `PlayClick()` / `PlayError()` / `PlayStack()` | UI audio feedback |
| `UpdateUIDetails()` | Refresh UI labels |
| `UpdateStats(bool)` | Recalculate inventory stats |

### Grid System

| Property/Method | Description |
|----------------|-------------|
| `items[]` | Array of Item children in the grid |
| `grid{}` | Dict `grid[x][y] = bool` for occupancy tracking |
| `gridWidth` / `gridHeight` | Grid dimensions in cells |
| `Spawn(item)` | Find first free slot (top-left → bottom-right) and place |
| `Place(item)` | Place item at its current position |
| `Pick(item)` | Remove item from grid tracking |
| `CheckGridSpace(x, y, w, h)` | Check if area is free |

Items have `slotData.itemData.size` as `Vector2` (width × height in cells). `item.rotated` swaps width/height.

## UI Injection

### Adding a Tab Button (Skills UI example)

The game's tool buttons live at `$Tools/Buttons/Margin/Buttons` (HBoxContainer):

```gdscript
var buttonsContainer = $Tools / Buttons / Margin / Buttons
var myButton = Button.new()
myButton.text = "Skills"
myButton.toggle_mode = true
myButton.size_flags_horizontal = Control.SIZE_EXPAND_FILL
myButton.size_flags_vertical = Control.SIZE_EXPAND_FILL
buttonsContainer.add_child(myButton)
# Insert before the last child to maintain layout
buttonsContainer.move_child(myButton, buttonsContainer.get_child_count() - 2)
```

### Adding Buttons to Container/Inventory Headers

Headers are `ColorRect` nodes — they have **no layout management**. Adding children to them won't auto-position anything. Instead, add to the parent `Control` with absolute positioning:

```gdscript
var btn_row = HBoxContainer.new()
btn_row.position = Vector2(0, -56)  # Above the header
btn_row.size = Vector2(256, 22)
container_ui.add_child(btn_row)     # Add to Container, not Header
```

### Button Best Practices

```gdscript
btn.mouse_filter = Control.MOUSE_FILTER_STOP   # Catch clicks (Button default)
btn.focus_mode = Control.FOCUS_NONE             # Prevent stealing keyboard focus
```

**Mouse filter warning**: Any `Control` node added to an existing panel defaults to `MOUSE_FILTER_STOP`, which blocks mouse events to sibling nodes behind it. Always set `mouse_filter = Control.MOUSE_FILTER_IGNORE` on layout containers; individual buttons will still catch clicks since Button defaults to `MOUSE_FILTER_STOP`.

### Adding UI to the Deal Panel (Cash System example)

```gdscript
var wallet_container = Control.new()
wallet_container.mouse_filter = Control.MOUSE_FILTER_IGNORE  # IMPORTANT!
deal_panel.add_child(wallet_container)
```

## Custom Hotkeys (InputMap)

Register custom input actions at runtime for configurable hotkeys:

```gdscript
const MY_ACTION = "my_mod_action"

func _register_hotkey(key_code: int):
    if not InputMap.has_action(MY_ACTION):
        InputMap.add_action(MY_ACTION)
    else:
        InputMap.action_erase_events(MY_ACTION)
    var ev = InputEventKey.new()
    ev.keycode = key_code
    InputMap.action_add_event(MY_ACTION, ev)
```

### Handling Input

Use `_input()` rather than polling `Input.is_action_just_pressed()` in `_process()` — it's more reliable, especially when UI is open:

```gdscript
func _input(event):
    if event is InputEventKey and event.pressed and not event.echo:
        if event.keycode == my_key:
            do_something()
    # Also handle mouse buttons:
    if event is InputEventMouseButton and event.pressed:
        if event.button_index == MOUSE_BUTTON_XBUTTON1:  # Mouse 4
            do_something()
```

**Note**: MCM's Keycode config type only captures keyboard keys. For mouse button support, use a Dropdown config with options like "Mouse 4 (Back)", "Mouse 5 (Forward)", and handle `InputEventMouseButton` separately.

## Inter-Mod Communication (Signals API)

Allow other mods to hook into your mod's events using signals on the Engine meta:

```gdscript
# In your mod — declare and emit signals:
signal on_sell(item_name: String, sell_price: int)
signal on_buy(item_name: String, buy_price: int)

func _ready():
    Engine.set_meta("MyModMain", self)

func _some_action():
    on_sell.emit("Bandage", 50)
```

```gdscript
# In another mod — connect to the signals:
func _ready():
    var other_mod = Engine.get_meta("MyModMain", null)
    if other_mod:
        other_mod.on_sell.connect(_on_item_sold)
        other_mod.on_buy.connect(_on_item_bought)

func _on_item_sold(item_name: String, price: int):
    print("Player sold %s for %d" % [item_name, price])
```

This pattern lets mods communicate without knowing each other's internals. Always null-check the meta in case the other mod isn't installed.

## Data Persistence

Use `ConfigFile` to save/load mod data to `user://`:

```gdscript
# Save
func SaveData():
    var cfg = ConfigFile.new()
    cfg.set_value("section", "key", value)
    cfg.save("user://MyModData.cfg")

# Load
func LoadData():
    var cfg = ConfigFile.new()
    if cfg.load("user://MyModData.cfg") == OK:
        value = cfg.get_value("section", "key", default_value)
```

`user://` maps to `%APPDATA%\Road to Vostok\` on Windows.

Save after every meaningful action (XP gain, purchase, etc.) — the game can crash or be force-quit at any time.

## MCM Integration

[Mod Configuration Menu](https://modworkshop.net) lets users configure your mod in-game.

### Setup

```gdscript
func _try_load_mcm():
    if ResourceLoader.exists("res://ModConfigurationMenu/Scripts/Doink Oink/MCM_Helpers.tres"):
        return load("res://ModConfigurationMenu/Scripts/Doink Oink/MCM_Helpers.tres")
    return null
```

### Config Types

```gdscript
var _config = ConfigFile.new()

# Int setting
_config.set_value("Int", "my_int", {
    "name" = "Display Name",
    "tooltip" = "Description for the user",
    "default" = 10, "value" = 10,
    "minRange" = 0, "maxRange" = 100,
    "menu_pos" = 1
})

# Float setting
_config.set_value("Float", "my_float", {
    "name" = "Float Setting",
    "tooltip" = "A decimal value",
    "default" = 0.5, "value" = 0.5,
    "minRange" = 0.0, "maxRange" = 1.0,
    "menu_pos" = 2
})

# Bool setting
_config.set_value("Bool", "my_toggle", {
    "name" = "Toggle Setting",
    "tooltip" = "On or off",
    "default" = true, "value" = true,
    "menu_pos" = 3
})

# Keycode setting (keyboard keys only)
_config.set_value("Keycode", "my_key", {
    "name" = "Hotkey",
    "tooltip" = "Press to activate",
    "default" = KEY_Z, "value" = KEY_Z,
    "menu_pos" = 4
})

# Dropdown setting
_config.set_value("Dropdown", "my_choice", {
    "name" = "Choose Option",
    "tooltip" = "Pick one",
    "default" = 0, "value" = 0,
    "options" = ["Option A", "Option B", "Option C"],
    "menu_pos" = 5
})

# String setting
_config.set_value("String", "my_text", {
    "name" = "Text Field",
    "tooltip" = "Enter text",
    "default" = "hello", "value" = "hello",
    "menu_pos" = 6
})

# Color setting
_config.set_value("Color", "my_color", {
    "name" = "Pick Color",
    "tooltip" = "Choose a color",
    "default" = Color.WHITE, "value" = Color.WHITE,
    "menu_pos" = 7
})
```

### Saving & Registering

```gdscript
const MCM_FILE_PATH = "user://MCM/MyModID"
const MCM_MOD_ID = "MyModID"

# First-time: create config. Otherwise: merge updates.
if !FileAccess.file_exists(MCM_FILE_PATH + "/config.ini"):
    DirAccess.open("user://").make_dir_recursive(MCM_FILE_PATH)
    _config.save(MCM_FILE_PATH + "/config.ini")
else:
    _mcm_helpers.CheckConfigurationHasUpdated(MCM_MOD_ID, _config, MCM_FILE_PATH + "/config.ini")
    _config.load(MCM_FILE_PATH + "/config.ini")

_apply_config(_config)

_mcm_helpers.RegisterConfiguration(
    MCM_MOD_ID,
    "My Mod Name",
    MCM_FILE_PATH,
    "Short description of what this mod does",
    {"config.ini" = _on_mcm_save}
)

func _on_mcm_save(config: ConfigFile):
    _apply_config(config)

# Safe helper — MCM config keys can be null if corrupted or outdated
func _mcm_val(config: ConfigFile, section: String, key: String, fallback):
    var entry = config.get_value(section, key, null)
    if entry == null or not entry is Dictionary:
        return fallback
    return entry.get("value", fallback)

func _apply_config(config: ConfigFile):
    my_setting = _mcm_val(config, "Int", "my_int", my_setting)
    my_key = _mcm_val(config, "Keycode", "my_key", my_key)
    my_choice = _mcm_val(config, "Dropdown", "my_choice", 0)  # Returns index
```

**⚠️ Never use `config.get_value(...)["value"]` directly** — if the key is missing or the config is corrupted, this crashes. Always use a safe helper like `_mcm_val()` above that falls back to the current value.

**Note**: Use `make_dir_recursive()` not `make_dir()` — the MCM parent folder may not exist yet.

### Graceful Fallback

Always support running without MCM using a local `ConfigFile`:

```gdscript
func _ready():
    var mcm = _try_load_mcm()
    if mcm:
        _register_mcm()
    else:
        _load_local_config()
```

## mod.txt Format

```ini
[mod]
name="My Mod Name"
id="my-mod-id"
version="1.0.0"
priority=0
[autoload]
MyAutoload="res://mods/MyMod/Main.gd"
[updates]
modworkshop=12345
```

**Fields**:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Human-readable mod name |
| `id` | Yes | Unique kebab-case identifier |
| `version` | Yes | Semver (e.g., `1.2.3`) |
| `priority` | No | Load order (default `0`). Higher = loads later = sits on top of chain |
| `[autoload]` | Yes | `NodeName="res://path/to/Main.gd"` |
| `[updates]` | No | `modworkshop=<page_id>` for auto-update checking |

**Rules**:
- No blank lines between sections
- No `*` prefix on autoload paths (that's VostokMods-specific, Metro loader doesn't support it)
- `id` should be kebab-case, unique across all mods
- The `[updates]` `modworkshop` value is your **mod page ID** from `modworkshop.net/mod/<id>`
- The mod loader uses `https://api.modworkshop.net/mods/<id>/download` to fetch updates — this downloads the **first file** on your mod page, so make sure your primary VMZ is listed first

## Packaging as VMZ

A VMZ is just a ZIP file with a `.vmz` extension. The structure inside must be:

```
mod.txt
mods/
└── YourModFolder/
    ├── Main.gd
    └── (other .gd files)
```

**Always put files under `mods/YourModName/`** — not at the root. Root-level files like `Scripts/Helper.gd` could collide with the game's own files or other mods.

### PowerShell packaging command

```powershell
# Package
Compress-Archive -Path "_modfiles\my_vmz\*" -DestinationPath "$env:TEMP\My-Mod.zip" -Force
Copy-Item "$env:TEMP\My-Mod.zip" "mods\My-Mod.vmz" -Force

# IMPORTANT: Clear the mount cache after updating a VMZ
$cache = "$env:APPDATA\Road to Vostok\vmz_mount_cache\My-Mod.zip"
if (Test-Path $cache) { Remove-Item $cache -Recurse -Force }
```

The resulting `.vmz` goes in the game's `mods/` folder.

**Mount cache gotcha**: The mod loader caches extracted VMZ contents in `%APPDATA%\Road to Vostok\vmz_mount_cache\`. If you update a VMZ, the game may still load the old cached version. **Always clear the cache** for your mod after repackaging.

## Compatibility Checklist

Before publishing, verify:

- [ ] **No script overrides you don't need** — prefer autoload-only when possible
- [ ] **`super()` called in all overridden lifecycle methods** — `_ready()`, `_process()`, `_physics_process()`, etc.
- [ ] **Files namespaced under `mods/YourMod/`** — no root-level scripts
- [ ] **`get_node_or_null()` used everywhere** — never assume a node exists
- [ ] **Unique names** on injected UI nodes, Engine meta keys, InputMap actions
- [ ] **`mouse_filter = MOUSE_FILTER_IGNORE`** on layout containers
- [ ] **`focus_mode = FOCUS_NONE`** on injected buttons (prevents stealing keyboard focus)
- [ ] **No `Database.gd` override** unless absolutely necessary — it's the most conflict-prone file
- [ ] **Check `modloader_conflicts.txt`** — should show `CHAIN OK` or no conflicts
- [ ] **Test with and without MCM** — fallback config should work
- [ ] **Test alongside popular mods** (ImmersiveXP, etc.)

## Known Gotchas

### 1. `take_over_path()` doesn't update preloaded resources
`gameData` is a `.tres` singleton loaded before mods run. You can read/write its properties but can't replace its script. Work with the properties directly.

### 2. Autoload node paths are wrong
Metro loader places autoloads at `/root/ModLoader/X`, not `/root/X`. Use `Engine.set_meta()`/`Engine.get_meta()` instead of absolute node paths.

### 3. Mouse filter on injected UI
Default `MOUSE_FILTER_STOP` on a Control will block clicks to everything behind it. Set `MOUSE_FILTER_IGNORE` on layout containers; individual buttons will still catch clicks since Button defaults to `MOUSE_FILTER_STOP`.

### 4. No new item resources
VMZ mods can't create new `ItemData` `.tres` resources that the game's item database recognizes. You'd need to override the database loading code, loot tables, trader inventories, slot validation, and the barter system. Virtual/stat-based approaches are much cleaner.

### 5. `super()` warnings
Metro loader warns about overridden `_ready()`/`_process()` not calling `super()`. If you intentionally fully replace a function, this is fine. If you want to extend vanilla behavior, always call `super()`. **When in doubt, call `super()`.**

### 6. Save data location
`user://` = `%APPDATA%\Road to Vostok\`. All mod config, XP data, wallet data, and MCM configs live here. The game log is at `user://logs/godot.log`.

### 7. Modworkshop update detection
The `[updates]` section must use the **mod page ID** (from the URL), not a file ID. If your mod has multiple download files, the API downloads the first one — make sure your primary VMZ is the first file listed.

### 8. Autoloads run before the first scene
`get_tree().current_scene` is `null` during early `_process()` frames. Always null-check before accessing scene nodes. The first scene is "Menu" (main menu); game maps are named "Map".

### 9. VMZ mount cache
The mod loader caches extracted VMZ files at `%APPDATA%\Road to Vostok\vmz_mount_cache\<ModName>.zip\`. After updating a VMZ during development, **delete this cache folder** or the game will load the old version.

### 10. ColorRect headers have no layout
The Inventory and Container headers are `ColorRect` nodes, not `HBoxContainer` or similar. Adding children to them won't auto-position. Use absolute positioning on the parent `Control` node instead.

### 11. MCM Keycode doesn't capture mouse buttons
MCM's `Keycode` config type only captures `InputEventKey` (keyboard). For mouse button bindings, use a `Dropdown` config with options and handle `InputEventMouseButton` in your `_input()` method.

### 12. `hoverGrid` vs `GetHoverGrid()`
`Interface.hoverGrid` is only set when hovering directly over an item. `Interface.GetHoverGrid()` checks mouse position against all visible grid rects — use this for hotkey-based grid detection.

### 13. `super()` in Character._ready() breaks skills
The base `Character._ready()` reinitializes survival stat values. If your override calls `super()` in `_ready()`, it will reset any stat modifications your mod applied. **Do NOT call `super()` in Character._ready()** — it's the one exception to the "always call super" rule. `super()` IS safe in: `Death()`, `Health()`, `Stamina()`, and other per-frame methods.

### 14. `get_script().resource_path` resolves to base game path
In override scripts, `get_script().resource_path` returns `res://Scripts/Foo.gd` (the base game path), NOT your mod's path. **Do not use it to locate mod resources.** Instead, hardcode the path: `"res://mods/YourMod/sounds/click.mp3"` — VMZ files are mounted at `res://mods/<ModName>/`.

### 15. Container grid timing — `call_deferred` is too early
When hooking into `LootContainer.Interact()`, the `containerGrid` on Interface is not yet populated when your override runs. `call_deferred()` is still too early. Use `get_tree().create_timer(0.1).timeout.connect(your_callback)` to give `OpenContainer()` time to set up the grid.

### 16. Loading custom audio (MP3) in mods
You cannot use `load()` or `preload()` for MP3 files in VMZ mods. Load them at runtime:

```gdscript
var file = FileAccess.open("res://mods/YourMod/sounds/click.mp3", FileAccess.READ)
if file:
    var stream = AudioStreamMP3.new()
    stream.data = file.get_buffer(file.get_length())
    var player = AudioStreamPlayer.new()
    player.stream = stream
    get_tree().root.add_child(player)
    player.play()
    player.finished.connect(player.queue_free)
```

Add the `AudioStreamPlayer` to `get_tree().root`, not to UIManager — UIManager's audio may not work reliably from mod context.

### 17. Game loot bucket system
`LootContainer` has three arrays: `commonBucket`, `rareBucket`, `legendaryBucket` (filled in `_ready()` via `FillBuckets()`). To spawn a loot item at runtime:

```gdscript
var item_data = container.commonBucket.pick_random()
var slot = SlotData.new()
slot.itemData = item_data
slot.amount = randi_range(1, item_data.capacity) if item_data.capacity > 1 else 1
slot.condition = randf_range(0.3, 1.0)
interface.Create(slot, interface.containerGrid, true)
```

### 18. `gameData.freeze` controls player input
The game's `Controller._input()` returns early when `gameData.freeze == true`, disabling mouse look and movement. Many other scripts also check it (weapon handling, leaning, NVG, flashlight, etc.). The game's own UI uses this: `UIManager.UIOpen()` sets `freeze = true`, `UIManager.UIClose()` sets `freeze = false`. Use this pattern for custom overlays.

### 19. `ResetCharacter()` clears gameData during death
`Character.Death()` → `Loader.ResetCharacter()` resets **most gameData fields** (health, energy, hydration, xpTotal, freeze, etc.) to defaults **before** the death scene loads. If you need end-of-run values, track them as accumulators during the run (high-water marks for XP, frame-by-frame drain for survival stats). Reading gameData at run end will give you reset values.

### 20. `inventoryGrid.get_child_count()` includes non-item children
The inventory/container grids contain both `Item` nodes and layout/internal children. To count actual items, filter children by checking for `slotData`:
```gdscript
var count = 0
for child in _interface.inventoryGrid.get_children():
    if child.get("slotData") != null:
        count += 1
```
The quick-stack mod uses `child is Item` — both approaches work.

### 21. Controller._ready() recaptures mouse on scene load
`Controller._ready()` calls `Input.set_mouse_mode(Input.MOUSE_MODE_CAPTURED)` when the scene loads. If you have a custom overlay, the game will steal the cursor back. Enforce your desired mouse mode in `_process()` while your overlay is visible.

### 22. Modal/overlay input handling
When building custom UI overlays (modals, panels), the correct architecture to prevent camera movement while keeping buttons clickable:

1. **Use a CanvasLayer** (high layer like 100) so your UI gets GUI events above all game layers
2. **Set `gameData.freeze = true`** to disable camera look and movement via `Controller._input()`
3. **In `_input()`**: block `InputEventMouseMotion` and `InputEventKey`, but **do NOT block `InputEventMouseButton`** — let clicks reach your GUI buttons
4. **In `_unhandled_input()`**: catch leftover mouse clicks so the game doesn't process them
5. **Enforce state in `_process()`** every frame — game scripts may reset freeze/mouse_mode during scene transitions
6. **Use `MOUSE_MODE_CONFINED`** (not `MOUSE_MODE_VISIBLE`) — this matches the game's own `UIManager.UIOpen()` pattern
7. **Always set `gameData.freeze = false` on close** — don't try to restore a previous freeze value, as it may have been captured during a transition when freeze was already true

```gdscript
func _input(event):
    if _modal_visible:
        if event is InputEventKey and event.pressed and not event.echo:
            if event.keycode == KEY_ESCAPE:
                _close_modal()
                get_viewport().set_input_as_handled()
                return
        # Block mouse motion + keyboard, let mouse buttons reach GUI
        if event is InputEventMouseMotion or event is InputEventKey:
            get_viewport().set_input_as_handled()
        return

func _unhandled_input(event):
    if _modal_visible and event is InputEventMouseButton:
        get_viewport().set_input_as_handled()

func _process(_delta):
    if _modal_visible:
        if Input.mouse_mode != Input.MOUSE_MODE_CONFINED:
            Input.mouse_mode = Input.MOUSE_MODE_CONFINED
        if "freeze" in gameData and not gameData.freeze:
            gameData.freeze = true
```

### 23. XP mod tracks its own xpTotal
The XP Skills System mod stores `xpTotal` on its own autoload instance (`Engine.get_meta("XPMain")`), separate from `gameData.xpTotal`. To read XP reliably, check the mod instance first:
```gdscript
func _get_xp_total() -> int:
    var xp_mod = Engine.get_meta("XPMain", null)
    if xp_mod and "xpTotal" in xp_mod:
        return xp_mod.xpTotal
    if "xpTotal" in gameData:
        return gameData.xpTotal
    return 0
```

### 24. Survival stat tracking requires frame-by-frame accumulators
Snapshot-based tracking (start vs end) breaks because: (a) death resets values, and (b) consumption + replenishment cancel out. Track each frame and accumulate decreases separately from increases:
```gdscript
var current_hydration = gameData.hydration
if current_hydration < _acc_last_hydration:
    _acc_hydration_drain += _acc_last_hydration - current_hydration
elif current_hydration > _acc_last_hydration:
    _acc_hydration_restored += current_hydration - _acc_last_hydration
_acc_last_hydration = current_hydration
```
```

## Debugging Tips

### Print to game log
```gdscript
print("[MyMod] Something happened")
# Logs to user://logs/godot.log and stdout
```

### Dump the scene tree
```gdscript
func _dump_tree(node, depth: int = 0):
    print("  ".repeat(depth) + node.name + " (" + node.get_class() + ")")
    for child in node.get_children():
        _dump_tree(child, depth + 1)
```

### Check if a property exists
```gdscript
if "someProperty" in target:
    var value = target.someProperty
```

### Check if a method exists
```gdscript
if obj.has_method("NewMethod"):
    obj.NewMethod()
```

### Check the conflict report
Always check `%AppData%\Road to Vostok Demo\modloader_conflicts.txt` after launching with mods. This tells you if chains are OK, broken, or conflicting.

## Surviving EA Updates

Game updates **will** break mods. Here's what usually goes wrong:

| What Changed | Impact |
|-------------|--------|
| Script renamed/moved | `extends` path breaks → hard crash |
| Method signature changed | `super()` calls fail → hard crash |
| Method renamed | Your override silently does nothing |
| New method added | No impact (you inherit it) |
| Node tree changed | `get_node()` paths break |
| New items in Database.gd | Database replacements miss them |

### Defensive coding

```gdscript
# Check before you access
if "someProperty" in target:
    var value = target.someProperty

# Check before you call
if obj.has_method("NewMethod"):
    obj.NewMethod()

# Never hard-code node paths without a null check
var ui = get_node_or_null("/root/Map/Core/UI")
if ui == null:
    return
```

The fewer scripts you override, the more resilient your mod is. **A mod that touches 2 scripts survives updates way better than one that touches 20.**

### When it breaks

1. Check `modloader_conflicts.txt` for new errors
2. Decompile the updated game with [GDRE Tools](https://github.com/GDRETools/gdsdecomp)
3. Diff the scripts you override against the new versions
4. Fix your overrides, bump your version, push an update
5. Document which game version you tested against in your ModWorkshop description

## File Paths Reference

| Path | Description |
|------|-------------|
| `res://Scripts/` | Vanilla game scripts |
| `res://mods/YourMod/` | Your mod's files (mounted from VMZ) |
| `user://` | `%APPDATA%\Road to Vostok\` |
| `user://XPData.cfg` | XP mod save data |
| `user://WalletData.cfg` | Cash mod save data |
| `user://XPConfig.cfg` | XP mod local config (non-MCM) |
| `user://MCM/<ModID>/config.ini` | MCM config storage |
| `user://modloader.gd` | Metro loader script |
| `user://logs/godot.log` | Game log |
| `%AppData%\Road to Vostok Demo\modloader_conflicts.txt` | Conflict report |
| `%AppData%\Road to Vostok\vmz_mount_cache\` | Extracted VMZ cache (clear when updating) |
| `<game_dir>/override.cfg` | Mod loader bootstrap |
| `<game_dir>/mods/` | VMZ files go here |

## External Resources

- [Vostok Modding Wiki](https://github.com/ametrocavich/vostok-modding-wiki/wiki) — Community wiki with walkthroughs and guides
- [ModWorkshop](https://modworkshop.net/g/road-to-vostok) — Browse and publish mods
- [VostokMods (original loader)](https://github.com/Ryhon0/VostokMods) — The original mod loader
- [GDRE Tools](https://github.com/GDRETools/gdsdecomp) — Decompile the game
- [Godot Docs](https://docs.godotengine.org/en/stable/) — Official engine documentation
