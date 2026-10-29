# Multi-colour and multi-material

## Recipes

- Filament slots: `run_action {"id":"orca_command:filament_add:orca"}` (the message names the new
  slot; it copies the last slot's preset) → `select_preset {"type":"filament","name":"…","filament_slot":2}`.
  Delete: `run_action {"id":"orca_command:filament_delete:orca","param":"2"}` (1-based; objects on it
  move). [ref]
- Object or part filament: `set_settings {"scope":"object","object":1,"values":{"extruder":2}}`;
  a part `{"scope":"volume","object":0,"volume":1,"values":{"extruder":2}}`; `0`/`null` = Default.
  Read back: `list_objects` `filament` per object and part. [ref]
- Slot colours, flushing, prime tower (project scope, as the sidebar's controls write them):
  `set_settings {"scope":"project","values":{"filament_colour":{"index":1,"value":"#000000"}}}`;
  flush from slot i to slot j (1-based) = entry `(i-1)*N + (j-1)` of the N×N row-major matrix
  (`{"flush_volumes_matrix":{"index":1,"value":400}}` for 1 → 2 of two slots), stored before
  `flush_multiplier`: the print purges entry × `flush_multiplier`, so a purge of V mm³ is the entry
  V / `flush_multiplier` (read it first; 400 mm³ at 0.3 is 1333.33), and the G-code's
  `; flush_volumes_matrix` line shows the products; tower
  `{"wipe_tower_x":{"index":0,"value":150},"wipe_tower_y":{"index":0,"value":180}}` (per plate).
  Set the matrix last: a colour change recalculates it. [ref]
- First-layer filament order: `set_settings {"scope":"plate","plate":0,"values":
  {"first_layer_print_sequence":[2,1]}}` (every slot once). [ref]
- Raised text in filament 2: `select {"objects":[0]}` → `set_view {"camera":"top"}` →
  `ui_click {"target":{"gizmo":"Text"}}` (`created: true`) →
  `ui_input {"target":{"imgui_name":"Text"},"value":"ORCA"}` → `wait {"for":"idle"}` →
  `ui_input {"target":{"imgui":"Height"},"value":8}` → wait idle →
  `ui_input {"target":{"imgui":"Depth"},"value":1.5}` → wait idle → `ui_click {"target":{"imgui":"Done"}}`
  → `set_settings {"scope":"volume","object":0,"volume":1,"values":{"extruder":2}}`. [ref]
- Pause, custom G-code, filament change at a layer (slice first; slice again before export):
  `set_layer_gcode {"add":[{"layer":26,"type":"pause"},{"z":10.2,"type":"custom","gcode":"M104 S215"},
  {"layer":60,"type":"change_filament","filament":2}]}`. `layer` is the slider's 1-based number, `z`
  the layer whose top is at or above it; the reply and `list_plates` `layer_gcode` give both.
  `"remove":[{"layer":26}]`, `"clear":true`. [ref]
- Manual swap on a single-extruder printer (M600): one slot and
  `set_layer_gcode {"add":[{"z":15,"type":"custom","gcode":"M600"}]}`. A `change_filament` with two
  slots, `manual_filament_change` on and `enable_prime_tower` off emits the printer's
  `change_filament_gcode`, which can be empty (Voron). [ref]
- HueForge: `set_settings {"scope":"print","values":{"layer_height":0.08,"initial_layer_print_height":0.16,
  "sparse_infill_density":"100%"}}` → add slots → slice → one `change_filament` per swap → slice. [ref]
- Less purge: `set_settings {"scope":"print","values":{"flush_into_infill":true}}` (two cubes: 23 → 19.6 g
  flushed), a lower project `flush_multiplier`, or a narrower `prime_tower_width`; compare Σ `flush_g`
  + `tower_g` before and after. Not `flush_into_objects`: it makes the objects themselves the purge, so
  they print in whatever filament is loaded (both cubes came out in slot 1). [ref]
- Band + top painted (Voron cube at the plate centre; world points): [ref]
  1. `ui_click {"target":{"gizmo":"Color Painting"}}` → `ui_click {"target":{"imgui_name":"extruder_color_2"}}` →
     `ui_click {"target":{"imgui_name":"Height Range"}}` → `ui_input {"target":{"imgui":"Height range:"},"value":5}`.
  2. `set_view {"camera":"front"}` → `set_view {"camera":"fit_objects"}`.
  3. `ui_mouse {"action":"move","at":{"object":0,"point":[116,113,10],"frame":"world"}}`, then the same
     call without `action`, twice. This paints 10–15 mm, upward.
  4. `ui_click {"target":{"imgui_name":"extruder_color_3"}}` → `ui_click {"target":{"imgui_name":"Fill"}}`.
  5. `set_view {"camera":"top"}` → `set_view {"camera":"fit_objects"}`.
  6. Move, then click, at `[118,135,30]`, a flat spot of the top face; `spot:"top"` hits the centre recess.
  7. `ui_click {"target":{"imgui":"Done"}}`.
- Remap colour 4 → 3 on a painted object: `select` → Color Painting →
  `ui_click {"target":{"imgui":"Remap filaments"}}` (it stays open; a second click folds it) →
  `ui_click {"target":{"imgui_name":"remap_src_4"}}` → `ui_click {"target":{"imgui_name":"remap_dst_3"}}` →
  `ui_click {"target":{"imgui":"Remap"}}` → Done; then delete slot 4. [ref]

## Traps

- ImGui text fields (the Text gizmo) drop spaces; `set_layer_gcode` keeps them.
- Text lands where the screen centre meets the model: turn the camera first.
- The first painter click after typing a height once wasn't painted: click twice (harmless).
- `painted.mmu` in `list_objects` is facet counts per filament, with no location.
- A filament change needs two or more slots and objects on one filament; a plate printed by object
  takes no layer codes.

## Not possible

Typed painting by region or height.
