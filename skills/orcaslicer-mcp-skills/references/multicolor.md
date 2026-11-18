# Multi-colour and multi-material

## Recipes

- Filament slots: `filament_add {}` (the reply's `filament_slot` is the new slot; it copies the last
  slot's preset) → `preset_select {"type":"filament","name":"…","filament_slot":2}`.
  Delete: `filament_delete {"filament_slot":2}` (1-based; objects on it move to the first filament,
  or to `"merge_into":N`). A mixed filament: `filament_add_mixed {"components":[1,2],"ratios":[50,50]}`.
  [ref] From the connected printer's AMS: `filament_sync_ams {"mode":"overwrite"}` (`"map"` maps the
  project's filaments to its slots).
  - A filament assigned to a slot does what a pick in the slot's sidebar selector does: the slot takes
    the preset's colour when the preset defines one, the flushing volumes are recalculated (while the
    preference `auto_calculate_flush` is `all`, its default) and every plate's slice is invalidated. [ref]
- Object or part filament: `settings_set {"target":"object","object":1,"values":{"extruder":2}}`;
  a part `{"target":"part","object":0,"part":1,"values":{"extruder":2}}`; `0`/`null` = Default.
  Read back: `object_list` `filament` per object and part. [ref]
- Slot colours, flushing, prime tower (the project's own settings, as the sidebar's controls write them):
  `project_set_settings {"values":{"filament_colour":{"index":1,"value":"#000000"}}}`;
  flush from slot i to slot j (1-based) = entry `(i-1)*N + (j-1)` of the N×N row-major matrix
  (`{"flush_volumes_matrix":{"index":1,"value":400}}` for 1 → 2 of two slots), stored before
  `flush_multiplier`: the print purges entry × `flush_multiplier`, so a purge of V mm³ is the entry
  V / `flush_multiplier` (read it first with `project_get_settings`; 400 mm³ at 0.3 is 1333.33), and
  the G-code's `; flush_volumes_matrix` line shows the products; tower
  `{"wipe_tower_x":{"index":0,"value":150},"wipe_tower_y":{"index":0,"value":180}}` (per plate).
  Set the matrix last: a colour change, and a filament assigned to a slot, recalculate it. [ref]
- First-layer filament order: `plate_set_settings {"plate":0,"values":
  {"first_layer_print_sequence":[2,1]}}` (every slot once). [ref]
- Raised text in filament 2: `object_select {"objects":[0]}` → `view_set {"camera":"top"}` →
  `widget_click {"target":{"gizmo":"Text"}}` (`created: true`) →
  `widget_set_value {"target":{"imgui_name":"Text"},"value":"ORCA"}` → `app_wait {"for":"idle"}` →
  `widget_set_value {"target":{"imgui":"Height"},"value":8}` → wait idle →
  `widget_set_value {"target":{"imgui":"Depth"},"value":1.5}` → wait idle → `widget_click {"target":{"imgui":"Done"}}`
  → `settings_set {"target":"part","object":0,"part":1,"values":{"extruder":2}}`. [ref]
- Pause, custom G-code, filament change at a layer (slice first; slice again before export):
  `plate_set_custom_gcode {"add":[{"layer":26,"type":"pause"},{"z":10.2,"type":"custom","gcode":"M104 S215"},
  {"layer":60,"type":"change_filament","filament":2}]}`. `layer` is the slider's 1-based number, `z`
  the layer whose top is at or above it; the reply gives both, and `plate_list` `layer_gcode` gives
  `z`, with `layer` once the plate is sliced again.
  `"remove":[{"layer":26}]`, `"clear":true`. [ref]
- Manual swap on a single-extruder printer (M600): one slot and
  `plate_set_custom_gcode {"add":[{"z":15,"type":"custom","gcode":"M600"}]}`. A `change_filament` with two
  slots, `manual_filament_change` on and `enable_prime_tower` off emits the printer's
  `change_filament_gcode`, which can be empty (Voron). [ref]
- HueForge: `settings_set {"type":"process","values":{"layer_height":0.08,"initial_layer_print_height":0.16,
  "sparse_infill_density":"100%"}}` → add slots → slice → one `change_filament` per swap → slice. [ref]
- Less purge: `settings_set {"type":"process","values":{"flush_into_infill":true}}` (two cubes: 23 → 19.6 g
  flushed), a lower project `flush_multiplier`, or a narrower `prime_tower_width`; compare Σ `flush_g`
  + `tower_g` before and after. Not `flush_into_objects`: it makes the objects themselves the purge, so
  they print in whatever filament is loaded (both cubes came out in slot 1). [ref]
- Band + top painted (Voron cube at the plate centre; world points): [ref]
  1. `widget_click {"target":{"gizmo":"Color Painting"}}` → `widget_click {"target":{"imgui_name":"extruder_color_2"}}` →
     `widget_click {"target":{"imgui_name":"Height Range"}}` → `widget_set_value {"target":{"imgui":"Height range:"},"value":5}`.
  2. `view_set {"camera":"front"}` → `view_set {"camera":"fit_objects"}`.
  3. `mouse_gesture {"action":"move","at":{"object":0,"point":[116,113,10],"frame":"world"}}`, then the same
     call without `action`, twice. This paints 10–15 mm, upward.
  4. `widget_click {"target":{"imgui_name":"extruder_color_3"}}` → `widget_click {"target":{"imgui_name":"Fill"}}`.
  5. `view_set {"camera":"top"}` → `view_set {"camera":"fit_objects"}`.
  6. Move, then click, at `[118,135,30]`, a flat spot of the top face; `spot:"top"` hits the centre recess.
  7. `widget_click {"target":{"imgui":"Done"}}`.
- Remap colour 4 → 3 on a painted object: `object_select` → Color Painting →
  `widget_click {"target":{"imgui":"Remap filaments"}}` (it stays open; a second click folds it) →
  `widget_click {"target":{"imgui_name":"remap_src_4"}}` → `widget_click {"target":{"imgui_name":"remap_dst_3"}}` →
  `widget_click {"target":{"imgui":"Remap"}}` → Done; then delete slot 4. [ref]

## Traps

- ImGui text fields (the Text gizmo) drop spaces; `plate_set_custom_gcode` keeps them.
- Text lands where the screen centre meets the model: turn the camera first.
- The first painter click after typing a height once wasn't painted: click twice (harmless).
- `painted.mmu` in `object_list` is facet counts per filament, with no location.
- A filament change needs two or more slots and objects on one filament; a plate printed by object
  takes no layer codes.
- A mixed-colour filament's slot (`project_get_settings` `filament_is_mixed`) has no preset of its own:
  `preset_select` with its `filament_slot` is refused, "Filament slot N is a mixed-color virtual slot,
  which has no filament preset of its own." (`mixed_filament_slot`). [ref]
- `filament_add` and `filament_delete` are the filament list's + and − buttons: a printer with one
  filament per extruder, which shows none, refuses them.

## Not possible

Typed painting by region or height.
