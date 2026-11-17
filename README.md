# OverworldEncounter

Visible wild encounters for Pokémon SDK. Wild creatures walk around the map, and you fight one by talking to it. You choose whether they add to the classic random step encounters or replace them.

- **Version:** 1.3.0.0
- **Author:** ralandel
- **Dependencies:** none

---

## Table of contents

1. [What the plugin does](#what-the-plugin-does)
2. [Installation](#installation)
3. [Uninstallation](#uninstallation)
4. [Configuration](#configuration)
5. [Turning the system on and off](#turning-the-system-on-and-off)
6. [Monkey-patching](#monkey-patching)
7. [Public API reference](#public-api-reference)
8. [Technical notes and limitations](#technical-notes-and-limitations)
9. [Changelog](#changelog)

---

## What the plugin does

When a category is enabled, the plugin **spawns wild creatures as map events** on the system tags of that category, around the player. The creatures come from the wild groups defined in Pokémon Studio for the current zone.

What happens to the **classic random step encounters** depends on `CLASSIC_ENCOUNTER`:

| `CLASSIC_ENCOUNTER` | Classic random encounters |
|---------------------|---------------------------|
| `true` (default)    | Kept everywhere. Overworld creatures are added on top of them. |
| `false`             | Turned off on the tags of the enabled categories. Overworld creatures replace them. |

### The three categories

| Category | Constant          | System tags                                                                 |
|----------|-------------------|-----------------------------------------------------------------------------|
| Water    | `WATER_ENCOUNTER` | `TPond`, `TSea`                                                             |
| Grass    | `GRASS_ENCOUNTER` | `TGrass`, `TTallGrass`                                                      |
| Other    | `OTHER_ENCOUNTER` | Every other tag a wild group can use (`TCave`, `TMount`, `TSand`, `TSnow`, regular ground…) |

When a category is **disabled**, no creature spawns on its tags, and they keep the classic random encounters whatever `CLASSIC_ENCOUNTER` says.

**Tool encounters are never affected.** These are the Old Rod, Good Rod, Super Rod, Rock Smash and Headbutt groups.

### Spawning

- **Same data as classic encounters.** A creature is picked from the group with its `encounter_rate`, so a 1% creature has a 1% chance to be picked at each spawn. The level range, shiny rate, form and extra data all come from Studio. Synchronize, Compound Eyes, Super Luck and the max-level abilities (Hustle, Pressure, Vital Spirit) apply as usual. Repel stops creatures weaker than your lead from spawning.
- **Shiny creatures look shiny on the map.** They use the shiny overworld charset directly.
- **Animated while standing still.** Creatures wander randomly, and they fade in when they spawn and fade out when they despawn.
- **Movement depends on the species.**
  - Move speed follows base Speed: below 50 gives slow (2), 50 to 100 gives normal (3), above 100 gives fast (4).
  - Each creature gets a random move frequency between 2 and 4, so they don't all move at the same moment.
- **They stay in their zone.** A creature only walks on tiles with the same system tag and terrain tag as its group. A creature from a `grass` group never leaves `TGrass`. Water creatures can stand on water.
- **Only around the player.** The active area is the screen plus a margin.
  - Creatures spawn within the screen plus 6 tiles.
  - They are unloaded once they are more than the screen plus 8 tiles away from the player.
  - The event count stays low, even on huge maps.
- **Always populated.** Each category has a **target population**, worked out from how many of its tiles are around the player (see `TARGET_FILL`).
  - On arrival, the whole area is filled at once.
  - Every `REFILL_INTERVAL` seconds (0.5 by default), missing creatures are spawned, up to `SPAWNS_PER_REFILL` each time.
  - Refills try tiles **outside the screen first**, so new creatures appear ahead of the player instead of popping in view.
  - Walking from one zone to another, or coming back to a zone you left, you always find creatures there.
- **Density limits:**
  - Grass: at most 3 creatures in any 5×5 area.
  - Water: at most 5 in any 7×7 area.
  - Other: at most 4 in any 6×6 area.
  - There is also a hard cap per map (`MAX_PER_MAP`, 25 by default).
- **Fair share of the cap.** When the categories together want more than `MAX_PER_MAP`, their targets are scaled down proportionally. If the cap is full and a category is missing creatures (for example grass that just became available), the farthest creatures of a category above its share despawn to make room.
- **Conditional groups.** Groups whose conditions are set in Studio (switches…) are followed live.
  - Every `REFILL_INTERVAL`, the plugin re-evaluates the conditions of the zone's groups.
  - When the set of active groups changes, the groups are reloaded. Classic encounters are reloaded too.
  - The creatures of groups that are no longer active fade out, and the newly active groups start spawning.
  - See `AUTO_REFRESH_GROUPS` and `OverworldEncounter.refresh_encounter_group`.
- **Cheap to compute.**
  - When a map loads, every tile is indexed once by system tag, terrain tag and 4×4 chunk.
  - The candidate tiles around the player are only recomputed when the player enters another chunk, not on every step.
  - Each refill only samples that cached list.
- **Lifetime.** Each creature gets a random lifetime between `LIFETIME_MIN` and `LIFETIME_MAX` seconds (90 to 150 by default), then despawns on its own. The timer only runs while the map is active, so it pauses during battles and menus.
- **When nothing spawns.** Creatures never spawn close to the player, during an event, while the player has no Pokémon able to battle, or while encounters are disabled with the *Change Encounter* event command.

### Battle

Talking to a creature (action button) plays its cry and starts a wild battle. It is exactly the battle the *Wild Battle* common event would start: same transition, same sound effect, same Safari support.

During that battle, **the player counts as standing on the creature's tile**:

- A grass creature gives the grass battle background, and a water creature gives the water background.
- Moves and effects that read the terrain behave the same way (Secret Power, Camouflage, Nature Power…).

The creature is removed from the map after the battle, whether it was caught, defeated or the player fled.

---

## Installation

1. Put `OverworldEncounter.psdkplug` in the `scripts/` folder of your project.
2. Start the game once (from Pokémon Studio or `Game.exe`). The PSDK Plugin Manager then:
   - checks that your PSDK version provides every method the plugin needs;
   - extracts the scripts to `scripts/00000 Plugins/xxxxx OverworldEncounter/`.
3. Done. By default, all three categories are enabled.

> **Do not edit the extracted files.** The Plugin Manager wipes and re-extracts `scripts/00000 Plugins/` every time the plugin list or the PSDK version changes. Put your settings in a project script instead (see [Configuration](#configuration)).

---

## Uninstallation

1. Delete `scripts/OverworldEncounter.psdkplug`.
2. Start the game once. The Plugin Manager removes the extracted scripts.
3. Delete your own settings script if you made one.

The plugin saves nothing in the save file, so existing saves keep working after uninstalling.

---

## Configuration

### All settings

The default values live in `000 Config.rb`:

| Setting                     | Default | Meaning |
|-----------------------------|---------|---------|
| `WATER_ENCOUNTER`           | `true`  | Overworld creatures on `TPond` / `TSea` |
| `GRASS_ENCOUNTER`           | `true`  | Overworld creatures on `TGrass` / `TTallGrass` |
| `OTHER_ENCOUNTER`           | `true`  | Overworld creatures on every other tag |
| `CLASSIC_ENCOUNTER`         | `true`  | Keep the classic random step encounters. `false` turns them off on the tags of the enabled categories |
| `AUTO_REFRESH_GROUPS`       | `true`  | Re-evaluate group conditions every `REFILL_INTERVAL` and reload the groups when they change. `false` = vanilla timing only, plus `refresh_encounter_group` |
| `DISABLE_SWITCH`            | `nil`   | Switch ID that suspends the system while ON (`nil` = no switch) |
| `CATEGORY_TAGS`             | see file | System tag db_symbols for `:water` and `:grass`. Everything else is `:other` |
| `DENSITY`                   | see above | `{ category => { size:, max: } }` |
| `MAX_PER_MAP`               | `25`    | Maximum creatures on one map |
| `TARGET_FILL`               | `0.6`   | Fraction (0.0–1.0) of the maximum density each category aims for on its tiles around the player |
| `REFILL_INTERVAL`           | `0.5`   | Seconds between two refills |
| `SPAWNS_PER_REFILL`         | `2`     | Maximum creatures spawned by one refill (arrival on a map fills everything at once) |
| `LIFETIME_MIN`              | `90`    | Minimum lifetime of a creature, in seconds |
| `LIFETIME_MAX`              | `150`   | Maximum lifetime of a creature, in seconds |
| `FRAMES_PER_SECOND`         | `60`    | Logic frames per second, used to convert the durations |
| `TILE_TRIES`                | `12`    | Random tiles tested per spawn. During refills, the first half only accepts tiles outside the screen |
| `INDEX_CHUNK_SIZE`          | `4`     | Chunk size (tiles) of the spatial index. Candidates are recomputed when the player changes chunk |
| `MIN_PLAYER_DISTANCE`       | `3`     | No spawn within this many tiles of the player |
| `SPAWN_AREA_MARGIN`         | `6`     | Creatures spawn within screen size + this many tiles around the player |
| `UNLOAD_AREA_MARGIN`        | `8`     | Creatures beyond screen size + this many tiles are unloaded. Keep it ≥ `SPAWN_AREA_MARGIN` |
| `SLOW_BASE_SPEED`           | `50`    | Base Speed below this value gives `MOVE_SPEED_SLOW` |
| `FAST_BASE_SPEED`           | `100`   | Base Speed above this value gives `MOVE_SPEED_FAST` |
| `MOVE_SPEED_SLOW`           | `2`     | Event move speed of slow species (1–6) |
| `MOVE_SPEED_NORMAL`         | `3`     | Event move speed of the other species (1–6) |
| `MOVE_SPEED_FAST`           | `4`     | Event move speed of fast species (1–6) |
| `MOVE_FREQUENCY_MIN`        | `2`     | Minimum random move frequency (1–6) |
| `MOVE_FREQUENCY_MAX`        | `4`     | Maximum random move frequency (1–6) |
| `FADE_STEP`                 | `16`    | Opacity change per frame for the fades (`255` = no fade) |
| `EVENT_ID_BASE`             | `100000`| First event ID used by the creatures |

The screen size comes from `Game_Map::NUM_TILE_VIEW_X` / `NUM_TILE_VIEW_Y`, so the active area follows your project's resolution.

**How the target population is computed.** Take a category with `T` tiles in the spawn area around the player:

```
target = round(T × DENSITY.max / DENSITY.size² × TARGET_FILL)   (at least 1 when T > 0)
```

For example, 100 grass tiles with a density of 3 per 5×5 and a `TARGET_FILL` of 0.6 give 100 × 3 / 25 × 0.6 ≈ 7 grass creatures. `MAX_PER_MAP` still caps the total across all categories.

### Changing a setting

Project scripts are loaded **after** plugins. Create a project script anywhere under `scripts/`, outside `00000 Plugins`. Its folder and file names must start with digits. For example:

`scripts/01 Plugin Settings/000 OverworldEncounter Settings.rb`

```ruby
OverworldEncounter.configure(
  CLASSIC_ENCOUNTER: false, # Overworld creatures replace the classic random encounters
  DISABLE_SWITCH: 42,      # Switch 42 suspends the system
  WATER_ENCOUNTER: false,  # No overworld creatures on water (classic encounters there)
  LIFETIME_MIN: 60,
  LIFETIME_MAX: 120,
  MAX_PER_MAP: 15,
  DENSITY: {
    grass: { size: 5, max: 2 },
    water: { size: 7, max: 5 },
    other: { size: 6, max: 4 }
  }
)
```

`configure` raises an error if a setting name is misspelled, so typos are caught at boot.

---

## Turning the system on and off

There are several ways to do it. Pick the one that fits your case.

### 1. With a switch (recommended, saved in the save file)

Set `DISABLE_SWITCH` (see [Configuration](#configuration)). While that switch is **ON**:

- every overworld creature is removed;
- nothing spawns;
- classic random encounters work on every tag (this only matters with `CLASSIC_ENCOUNTER: false`).

Turn the switch OFF and the system resumes on its own: missing creatures come back within the next refills. This is the best option for cutscenes, scripted areas or story phases, because the switch is saved with the game.

### 2. With a script call (temporary, not saved)

In an event, use the *Script* command:

```ruby
OverworldEncounter.suspend   # Same effect as the switch being ON
OverworldEncounter.resume    # Back to normal
```

This flag lives in memory only. It resets when the game restarts, and loading a save does not restore it. Use it for short sequences inside one event.

### 3. Remove the creatures right now, without suspending

```ruby
OverworldEncounter.despawn_all
```

New creatures spawn again with the next refills (unless the system is suspended). This is useful before a cutscene that needs a clear path.

### 4. Pause spawning only (*Change Encounter* event command)

While encounters are disabled with RMXP's *Change Encounter* command (`$game_system.encounter_disabled`):

- **no new creature spawns**;
- creatures already on the map stay until their lifetime ends or they get unloaded.

### 5. Turn one category on or off during the game

`configure` can be called at any time, but its effect is **not saved**:

```ruby
OverworldEncounter.configure(GRASS_ENCOUNTER: false)
```

For a per-save toggle (for example, water creatures only after a story flag), use a switch with a small monkey-patch. See the next section.

### 6. Refresh the wild groups after changing a condition

With `AUTO_REFRESH_GROUPS: true` (the default), there is nothing to do. When a switch used by a group condition changes, the groups are reloaded within half a second.

With `AUTO_REFRESH_GROUPS: false`, the groups are only refreshed like in vanilla PSDK: on map arrival, when coming back from a battle or menu, and when the hour changes. To apply a condition change right away, call this from the event after changing the switch:

```ruby
OverworldEncounter.refresh_encounter_group
```

It reloads the groups for both overworld creatures and classic encounters. The creatures of groups that are no longer active fade out.

### 7. Turn the plugin off for good

- Set the three category constants to `false` with `configure`, **or**
- uninstall the plugin.

With every category disabled, the plugin does nothing and the game behaves like vanilla PSDK.

---

## Monkey-patching

All the logic lives in the `OverworldEncounter` module (module methods). The cleanest way to change its behaviour is to `prepend` a module to `OverworldEncounter.singleton_class` from a project script. Call `super` to keep the original behaviour, and it will survive plugin updates.

### Toggle a category with a switch (saved)

```ruby
module OverworldEncounterSwitchCategories
  # Water creatures only once switch 50 is ON; grass creatures off while switch 51 is ON
  def category_enabled?(category)
    return false if category == :water && !$game_switches[50]
    return false if category == :grass && $game_switches[51]

    return super
  end
end
OverworldEncounter.singleton_class.prepend(OverworldEncounterSwitchCategories)
```

`category_enabled?` drives both spawning and the blocking of random encounters (with `CLASSIC_ENCOUNTER: false`), so the two always stay consistent.

### Suspend the system on some maps

```ruby
module OverworldEncounterMapBlacklist
  NO_OVERWORLD_MAPS = [12, 45, 46]

  def suspended?
    return true if NO_OVERWORLD_MAPS.include?($game_map.map_id)

    return super
  end
end
OverworldEncounter.singleton_class.prepend(OverworldEncounterMapBlacklist)
```

### Move a system tag to another category

```ruby
OverworldEncounter.configure(
  CATEGORY_TAGS: {
    water: %i[pond sea under_water], # Underwater tiles now follow WATER_ENCOUNTER
    grass: %i[grass tall_grass]
  }
)
```

### Stop some groups from spawning (they keep classic encounters)

```ruby
module OverworldEncounterGroupFilter
  def spawnable_group?(group)
    return false if group.db_symbol == :legendary_cave_group

    return super
  end
end
OverworldEncounter.singleton_class.prepend(OverworldEncounterGroupFilter)
```

> With `CLASSIC_ENCOUNTER: false`, the random encounter blocker works per tag, not per group. If you filter a group out, that tag also needs to go back to classic encounters, otherwise the group will never be met. To do that, also override `blocks_random_encounter?`.

### Change the active area shape

`around_player?(x, y, margin)` decides both the spawn area and the unload area. For example, to use a circle instead of the screen rectangle:

```ruby
module OverworldEncounterCircleArea
  def around_player?(x, y, margin)
    radius = 10 + margin
    dx = x - $game_player.x
    dy = y - $game_player.y
    return dx * dx + dy * dy <= radius * radius
  end
end
OverworldEncounter.singleton_class.prepend(OverworldEncounterCircleArea)
```

### Custom move speed per species

`move_speed_for(pokemon)` is a class method of `OverworldEncounter::OverworldEvent`:

```ruby
module OverworldEncounterCustomSpeed
  def move_speed_for(pokemon)
    return 5 if pokemon.db_symbol == :ninjask

    return super
  end
end
OverworldEncounter::OverworldEvent.singleton_class.prepend(OverworldEncounterCustomSpeed)
```

### Change how creatures are generated

`generate_creature(group)` is private, but you can still override it with `prepend`. Keep your override private as well:

```ruby
module OverworldEncounterNoShinyOnRoute1
  private

  def generate_creature(group)
    creature = super
    return nil if creature&.shiny? && $game_map.map_id == 5 # Reject shinies on map 5

    return creature
  end
end
OverworldEncounter.singleton_class.prepend(OverworldEncounterNoShinyOnRoute1)
```

The other private methods you can override the same way:

- `spawn_tile_valid?(x, y, category)`
- `density_ok?(x, y, category)`
- `pick_encounter(encounters)`
- `can_spawn_now?`
- `unload_far_entries`
- `category_targets(categories)` (change how many creatures each category aims for)
- `most_needed_category(targets, exhausted)`
- `pick_group(groups)`

### Change the creature event itself

The creature events are instances of `OverworldEncounter::OverworldEvent`, a subclass of `Game_Event`. Reopen it or prepend to it like any class. For example, to make creatures walk through each other:

```ruby
module OverworldEventCustom
  def passable?(x, y, d, skip_event = false)
    return super(x, y, d, true) # Ignore other events, still restricted to the group's tiles
  end
end
OverworldEncounter::OverworldEvent.prepend(OverworldEventCustom)
```

---

## Public API reference

### Script calls

| Method | Description |
|--------|-------------|
| `OverworldEncounter.configure(**settings)` | Override settings from `000 Config.rb` |
| `OverworldEncounter.suspend` / `.resume` | Suspend or resume the system (not saved) |
| `OverworldEncounter.suspended?` | `true` if suspended by the switch, by script, or because every category is off |
| `OverworldEncounter.despawn_all` | Remove every creature on the current map |
| `OverworldEncounter.refresh_encounter_group` | Reload the zone's wild groups according to their current conditions (classic encounters included) |

### Methods meant to be overridden

| Method | Description |
|--------|-------------|
| `OverworldEncounter.group_conditions_met?(group)` | Whether a group's Studio conditions are met (same evaluation as `load_groups`) |
| `OverworldEncounter.encounter_groups_outdated?` | Whether the active groups of the zone differ from the loaded ones |
| `OverworldEncounter.category_of(system_tag)` | `:water`, `:grass` or `:other` for a system tag db_symbol |
| `OverworldEncounter.category_enabled?(category)` | Whether a category is handled by the plugin |
| `OverworldEncounter.blocks_random_encounter?(system_tag)` | Whether classic encounters are blocked on that tag |
| `OverworldEncounter.spawnable_group?(group)` | Whether a `Studio::Group` may spawn creatures |
| `OverworldEncounter.tile_matches_group?(x, y, group)` | Whether a tile belongs to a group's zone |
| `OverworldEncounter.around_player?(x, y, margin)` | Whether a tile is within screen size + `margin` tiles of the player |
| `OverworldEncounter::OverworldEvent.move_speed_for(pokemon)` | Move speed of a creature event |

### Called by the creature event

| Method | Description |
|--------|-------------|
| `OverworldEncounter.interact(event_id)` | Plays the cry and starts the battle (used by the creature event) |

### Engine hooks

The plugin hooks into the engine with `prepend` only:

- `PFM::Wild_Battle#group_encounter_detected?`
- `PFM::Wild_Battle#load_groups`
- `Game_Map#setup`
- `Game_Map#update`
- `Game_Player#system_tag` (overridden only during an overworld battle)
- `Spriteset_Map`: two new methods, `ow_add_character` and `ow_remove_character`

---

## Technical notes and limitations

- **Nothing is saved.** Creatures are temporary. After loading a save, the area around the player fills up again with new creatures.
- **Roaming Pokémon** use the same trigger as random encounters. With `CLASSIC_ENCOUNTER: false`, they cannot be met on tags handled by the plugin.
- **Group refresh and step counter.** Reloading the groups (after a condition change) also resets PSDK's classic encounter step counter, exactly like vanilla does on each group load.
- **`:custom_all` groups** (map-wide forced groups) are left alone. They neither spawn nor get blocked.
- **Encounter-rate abilities** (Static, Magnet Pull, Cute Charm, Flash Fire, Storm Drain, Harvest, Intimidate/Keen Eye) do not change which creature spawns.
- **First spawn on a map.** The plugin scans every tile of the map once to build its spatial index. That takes a few dozen milliseconds on a 200×120 map, and the index is then reused until the map changes.
- **MapLinker.** Creatures only spawn inside the current map, never on linked maps.
- **Compatible with the PSDK event culling** (Events Optimizer) and with any plugin that also uses `prepend` on the same classes.
- **Event IDs** start at `EVENT_ID_BASE` (100000) so they never collide with map events.

---

## Changelog

### 1.3.0.0

- **Added: `CLASSIC_ENCOUNTER`** (default `true`). Choose whether classic random step encounters are kept alongside overworld creatures (`true`) or turned off on the enabled categories (`false`).
- **Added: live conditional groups.**
  - The conditions of the zone's wild groups (switches…) are re-evaluated every `REFILL_INTERVAL`.
  - When they change, the groups are reloaded for both systems.
  - Creatures of groups that are no longer active fade out.
  - This can be turned off with `AUTO_REFRESH_GROUPS`.
- **Added: `OverworldEncounter.refresh_encounter_group`** to force a group reload from an event.
- **Fixed:** a newly active category could not spawn while another category already filled `MAX_PER_MAP`. Targets are now scaled to share the cap fairly, and creatures above their category's share make room.
- **⚠️ Default behaviour change:** classic random encounters are now kept by default. Set `CLASSIC_ENCOUNTER: false` to get the 1.2 behaviour back.

### 1.2.0.0

- **Changed: spawning is driven by a target population.**
  - Each category aims for a population computed from its tiles around the player (`TARGET_FILL`).
  - Missing creatures are spawned every `REFILL_INTERVAL` (0.5 s), up to `SPAWNS_PER_REFILL` each time.
  - Arriving on a map fills the whole area at once.
  - New areas, and areas you come back to, are no longer empty.
- **Added: refills prefer tiles outside the screen**, so new creatures appear ahead of the player.
- **Optimized: spatial tile index.** Tiles are indexed by tag and by chunk (`INDEX_CHUNK_SIZE`). The candidate tiles around the player are cached and only recomputed when the player enters another chunk.
- **Removed:** `CHECK_INTERVAL`, `INITIAL_SPAWN_ATTEMPTS`, `SPAWN_ATTEMPTS_PER_CHECK`, `SPAWN_ATTEMPTS_ON_DESPAWN` and `SPAWN_CHANCE`. They are replaced by `TARGET_FILL`, `REFILL_INTERVAL` and `SPAWNS_PER_REFILL`. Update your `configure` call if it used them.
- **Changed:** `TILE_TRIES` now defaults to 12.

### 1.1.0.0

- **Added: active area.**
  - Creatures only spawn within the screen plus `SPAWN_AREA_MARGIN` tiles (6) around the player.
  - They are unloaded beyond the screen plus `UNLOAD_AREA_MARGIN` tiles (8).
  - Each unload triggers a spawn near the player.
- **Added: move speed by base Speed.** Below 50 gives 2, 50 to 100 gives 3, above 100 gives 4.
- **Added: random move frequency** between 2 and 4, plus a random initial delay, so creatures don't move in sync.
- **Changed: random lifetime.** It is now between `LIFETIME_MIN` and `LIFETIME_MAX` (90–150 s). This replaces the fixed `LIFETIME`.
- **Changed:** `MAX_PER_MAP` now defaults to 25.
- **Removed:** `LIFETIME`, `MOVE_SPEED` and `MOVE_FREQUENCY`. They are replaced by the settings above. Update your `configure` call if it used them.

### 1.0.1.0

- **Fixed:** a crash (`undefined method 'unlock' for nil`) after defeating or fleeing from an overworld creature. The creature's event was removed before the interpreter had finished running it. The creature is now hidden as soon as the battle ends and removed once its event is done.
- **Added:** the creature plays its cry when the player interacts with it.

### 1.0.0.0

- Initial release.
