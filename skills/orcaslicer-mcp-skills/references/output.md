# Slicing, preview numbers, export, plates, upload

## Recipes

- Slice: `slice {}` (current plate), `{"plates":1}` (0-based), `{"plates":"all"}`; long models
  `"timeout_s":1200`. Each plate has `outcome` (`ok`, `error`, `cancelled`, `empty`, `skipped`); an
  `ok` one has `time_s`, `time`, `filament_g`/`filament_m` (flush included), `filament_cost`,
  `layers`, `max_z`, `filament_changes`, and `filaments[i]` for slot i+1 with `g`, `model_g`,
  `support_g`, `flush_g`, `tower_g`. Purge waste = Σ `flush_g` + Σ `tower_g`. [ref]
- Preview numbers, no screenshot: `get_preview {"include":["features","layers","objects","view"]}`
  ( `"plate":N` for another sliced plate). [ref]
  - `features`: seconds, share and grams per feature type (`"Outer wall"`, `"Support"`…), `Travel`.
  - `layers`: per layer `layer` (1-based, the slider's), `z`, `height`, `time_s`, `g`; paged with
    `limit`/`offset`. `"layer_features"` adds seconds per feature type per layer.
  - `objects`: grams per object instance.
  - `view.ranges`: `[min, max]` per view type: highest flow =
    `volumetric_flow_rate[1]`, shortest layer time = `layer_time_linear[0]`.
  - `summary` adds `prepare_time_s`, `travel_time_s`, `travel_mm`.
- Export: `export {"format":"gcode","path":"/abs/x.gcode","overwrite":true}` (current plate,
  `"plate":N` for another); `"sliced_3mf"` with `"plate":N|"all"` and a `.gcode.3mf` path;
  `"3mf"` the model without slicing data; `"stl"` all objects in one file. [ref]
- One STL per object, named: rename the objects (`select {"objects":[i]}` →
  `run_action {"id":"orca_command:obj_rename:orca","param":"bunny_keychain"}`), then
  `run_action {"id":"orca_command:export_stl_multi:orca","param":"/abs/existing_dir"}` writes
  `<name>.stl` each. [ref]
- Save + reopen: `save_project {"path":"/abs/p.3mf","overwrite":true}` →
  `open_files {"paths":["/abs/p.3mf"],"discard_changes":true}`. With modified presets the load asks
  "Load project: unsaved changes": auto-answer `{"match":"unsaved changes","button":"Discard"}`. [ref]
- A plate's own settings (Plate Settings dialog, lock, name):
  `set_settings {"scope":"plate","plate":1,"values":{"curr_bed_type":"Cool Plate","print_sequence":"by object"}}`
  (also `spiral_mode`, `first_layer_print_sequence` `[2,1]`, `other_layers_print_sequence`
  `[{"from":2,"to":9,"order":[2,1]}]`, `name`, `locked`; `null` = Same as Global).
  `list_plates {}` reads every plate back: `overrides`, `objects`, `sliced`, `layer_gcode`. [ref]
- Upload to OctoPrint/Moonraker/PrusaLink: `set_settings {"scope":"printer","values":
  {"host_type":"moonraker","print_host":"http://127.0.0.1:7125"}}` (`"printhost_apikey"` for
  OctoPrint; `"bbl_use_printhost":true` on a Bambu printer) → `slice {}` → in one `run_script`:
  `run_action {"id":"orca_command:send_gcode:orca"}` (`"await":false`, `"save_as":"send"`) →
  `ui_input {"target":{"class":"wxTextCtrl","within":"Send G-code to printer host"},"value":"part.gcode","submit":false}`
  → `dialog_respond {"button":"Upload"}` (or `"Upload and Print"`) → wait job. Then poll
  `get_state.uploads` until the row reads `completed` or `error`. [ref]
- Preview view for screenshots:
  `set_view {"tab":"preview","view_type":"volumetric_flow_rate","layer_z":10.2}`;
  `"legend":{"Sparse infill":false,"travel":true}` hides or shows rows (after `view_type`). [ref]

## Traps

- After an edit, a plate switch or `slice {"plates":"all"}`, `export` or `get_preview` can refuse
  "Slicing is in progress": `wait {"for":"idle"}`, then repeat.
- Any edit invalidates the slice: slice again before `get_preview` or export.
- A by-object plate needs objects ≥ `extruder_clearance_radius` apart (`references/scene.md`).
- Opening a `.gcode` file to view it asks to save the project even with `discard_changes`:
  auto-answer `{"match":"unsaved changes. Would you like to save","button":"no"}` first.

## Not possible

G-code text through MCP (export it and read the file); a print host connection test; a typed
upload without the Send dialog.
