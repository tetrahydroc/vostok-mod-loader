# Road to Vostok — Community Mod Loader

A community-built mod loader for **Road to Vostok** (Godot 4). Adds a launcher UI before the game starts, letting you enable/disable mods, set load order, and check for updates.

---

## Installation

Both files go in the **game installation folder** (next to `RTV.exe`):

```
C:\Program Files (x86)\Steam\steamapps\common\Road to Vostok\
```

1. Copy `override.cfg` and `modloader.gd` into the game folder.

2. Create a `mods` folder inside the game folder if it doesn't exist.

3. Place your `.vmz` mod files inside the `mods` folder.

4. Launch the game normally. The mod loader UI will appear before the main menu.

**That's it.** No AppData setup needed — everything lives in one folder.

---

## Installing Mods

Drop `.vmz` mod files into the `mods` folder. The mod loader finds them automatically on next launch.

If a mod was distributed as a `.zip` file, rename it to `.vmz` before placing it in the mods folder.

---

## Using the Launcher

When you start the game, the mod loader window opens with two tabs:

### Mods
Enable or disable mods with checkboxes. The **priority number** controls load order — higher number loads later and wins when two mods change the same file. The **Load Order** panel on the right shows the final order in real time.

### Updates
Click **Check for Updates** to see if any of your mods have newer versions available on ModWorkshop.

### Settings
Toggle **Developer Mode** to enable the Compatibility tab and verbose conflict logging. Off by default — only needed for mod authors or troubleshooting.

Click **Launch Game** when you're ready to play.

---

## Troubleshooting

If the game crashes or gets stuck after enabling mods:

- **Wait it out.** After 2 failed launches, the mod loader automatically resets to a clean state.
- **Manual reset:** Create an empty file named `modloader_safe_mode` (no file extension) in the game folder. On next launch, the mod loader resets and deletes the file.
- **Full reset:** Delete `override.cfg` from the game folder and replace it with a fresh copy from the mod loader release.

---

## Uninstalling

Delete `override.cfg` and `modloader.gd` from the game folder. The `mods` folder and its contents can be removed separately.

Settings are stored in `%APPDATA%\Road to Vostok\mod_config.cfg` and can be deleted safely.

---

# For Mod Authors

Everything below is for mod developers.

---

## mod.txt Reference

Mod archives must contain a `mod.txt` file at their root:

```ini
[mod]
name="My Mod"
id="my_mod"
version="1.0.0"
priority=0

[autoload]
MyModMain="res://MyMod/Main.gd"

[updates]
modworkshop=12345
```

| Field | Description |
|---|---|
| `name` | Display name shown in the UI (must be quoted) |
| `id` | Unique identifier — duplicates are skipped |
| `version` | Semver string used for update comparison |
| `priority` | Load order number. Higher = loads later = wins. Default 0 |
| `[autoload]` | `Name="res://path/to/script.gd"` — instantiated as a Node after all mods mount |
| `[autoload]` `!` prefix | `Name="!res://path.gd"` — loads **before** game autoloads (see Early Autoloads) |
| `[updates] modworkshop` | ModWorkshop mod ID for update checking |

String values must be quoted. Mods without `mod.txt` will mount but show a warning.

### Supported archive formats

| Format | Notes |
|--------|-------|
| `.vmz` | Road to Vostok's native mod format (renamed zip) |
| `.zip` | Must be renamed to `.vmz` before use |
| `.pck` | Godot PCK — mount only, no mod.txt or autoloads |

### Load priority

Higher number = loads later = wins any file conflict. Default is `0`. Equal priority sorts alphabetically.

Priority controls both which archive's files win *and* which mod's `take_over_path()` executes last — it's the main tool for resolving conflicts between mods that touch the same scripts.

---

## Early Autoloads (Two-Pass Loading)

Most mods load **after** the game's core systems (Loader, Database, Simulation) are already initialized. If your mod needs to run **before** those systems — for example, to modify the shelter list before `Loader._ready()` validates saves — prefix its autoload path with `!`:

```ini
[autoload]
ShelterFix="!res://ShelterMod/Fix.gd"
```

When the mod loader detects `!` prefix autoloads, it:

1. Shows the config UI as usual
2. Writes a temporary `override.cfg` that tells the engine to load these mods first
3. Restarts the game automatically (~5 seconds)
4. On the second launch, mods load before game autoloads — no UI is shown

Mods without `!` are unaffected and never trigger a restart. Only use `!` when your mod genuinely needs to run before game autoloads — most mods don't.

---

## Conflict Report

With Developer Mode enabled, a full conflict log is written to `%APPDATA%\Road to Vostok\modloader_conflicts.txt` after each launch.

| Message | Meaning |
|---------|---------|
| **CONFLICT** | Two mods ship the same file. Last-loaded wins. Adjust priorities. |
| **SCRIPT CONFLICT** | Two mods both `take_over_path()` the same script. Hard incompatibility. |
| **CHAIN OK / CHAIN BROKEN** | Override chain via `super()` — OK means mods stack cleanly, BROKEN means one skips `super()`. |
| **DATABASE OVERRIDE** | A mod replaced `Database.gd`. Normal for overhauls, may block other mods' scene overrides. |
| **OVERHAUL** | 5+ core script overrides. Likely incompatible with other overhaul mods. |
| **NO SUPER** | Lifecycle method override without `super()`. Breaks other mods in the chain. |
| **BAD ZIP** | Backslash file paths in the archive. Re-pack with 7-Zip. |

---

## Best Practices

- **Package as `.vmz`** with forward-slash paths. Use 7-Zip, not .NET `ZipFile.CreateFromDirectory()` (writes backslashes).
- **Include a `mod.txt`** at the archive root. Without it, autoloads won't run.
- **Use `super()` in lifecycle methods.** Skipping it breaks other mods that override the same class.
- **Prefer `extends + super()` over `take_over_path()`** for commonly-overridden scripts. It composes across mods; flat `take_over_path()` doesn't.
- **If you replace Database.gd**, every `preload()` path must exist or the game breaks.
- **`UpdateTooltip()` is inventory-only.** World-item tooltips come from `HUD._physics_process` reading `gameData.tooltip`.
- **Test with other mods installed** and check the conflict report.

---

## VostokMods Compatibility

Mods packaged for [VostokMods](https://github.com/Ryhon0/VostokMods) generally work with this loader.

| Feature | Status |
|---------|--------|
| `.vmz` archives | Supported |
| `mod.txt` format | Supported |
| `[mod] priority` | Supported |
| Filename priority prefix (`100-ModName.vmz`) | Supported |
| `!` early autoload prefix | Supported |

### Features that require VostokMods

VostokMods runs as a separate launcher before Godot starts. This loader runs inside the game, so it cannot:

- **Merge `override.cfg`** — engine settings are read at startup before GDScript runs
- **Register `class_name`** — global class cache is read-only at runtime (use path references instead)
- **Extract native plugins** — GDExtension `.dll`/`.so` files must be on disk at startup

---

## Recovery (Technical Details)

- **Heartbeat file:** `user://modloader_heartbeat.txt` is written at launch and deleted on success. If it persists, the mod loader increments a crash counter. After 2 crashes, it wipes `override.cfg` and all two-pass state.
- **Safe mode:** An empty `modloader_safe_mode` file in the game folder triggers a full reset on next launch.
- **State files:** `user://mod_pass_state.cfg` stores archive paths for the two-pass restart. Deleted after successful Pass 2.

---

## License

MIT License

Copyright (c) 2025

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
