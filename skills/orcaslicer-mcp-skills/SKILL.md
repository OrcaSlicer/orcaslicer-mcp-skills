---
name: orcaslicer-mcp-skills
description: "Use OrcaSlicer through its MCP server (tools named mcp__orcaslicer__* or similar): load, place and arrange models, plates, parts and modifiers, height ranges, print/filament/printer settings and presets (save, delete, import), supports, multi-colour, layer pauses and filament changes, cut/text/SVG/boolean gizmos and painting, calibration tests, slicing with time, filament and preview numbers, G-code/3MF/STL export and upload, and the Design (CAD) tab. Also for developing OrcaSlicer: launching it with MCP, reproducing issues, replayable scripts. Use whenever OrcaSlicer MCP tools are available or the user wants something done in OrcaSlicer."
---

# OrcaSlicer through MCP

## Start

1. MCP tools may be deferred. Where the client loads them on demand (ToolSearch in Claude Code),
   load what the task needs in one call, for example
   `select:mcp__orcaslicer__get_state,mcp__orcaslicer__list_objects,mcp__orcaslicer__set_settings,mcp__orcaslicer__get_settings,mcp__orcaslicer__slice,mcp__orcaslicer__export`;
   add `transform`, `add_part`, `arrange`, `select`, `run_action` for scene work, `select_preset`,
   `save_preset`, `list_presets` for presets, and the `ui_*`, `wait`, `dialog_auto_answer` tools only
   for gizmos and dialogs: every loaded schema costs on every call.
2. Call `get_state`: ready, tab, presets, objects, plates, open `dialogs`, the open `gizmo`, `jobs`.
3. **No OrcaSlicer tools at all** means the app is not connected: the user turns on Preferences >
   General > AI features > MCP server and adds it ("Add to an agent"); in an OrcaSlicer checkout, read
   `references/development.md`.

## Work efficiently

- **Typed tools first,** then `run_action`, and the GUI tools (`ui_*`) last.
  - A Speed Dial id is the full string with a string param:
    `run_action {"id":"orca_command:add_primitive_cube:orca","param":"40 40 10"}`. Every key in
    `references/actions.md` is `orca_command:<key>:orca`. Object commands act on what `select` picked.
- **Exact shapes** (these are what runs got wrong):
  - One camera per call: `set_view {"camera":"top"}`, then `set_view {"camera":"fit_selection"}`.
  - `export {"format":"gcode"|"sliced_3mf"|"3mf"|"stl","path":"/abs/…","overwrite":true}`.
  - `select {"objects":[0]}`; a part `select {"volumes":[[0,1]]}`; nothing `select {"none":true}`.
  - `transform {"object":0,"scale":1.5}` (relative), `"size":[80,null,null]`, `"plate":1`, a part with `"volume":1`.
  - `add_part {"object":0,"type":"negative","shape":"cylinder","size":[5,20],"position":[128,128,10]}`.
  - `dialog_auto_answer {"rules":[{"match":"…","button":"…"}]}`; it replaces earlier rules.
  - `ui_input` values are plain JSON (`"M5"`, `15`, `true`), never quoted twice.
- **Verify through state, not pictures:** `list_objects`, `list_plates`, `get_settings`, the `slice`
  reply, `get_preview`, exported files. Take a screenshot only when the user wants an image.
- **Settings:**
  - Find keys with `get_settings {"query":"<words>"}` (key or label): key, label, enum `values` with
    their `labels` (write the value), current value.
  - What a setting does, or which one controls an effect: `explain_settings {"keys":[…]}` or
    `{"query":"stringing"}` (a few distinctive words, matched in tooltips too): the full tooltip,
    `where` (page > group) and `wiki`. Check it before changing a setting you don't know.
  - Tuning (stronger, faster, less stringing): `explain_settings {"scope":"process"}` maps every
    setting by page > group; read the groups involved in full with `{"where":"Strength > Walls"}`
    (a group, not a whole page), then their values with `get_settings`.
  - `set_settings {"scope":"print"|"filament"|"printer","values":{...}}`, many keys per call. Other
    scopes: `"object","object":N` and `"volume","object":N,"volume":M` (overrides; `extruder` is the
    filament), `"project"` (plate type, slot colours, flushing, prime tower), `"plate","plate":N`.
  - Percents are strings (`"25%"`). Check the reply's `applied` and `changed`.
  - A per-variant value (meta `variants`, e.g. X1C speeds): one value writes the entry in use,
    `{"index":i,"value":v}` another.
- **Jobs:** a reply `{"status":"running"|"blocked_by_dialog","job":"jN"}` is still running: never
  repeat the call; `wait {"for":"job","job":"jN"}` (≤ 50 s, repeat it). A call that opens a dialog
  goes in one `run_script`: `"await":false,"save_as":"x"`, the answers, then `wait` job `${jobs.x}`.
- **Indices shift** after arrange, split, cut, assemble, change of part type, order by and a CAD
  commit: re-read `list_objects`. **Paths are absolute**; replacing a file needs `"overwrite":true`.

## Traps that decide tasks

- **Unsaved preset edits outlive the project.** `new_project {"discard_changes":true}` keeps them
  (the reply lists `modified`): add `"changes":"discard"` for a clean start. `select_preset` with
  edits opens a prompt unless `"changes":"transfer"|"discard"`.
- **A sliced file is not a project.** A "plate sliced file", "print from SD card" or a `.gcode.3mf`
  name means `export {"format":"sliced_3mf","plate":"all"|N,"path":"/abs/x.gcode.3mf"}` after
  slicing. `save_project` writes no G-code, whatever the file is called.
- **Slice numbers:** `filament_g` includes flush; `filaments[i]` (slot i+1) splits `model_g`,
  `support_g`, `flush_g`, `tower_g`. Feature times, per-layer times and the highest flow are in
  `get_preview`. Any edit invalidates the slice: slice again before reading or exporting.
- **Values whose meaning the label hides:**
  - `elefant_foot_compensation` is per side: a 20.4 mm first layer on a 20.0 mm part means 0.2.
  - With `brim_type` `brim_ears`, `brim_width` is the ear radius.
  - "N° from vertical" is `support_threshold_angle` = 90 − N.
  - "Manual supports" means `support_type` `normal(manual)` or `tree(manual)`.
  - `flush_into_objects` prints the objects in whatever filament is loaded; use `flush_into_infill`.
  - The print purges `flush_volumes_matrix` × `flush_multiplier` (0.3 on Bambu printers): a flush
    of V mm³ is the entry V / `flush_multiplier`.
- **Placement:** on the X1C rotate −45°, not +45° (front-left exclusion area). To fit a smaller
  printer use `obj_fit_volume`, not a hand-made scale. Printing by object needs
  `extruder_clearance_radius` (68 mm on the X1C) between objects: `references/scene.md`.
- **"Slicing is in progress"** from `export` or `get_preview` right after an edit, a plate switch or
  `slice {"plates":"all"}`: `wait {"for":"idle"}`, then repeat.
- **Prompts to pre-answer:** "too small" (inch model) → yes; STEP "import parameters" → ok, "floating
  parts" → no; spiral mode → yes; opening a `.3mf` over modified presets: "unsaved changes" →
  Discard; after Design-tab edits, "save before continuing" → no (the project reads not dirty).
- **Object overrides:** `get_settings {"scope":"object","object":N}` lists the overrides with
  `effective` and `source`; `"values":{"brim_type":null}` removes one. Only keys the object list
  offers there are taken.
- **Clicks on the canvas** under a panel or notification are refused, naming what covers it:
  enlarge the window (`set_view {"window_size":[1600,1000]}`) or turn the camera.

## When no tool reaches it

If an area file lists a task under "Not possible", don't probe the GUI for it; tell the user.
Installing system printers or filaments has no route. A few read-only details live only in the
saved 3MF (`references/project-file.md`).

## Where the recipes are

The files are in `references/` beside this SKILL.md; in Claude Code the "Base directory" line above
gives its absolute path. Open them directly; don't search the disk for them. Read the area's file
before the first call that moves objects, adds parts, changes plates or presets, uses a gizmo or the
Design tab, or exports anything other than G-code. Tags: **[ref]** verified on the current build,
**[hand]** verified by hand only; `→` means the next call.

| Task | File |
|---|---|
| Open, import, scale, size, rotate, mirror, copies, arrange, fill, plates, order, assemble | `references/scene.md` |
| Slice numbers, preview data, export, save, plate settings, upload to a printer | `references/output.md` |
| Setting keys; presets: select, compare, save, rename, delete, import, export; nozzle, plate type, host, preferences | `references/settings.md` |
| Supports, blockers and enforcers, brim and ears, raft, elephant-foot compensation | `references/supports.md` |
| Parts, negative parts, modifiers, height ranges, per-object and per-part settings, change type, rename | `references/modifiers.md` |
| Filament slots, colours, flushing, part filaments, painting, pauses and filament changes, HueForge | `references/multicolor.md` |
| Cut, connectors, text, SVG, boolean, measure, simplify, seam, fuzzy skin, texture, variable layer height; any gizmo | `references/gizmos.md` |
| Calibration tests, reading and saving the result | `references/calibration.md` |
| Design (CAD) tab: sketches, extrude, holes, threads, fillets, patterns, variables, edits, commit; any request to model or design a part, which is built there (not as a mesh written by a script), so the user gets a design they can edit | `references/cad.md` |
| Speed Dial ids | `references/actions.md` |
| Reading a saved project file | `references/project-file.md` |
| Launching OrcaSlicer with MCP, reproducing issues, scripts, tests, platform notes | `references/development.md` |

## GUI tools

- `ui_snapshot {"query":"…"}` finds widgets, list rows, menus, GL toolbar items, plate icons and
  ImGui items. Prefer locators to refs: `{"label":…}`, `{"imgui":…}`, `{"imgui_name":…}`,
  `{"gizmo":…}`, `{"gltoolbar":…}`, `{"row":"Cube>Generic-Cylinder"}`, `{"plate_icon":"lock","plate":0}`,
  `{"class":…,"index":N}`, with `"within":"<dialog title>"` for a control in a dialog.
- Menus are never shown: a click that opens one replies `popup_menu`; the same call with
  `"popup_item":"Add Modifier>Cylinder"` runs an item (`ran`). Clicks on the model take points, not
  pixels: `ui_mouse {"at":{"object":0,"point":[x,y,z]}}` (`references/gizmos.md`).
- When a call fails, `get_log {"level":"warning"}` says why.
