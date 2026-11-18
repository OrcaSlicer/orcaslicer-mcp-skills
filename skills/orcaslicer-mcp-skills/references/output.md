# Slicing, preview numbers, export, plates, upload

## Recipes

- Slice: `plate_slice {}` (current plate), `{"plates":1}` (0-based), `{"plates":"all"}`; long models
  `"timeout_s":1200`. Each plate has `outcome` (`ok`, `error`, `cancelled`, `empty`, `skipped`); an
  `ok` one has `time_s`, `time`, `filament_g`/`filament_m` (flush included), `filament_cost`,
  `layers`, `max_z`, `filament_changes`, and `filaments[i]` for slot i+1 with `g`, `model_g`,
  `support_g`, `flush_g`, `tower_g`. Purge waste = Σ `flush_g` + Σ `tower_g`. [ref]
- Preview numbers, no screenshot: `plate_slice_stats {"include":["features","layers","objects","view"]}`
  ( `"plate":N` for another sliced plate). [ref]
  - `features`: seconds, share and grams per feature type (`"Outer wall"`, `"Support"`…), `Travel`.
  - `layers`: per layer `layer` (1-based, the slider's), `z`, `height`, `time_s`, `g`; paged with
    `limit`/`offset`. `"layer_features"` adds seconds per feature type per layer.
  - `objects`: grams per object instance.
  - `view.ranges`: `[min, max]` per view type: highest flow =
    `volumetric_flow_rate[1]`, shortest layer time = `layer_time_linear[0]`.
  - `summary` adds `prepare_time_s`, `travel_time_s`, `travel_mm`.
- Export: `project_export {"format":"gcode","path":"/abs/x.gcode","overwrite":true}` (current plate,
  `"plates":N` for another); `"sliced_3mf"` with `"plates":N|"all"` and a `.gcode.3mf` path;
  `"3mf"` the model without slicing data; `"stl"` all objects in one file. [ref]
- One STL per object, named: rename the objects (`object_rename {"object":i,"name":"bunny_keychain"}`),
  then `project_export {"format":"stl","per_object":true,"path":"/abs/existing_dir"}` writes
  `<name>.stl` each. [ref]
- Save + reopen: `project_save {"path":"/abs/p.3mf","overwrite":true}` →
  `project_open {"path":"/abs/p.3mf","discard_changes":true}`. With modified presets the load asks
  "Load project: unsaved changes": auto-answer `{"match":"unsaved changes","button":"Discard"}`. [ref]
- A plate's own settings (Plate Settings dialog):
  `plate_set_settings {"plate":1,"values":{"curr_bed_type":"Cool Plate","print_sequence":"by object"}}`
  (also `spiral_mode`, `first_layer_print_sequence` `[2,1]`, `other_layers_print_sequence`
  `[{"from":2,"to":9,"order":[2,1]}]`; `null` = Same as Global); `plate_get_settings {"plate":1}` reads
  them with `effective`. Name and lock: `plate_rename {"plate":1,"name":"PETG"}`,
  `plate_lock {"plate":1,"locked":true}`. `plate_list {}` reads every plate back: `overrides`, `objects`
  (how many), `sliced`, `layer_gcode`. [ref]
- Upload to OctoPrint/Moonraker/PrusaLink: `settings_set {"type":"printer","values":
  {"host_type":"moonraker","print_host":"http://127.0.0.1:7125"}}` (`"printhost_apikey"` for
  OctoPrint; `"bbl_use_printhost":true` on a Bambu printer) → `plate_slice {}` → in one `replay_run`:
  `printer_send {}` (`"await":false`, `"save_as":"send"`) →
  `widget_set_value {"target":{"class":"wxTextCtrl","within":"Send G-code to printer host"},"value":"part.gcode","submit":false}`
  → `dialog_answer {"button":"Upload"}` (or `"Upload and Print"`) → wait job. Then poll
  `app_state.uploads` until the row reads `completed` or `error`. [ref]
- Preview view for screenshots:
  `view_set {"tab":"preview","view_type":"volumetric_flow_rate","layer_z":10.2}`;
  `"legend":{"Sparse infill":false,"travel":true}` hides or shows rows (after `view_type`). [ref]
- Record a video: `recording_start {"captions":true,"pointer":true,"encode":true,"save_path":"/abs/demo.mkv","overwrite":true}`
  (`"target":"canvas"` for the 3D view only; it stops by itself after `max_seconds`, an hour by
  default) → the calls to show → `recording_stop {"silent":true}`: frames, seconds, and with `encode`
  an `encode` job → `app_wait {"for":"job","job":"jN"}` returns the MP4's path (H.265 with the system
  ffmpeg; `"codec":"h264"` for H.264). Every encode, `recording_encode` too, deletes the `.mkv`
  once the MP4 is written; `"keep_raw_video_file":true` keeps it, to encode it again. Captions show each call; `pointer`
  highlights the pointer and its clicks. [ref]
  - Without arguments both are the person's commands: `recording_start {}` opens the Start Recording
    dialog, and `recording_stop {}` stops at once, shows the person a notification and replies with the
    file's path only.
  - Answer each dialog as soon as it opens: `app_wait {"for":"dialog"}`, pause 2–3 s so a viewer can read
    it (`app_wait {"for":"no_dialog","timeout_s":3}` returns after 3 s while it stays open), then
    `dialog_answer`. [ref] Never `app_wait {"for":"idle"}` while a dialog may be open: idle needs no
    dialog open, so the wait runs to its timeout and the video shows a frozen window.
  - On Linux record under X11, launched with `--headless` (`--xvfb` on a machine without a desktop;
    `references/development.md`): a Wayland recording draws each dialog where the compositor usually
    puts it, not where it is, and leaves out modeless windows and native dialogs.

## Traps

- After an edit, a plate switch or `plate_slice {"plates":"all"}`, `project_export` or
  `plate_slice_stats` can refuse "Slicing is in progress": `app_wait {"for":"idle"}`, then repeat.
- Any edit invalidates the slice: slice again before `plate_slice_stats` or export.
- A project with no objects still exports as `"3mf"`: a design not yet committed from the Design tab
  is one. `"gcode"`, `"sliced_3mf"` and `"stl"`, and their Speed Dial actions, are refused with "The
  project has no objects." (`no_objects`). [ref]
- `project_save` refuses a path ending in `.gcode.3mf` (`sliced_file_name`): a project is saved
  without G-code, and that file is
  `project_export {"format":"sliced_3mf","path":"/abs/x.gcode.3mf"}`. [ref]
- `view_set` with `layer`, `layer_range`, `layer_z` or `layer_percent` is refused with
  "The preview has no layers. Slice the plate first." (`no_layers`) until a slice has given the
  Preview layers. From then on the Preview keeps the last slice's layers, also after the objects are
  deleted or a new project is started, and the reply's `layers` (`[first, last, count]`) is of that
  slice: slice the plate before pointing the view at a layer. [ref]
- A by-object plate needs objects ≥ `extruder_clearance_radius` apart (`references/scene.md`).
- Opening a `.gcode` file to view it (`project_open`) asks to save the project even with
  `discard_changes`: auto-answer `{"match":"unsaved changes. Would you like to save","button":"no"}`
  first. With modified presets it also asks "Load project: unsaved changes": answer it Discard. [ref]

## Not possible

G-code text through MCP (export it and read the file); a print host connection test; a typed
upload without the Send dialog.
