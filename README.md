# OverworldEncounter

Visible wild encounters for Pokémon SDK. Wild creatures walk around the map, and you fight one by talking to it. They replace the random step encounters.

- **Version:** 1.0.0.0
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

---

## What the plugin does

When a category is enabled, the plugin does two things:

1. It **turns off random step encounters** on the system tags of that category.
2. It **spawns wild creatures as map events** on those tags. The creatures come from the wild groups defined in Pokémon Studio for the current zone.

### The three categories

| Category | Constant          | System tags                                                                 |
|----------|-------------------|-----------------------------------------------------------------------------|
| Water    | `WATER_ENCOUNTER` | `TPond`, `TSea`                                                             |
| Grass    | `GRASS_ENCOUNTER` | `TGrass`, `TTallGrass`                                                      |
| Other    | `OTHER_ENCOUNTER` | Every other tag a wild group can use (`TCave`, `TMount`, `TSand`, `TSnow`, regular ground…) |

When a category is **disabled**, its tags keep the classic random encounters.

**Tool encounters are never affected.** These are the Old Rod, Good Rod, Super Rod, Rock Smash and Headbutt groups.

### Spawning

- **Same data as classic encounters.** A creature is picked from the group with its `encounter_rate`, so a 1% creature has a 1% chance to be picked at each spawn. The level range, shiny rate, form and extra data all come from Studio. Synchronize, Compound Eyes, Super Luck and the max-level abilities (Hustle, Pressure, Vital Spirit) apply as usual. Repel stops creatures weaker than your lead from spawning.
- **Shiny creatures look shiny on the map.** They use the shiny overworld charset directly.
- **Animated while standing still.** Creatures wander randomly, and they fade in when they spawn and fade out when they despawn.
- **They stay in their zone.** A creature only walks on tiles with the same system tag and terrain tag as its group. A creature from a `grass` group never leaves `TGrass`. Water creatures can stand on water.
- **Density limits:**
  - Grass: at most 3 creatures in any 5×5 area.
  - Water: at most 5 in any 7×7 area.
  - Other: at most 4 in any 6×6 area.
  - There is also a hard cap per map (`MAX_PER_MAP`).
- **When they spawn:**
  - When the player arrives on a map.
  - Every `CHECK_INTERVAL` seconds (10 by default).
  - Right after a creature despawns.
- **Lifetime.** A creature despawns on its own after `LIFETIME` seconds (120 by default). The timer only runs while the map is active, so it pauses during battles and menus.
- **When nothing spawns.** Creatures never spawn close to the player, during an event, while the player has no Pokémon able to battle, or while encounters are disabled with the *Change Encounter* event command.

### Battle

Talking to a creature (action button) starts a wild battle. It is exactly the battle the *Wild Battle* common event would start: same transition, same sound effect, same Safari support.

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
| `DISABLE_SWITCH`            | `nil`   | Switch ID that suspends the system while ON (`nil` = no switch) |
| `CATEGORY_TAGS`             | see file | System tag db_symbols for `:water` and `:grass`. Everything else is `:other` |
| `DENSITY`                   | see above | `{ category => { size:, max: } }` |
| `MAX_PER_MAP`               | `50`    | Maximum creatures on one map |
| `CHECK_INTERVAL`            | `10`    | Seconds between two spawn checks |
| `LIFETIME`                  | `120`   | Seconds before a creature despawns |
| `FRAMES_PER_SECOND`         | `60`    | Logic frames per second, used to convert the durations |
| `INITIAL_SPAWN_ATTEMPTS`    | `10`    | Spawn attempts when arriving on a map |
| `SPAWN_ATTEMPTS_PER_CHECK`  | `3`     | Spawn attempts at each periodic check |
| `SPAWN_ATTEMPTS_ON_DESPAWN` | `1`     | Spawn attempts triggered by each despawn |
| `SPAWN_CHANCE`              | `70`    | % chance that a periodic attempt spawns something |
| `TILE_TRIES`                | `8`     | Random tiles tested per attempt |
| `MIN_PLAYER_DISTANCE`       | `3`     | No spawn within this many tiles of the player |
| `MOVE_SPEED`                | `3`     | Event move speed (1–6) |
| `MOVE_FREQUENCY`            | `3`     | Event move frequency (1–6) |
| `FADE_STEP`                 | `16`    | Opacity change per frame for the fades (`255` = no fade) |
| `EVENT_ID_BASE`             | `100000`| First event ID used by the creatures |

### Changing a setting

Project scripts are loaded **after** plugins. Create a project script anywhere under `scripts/`, outside `00000 Plugins`. Its folder and file names must start with digits. For example:

`scripts/01 Plugin Settings/000 OverworldEncounter Settings.rb`

```ruby
OverworldEncounter.configure(
  DISABLE_SWITCH: 42,      # Switch 42 suspends the system
  WATER_ENCOUNTER: false,  # Keep classic random encounters on water
  LIFETIME: 90,
  MAX_PER_MAP: 25,
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
- **classic random encounters come back** on every tag.

Turn the switch OFF and the system resumes on its own, with new creatures at the next check. This is the best option for cutscenes, scripted areas or story phases, because the switch is saved with the game.

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

New creatures will spawn again at the next check. This is useful before a cutscene that needs a clear path.

### 4. Pause spawning only (*Change Encounter* event command)

While encounters are disabled with RMXP's *Change Encounter* command (`$game_system.encounter_disabled`):

- **no new creature spawns**;
- creatures already on the map stay until their lifetime ends.

### 5. Turn one category on or off during the game

`configure` can be called at any time, but its effect is **not saved**:

```ruby
OverworldEncounter.configure(GRASS_ENCOUNTER: false)
```

For a per-save toggle (for example, water creatures only after a story flag), use a switch with a small monkey-patch. See the next section.

### 6. Turn the plugin off for good

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

`category_enabled?` drives both spawning and the blocking of random encounters, so the two always stay consistent.

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

> The random encounter blocker works per tag, not per group. If you filter a group out, that tag also needs to go back to classic encounters, otherwise the group will never be met. To do that, also override `blocks_random_encounter?`.

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

### Play the creature's cry before the battle

```ruby
module OverworldEncounterCry
  def interact(event_id)
    entry = OverworldEncounter.instance_variable_get(:@entries)[event_id]
    Audio.se_play(entry.pokemon.cry) if entry && !entry.despawning

    super
  end
end
OverworldEncounter.singleton_class.prepend(OverworldEncounterCry)
```

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

### Methods meant to be overridden

| Method | Description |
|--------|-------------|
| `OverworldEncounter.category_of(system_tag)` | `:water`, `:grass` or `:other` for a system tag db_symbol |
| `OverworldEncounter.category_enabled?(category)` | Whether a category is handled by the plugin |
| `OverworldEncounter.blocks_random_encounter?(system_tag)` | Whether classic encounters are blocked on that tag |
| `OverworldEncounter.spawnable_group?(group)` | Whether a `Studio::Group` may spawn creatures |
| `OverworldEncounter.tile_matches_group?(x, y, group)` | Whether a tile belongs to a group's zone |

### Called by the creature event

| Method | Description |
|--------|-------------|
| `OverworldEncounter.interact(event_id)` | Starts the battle (used by the creature event) |

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

- **Nothing is saved.** Creatures are temporary. After loading a save, the map fills up again with new creatures.
- **Roaming Pokémon** use the same trigger as random encounters, so they cannot be met on tags handled by the plugin.
- **`:custom_all` groups** (map-wide forced groups) are left alone. They neither spawn nor get blocked.
- **Encounter-rate abilities** (Static, Magnet Pull, Cute Charm, Flash Fire, Storm Drain, Harvest, Intimidate/Keen Eye) do not change which creature spawns.
- **First spawn on a map.** The plugin scans every tile of the map once. This is instant on normal maps, but very large maps may hitch briefly the first time.
- **MapLinker.** Creatures only spawn inside the current map, never on linked maps.
- **Compatible with the PSDK event culling** (Events Optimizer) and with any plugin that also uses `prepend` on the same classes.
- **Event IDs** start at `EVENT_ID_BASE` (100000) so they never collide with map events.
