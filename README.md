# Alfred the Butler — v1.1.1

## Installation

1. Download `SteroidAlfredV2-1.1.1.pack` from the [latest release](https://github.com/OldOnSteroid/-QQT-SteroidAlfred/releases/latest)
2. Place it directly in your `qqt/scripts/` folder
3. Do **not** put it inside a subfolder — it must sit at the root of `scripts/`

Alfred will appear in the QQT plugin list on next inject.

> **First load notice:** On the very first load (and whenever Alfred's item data is updated), there will be a **3–5 second delay** at startup while Alfred downloads its affix/item database in the background. This only happens once — all subsequent loads are instant. An internet connection is required for this one-time download.

---

Alfred handles all town management automatically: selling, salvaging, stashing, repairing, restocking, gambling, and teleporting back out. External farming scripts integrate with Alfred via a simple one-call API.

---

## How it works

Alfred scans the player's inventory every 0.5s and updates an internal `need_trigger` flag. This flag becomes true when any of the following conditions are met:

- Main inventory is full (`>= max_inventory` setting)
- Talisman inventory is full
- Socketable/consumable/key bags are full (when set to FULL mode)
- Any equipped item durability is <= 10
- Restock items fall below their minimum threshold

When `need_trigger` is true, Alfred expects an external script to hand it control. Alfred then teleports to town, runs the full task chain, and fires a callback when done.

---

## Task chain

Alfred runs the following tasks in order on each town visit:

| # | Task | NPC | Notes |
|---|------|-----|-------|
| 1 | **Sell** | Vendor | Sells items marked for selling by the loot filter |
| 2 | **Salvage** | Blacksmith | Salvages items marked for salvaging; also handles non-favourited dungeon sigils |
| 3 | **Salvage Talisman** | Occultist | Salvages seals and charms based on configured action |
| 4 | **Repair** | Blacksmith | Repairs all gear when any piece is below 10% durability |
| 5 | **Stash** | Stash | Stashes everything not marked to sell or salvage; handles keys, sigils, socketables, talismans |
| 6 | **Gamble** | Curiosity Vendor | Spends obols on the configured item category until below the obol threshold |
| 7 | **Stash Pull** | Stash | Pulls specific items from stash if configured |
| 8 | **Teleport** | — | Teleports back out when done |

---

## Integrating Alfred into your script

Add Alfred's task as **first priority** in your task list using `create_task`. The task names below (`run`, `fight`, etc.) are examples from your own script — replace them with whatever tasks your script actually has.

```lua
local alfred = AlfredTheButlerPlugin or PLUGIN_alfred_the_butler

local tasks = {
    alfred.create_task('my_script_name', function()
        -- called when Alfred finishes the town run
        -- reset your script state here
        -- e.g. clear run flags, force teleport back to dungeon, etc.
    end),
    require('tasks.my_run_task'),    -- your script's tasks go here
    require('tasks.my_fight_task'),
    -- ... rest of your tasks
}
```

Then in your update loop, iterate tasks in order — first `shouldExecute()` that returns true runs `Execute()`:

```lua
for _, task in ipairs(tasks) do
    if task.shouldExecute() then
        task.Execute()
        break
    end
end
```

Alfred's task is first in the list, so when `need_trigger` is true it intercepts before any of your tasks run. Your script is effectively paused until Alfred calls your callback.

---

## API reference

All functions are available on the global `AlfredTheButlerPlugin` (or `PLUGIN_alfred_the_butler`).

### `create_task(caller, on_done)` — recommended integration
Returns a task object (with `shouldExecute` and `Execute`) ready to drop into your task list. Handles all polling and state internally.

| Parameter | Type | Description |
|-----------|------|-------------|
| `caller` | string | Your script's label, used for logging |
| `on_done` | function | Called when Alfred completes the town run |

---

### `get_status()` — read Alfred's current state
Returns a table with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | bool | Whether Alfred is enabled in settings |
| `need_trigger` | bool | True when a town run is needed — the main flag to watch |
| `inventory_full` | bool | Main inventory >= max_inventory setting |
| `talisman_inventory_full` | bool | Talisman bag >= max_inventory setting |
| `need_repair` | bool | Any equipped item durability <= 10 |
| `inventory_count` | number | Current main inventory item count |
| `sell_count` | number | Number of items queued to sell |
| `salvage_count` | number | Number of items queued to salvage |
| `stash_count` | number | Number of items queued to stash |
| `restock_count` | number | Number of restock items below minimum |
| `trigger_tasks` | bool | Alfred is currently running the task chain |
| `all_task_done` | bool | All tasks completed this cycle |

---

### `trigger_tasks_with_teleport(caller, callback)` — manual trigger
Tells Alfred to run the full town task chain including teleport. Use this if you need direct control instead of `create_task`.

---

### `pause(caller)` / `resume()` — movement control
Pause or resume Alfred's movement control. Alfred pauses Batmobile automatically during task runs.

---

## Gambling

Alfred can automatically spend obols at the Curiosity Vendor after each town run.

- Enable gambling in the **Gambling** section of the menu
- Set an **obol threshold** — Alfred gambles until obols drop below this value
- Select the **item category** to gamble on (e.g. Helm, Chest, Boots)

When inventory fills up mid-gamble, Alfred automatically pauses, runs sell → salvage → stash to clear space, then resumes gambling until obols are spent.

> **Note:** Gamble categories are currently confirmed on **Sorcerer** only. If you're on a different class, please use the **Dump Vendor Items** button in the menu while the Gambler is open, and paste the console output in Discord so we can map your class correctly.

---

## Sigil Salvage

Alfred can automatically salvage non-favourited Nightmare Dungeon Sigils at the Blacksmith.

Enable **Salvage Non-Favourited Sigils** in the Dungeon Keys section of the menu. Only sigils not marked as favourited (locked) will be salvaged. Favourited sigils can optionally be stashed instead.

---

## Talisman / Seal / Charm Salvage

Alfred handles seals and charms at the Occultist separately from regular salvage. Configure the action (sell, salvage, or stash) for each type independently in the Talisman section of the menu.

---

## Changelog

### v1.1.1
- **Affix data** now auto-downloads from GitHub on first load — no manual file placement needed
- **Auto-updates** when item data changes, triggered silently on next load
- **Smaller pack size**, improved startup performance

### v1.1.0
- **Gambling** — full obol-spending loop with automatic inventory management mid-cycle
- **Salvage reliability** — rewrote completion detection to scan inventory directly instead of a cached counter; eliminates false-positive failures
- **Salvage smoothness** — removed artificial delays; salvage now runs as fast as talisman salvage
- **Sigil salvage** — non-favourited dungeon sigils now correctly salvaged at the Blacksmith via API; `S05_` prefixed BSK sigils excluded as they are not salvageable
- **Talisman salvage** — seal and charm salvage at the Occultist runs cleanly with no timing issues
- **Stash** — transmutation section removed from public build
- **UI** — version number now visible inside the Alfred menu panel
