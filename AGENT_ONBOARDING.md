# AI Agent Onboarding — Road to Vostok Modding

Read this when you're an AI agent asked to help mod Road to Vostok. It gives you the workflow, conventions, and landmines so you don't repeat mistakes we've already learned from.

## Context

- Game: Road to Vostok (Godot 4.6.1 EA)
- Mod loader: Metro/Community Mod Loader (tracks override chains, emits `CHAIN BROKEN` / `NO SUPER` warnings)
- Project layout: each mod lives in its own repo under `D:\Projects\vostok-<name>\`

## Paths you'll need

| Path | Purpose |
|---|---|
| `D:\Steam\steamapps\common\Road to Vostok\` | Game install |
| `D:\Steam\steamapps\common\Road to Vostok\mods\` | Deployed `.vmz` files |
| `C:\Users\<user>\AppData\Roaming\Road to Vostok\` | Save files + user data |
| `C:\Users\<user>\AppData\Roaming\Road to Vostok\logs\` | Godot logs (`godot.log` = current, timestamped files = rotations) |
| `D:\Projects\vostok-decompiled\` | Decompiled base game source — read-only reference |
| `D:\Projects\vostok-mods\MODDING_GUIDE.md` | **READ FIRST.** Full modding reference we maintain, ~1000 lines of hard-won context |
| `D:\Projects\vostok-cash\mods\CashSystem\Main.gd` | Canonical reference for runtime ItemData + pickup scene creation |
| https://github.com/ametrocavich/vostok-modding-wiki/wiki | Community modding wiki |

Other mods in my orbit: XP Skills, Cash System, Run Summary, Quick Stack & Sort, Secure Container, Vostok AI, Echoes of Vostok, Debug Tweaker, Item Spawner.

## Modding conventions (hard rules, read `MODDING_GUIDE.md` for nuance)

### Two patterns
- **Script override** via `overrideScript("res://Scripts/Foo.gd")` in an autoload's `_ready`.
- **Autoload-only** (UI injection + polling). Compat-safer — zero chain risk.

Prefer autoload-only when possible.

### `super()` chain rules
- Call `super()` in lifecycle methods (`_ready`, `_process`, `_physics_process`) and any overridden game method. Missing `super()` = `CHAIN BROKEN` warning + silently drops earlier mods' logic.
- **One exception**: never call `super()` in `Character._ready()` — base re-initializes survival stats and wipes your mod's setup.
- When your override modifies a stat/bound, pick the right pattern (see gotcha #25 in `MODDING_GUIDE.md`):
  - **TIGHTEN** (reduce drain, lower cap) → call `super()` first, then modify the result. Canonical wiki pattern.
  - **DELTA-SCALABLE** (base formula is `delta * x` with no xp term) → scale `delta` before `super`.
  - **LOOSEN** (higher cap, extra regen) → full replacement. `super`'s `clampf` can't be undone by a post-super `clampf`.
  - **Never** inject-super-restore on `gameData` — it's a preloaded `Resource` shared by every script (`preload("res://Resources/GameData.tres")` returns the same instance). Mutations across `super()` are a global side channel; `Loader.SaveCharacter()` firing during the injection window persists the temporary value to `Character.tres`. We burned two releases learning this.

### `gameData` gotchas
- Preloaded once, shared across all scripts. Treat as global singleton state.
- `gameData.xp<Stat>` (xpHealth, xpStamina, etc.) are the base game's own xp fields — they're read by base game code for maxHP / drain multipliers. Our mods sync their own skill state into these fields via `_sync_to_gamedata()` every frame so base-game code picks up our values.

### Custom items
- `Database.get(file)` returns `null` for mod-created items. Base game silently deletes items it can't resolve:
  - `Interface.Drop` calls `queue_free` — override to route your own items before falling through to `super`.
  - `Interface.ContextPlace` — same.
  - `Loader.LoadShelter` — prints "File missing" and skips. Compensate by re-instantiating your items ~0.3s after scene change (detect via `mapName` field flipping).
- `LootSimulation.SpawnItems` early-returns (not continue!) on unknown items — if your item rolls FIRST in a spawn batch, the rest of the batch is dropped. Low impact but worth knowing.
- Defensive `add_to_group("Item")` after instantiate — belt-and-braces in case the group tag gets lost.

### Runtime ItemData creation (Cash-mod pattern)
- Save `ImageTexture` to `user://` as `.tres`, then call `take_over_path(path)` on the in-memory instance so `ItemData` serializes the icon as a cheap `ext_resource` reference instead of embedding the full bitmap inline (~500 KB per item otherwise).
- Reference vanilla meshes via `ResourceLoader.load(obj_path)` — don't try to `FileAccess.open()` raw `.obj` files; Godot strips source in packaged builds.
- Pickup `.tscn` is written as plaintext from GDScript — forward-slash `[ext_resource path=...]`, never hardcode UIDs (they break across game updates).

## Packaging

- Use the Python repack script: `python _tools/repack.py` from the repo root.
- **Never** use PowerShell's `Compress-Archive` — it writes backslash path separators in zip entries, which silently break Godot's resource loader. The mod appears to load but nothing inside it resolves.
- `.vmz` is gitignored. The repo root `.vmz` is a convenience artifact; the real deploy target is `D:\Steam\steamapps\common\Road to Vostok\mods\<Name>.vmz`.
- Game must be closed when overwriting the deployed `.vmz` (file lock).

## Git workflow

- **Never push directly to `main`.** Create a branch: `fix/<name>`, `feat/<name>`, `docs/<name>`, `refactor/<name>`.
- Commit → push the branch → share the PR URL. The human merges via GitHub web UI.
- **No AI attribution in commits.** No "Claude", no "Co-Authored-By: Claude Opus X.Y", no "🤖 Generated with". Commit as the repo owner.
- Version bumps: edit `mod.txt` and add a matching `### vX.Y.Z` section at the top of `CHANGELOG.md` in the same commit as the code change. Keep the changelog entry user-facing first (lead with "Fixed: X" or "New: Y"), follow with the why/how.
- Credit community contributors by name when one suggested a feature or reported a bug. See existing CHANGELOG for tone.
- After the human merges, tag: `git tag vX.Y.Z HEAD -m "..."; git push origin vX.Y.Z`. The tag push triggers a GitHub Action that builds the release notes from `CHANGELOG.md`.
- `description.md` is the Modworkshop description. It is **not** bundled in the `.vmz`. Update separately when user-visible features change.

## Debugging

- Logs: `C:\Users\<user>\AppData\Roaming\Road to Vostok\logs\godot.log` (current session) + rotated files.
- Filter for real errors: `grep -iE "SCRIPT ERROR|Parse Error|Failed to load|push_error"`. Ignore `InputMap`/`UID`/`using text path` noise — those are cosmetic.
- **If a feature vanishes with no crash**: check for parse errors on the relevant override file. Godot silently drops entire overrides when parsing fails (e.g., calling `super(delta)` in `_process` when base has no `_process`).
- Known unfixable noise: `CHAIN BROKEN` on `_process` for `Interface.gd` (base has no `_process`, so any `super(delta)` call fails to parse — we accept the warning).

## When I ask for exploratory input

- Give a 2-3 sentence recommendation with the main tradeoff. Don't jump to implementation until I agree.
- For scope questions ("should this be one feature or split?"), lean toward smaller shipped increments and say so explicitly.
- If you're unsure which mod repo I'm working in, ask before starting.

## Pattern examples to crib from

- Runtime item creation: `D:\Projects\vostok-cash\mods\CashSystem\Main.gd`
- Override chaining done right: `D:\Projects\vostok-skills\mods\XPSkillsSystem\Character.gd` (see `Energy`, `Hydration`, `Mental`, `Stamina` for the TIGHTEN pattern; `Clamp` for the full-replacement case)
- MCM integration: any mod's `Main.gd`, search for `_register_mcm` / `_apply_mcm_config`
- Shelter persistence: `XP Skills` or `Cash System`'s `_respawn_*_in_shelter` functions
