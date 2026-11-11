---
name: orcaslicer-mcp-skills
description: "Use OrcaSlicer through its MCP server (tools named mcp__orcaslicer__* or similar): load, place and arrange models, plates, parts and modifiers, height ranges, print/filament/printer settings and presets (save, delete, import), supports, multi-colour, layer pauses and filament changes, cut/text/SVG/boolean gizmos and painting, calibration tests, slicing with time, filament and preview numbers, G-code/3MF/STL export and upload, the Design (CAD) tab, dialogs, video recordings, Python scripts and macros, replays of tool calls, and plugins: running a plugin's commands and operations, and writing, installing, reloading and debugging Python plugins. Also for developing OrcaSlicer: launching it with MCP, reproducing issues, replays. Use whenever OrcaSlicer MCP tools are available or the user wants something done in OrcaSlicer."
---

# OrcaSlicer through MCP

## Start

1. MCP tools may be deferred. Where the client loads them on demand (ToolSearch in Claude Code),
   load what the task needs in one call, for example
   `select:mcp__orcaslicer__get_app_state,mcp__orcaslicer__list_objects,mcp__orcaslicer__set_settings,mcp__orcaslicer__get_settings,mcp__orcaslicer__slice,mcp__orcaslicer__export`;
   add `transform`, `add_part`, `arrange`, `select_objects`, `run_action` for scene work, `select_preset`,
   `save_preset`, `list_presets` for presets, `search_tools` when unsure what does something,
   `run_script`, `search_python_api`, `run_replay`, `call_tool` for scripts, replays and plugins,
   `list_cad_tools`, `describe_cad_tool`, `call_cad_tool` for the Design tab, and `list_widgets`,
   `click`, `set_widget_value`, `mouse_gesture`, `wait`, `answer_dialog`, `auto_answer_dialogs` only
   for gizmos and dialogs: every loaded schema costs on every call.
2. Call `get_app_state`: ready, tab, presets, objects, plates, the `selection`, open `dialogs`, the open
   `gizmo`, `jobs`, whether Python scripts may run (`python.scripts_allowed`) and whether the Design
   tab is on (`design_tab`).
3. **No OrcaSlicer tools at all** means the app is not connected: the user turns on Preferences >
   General > AI features > MCP server and adds it ("Add to an agent"); in an OrcaSlicer checkout, read
   `references/development.md`.

## Work efficiently

- **Typed tools first,** then `run_action`, and the GUI tools (`list_widgets`, `click`,
  `set_widget_value`, `press_keys`, `mouse_gesture`) last.
  - A Speed Dial action is its key with a string param:
    `run_action {"id":"add_primitive_cube","param":"40 40 10"}`. The keys are in
    `references/actions.md`; object commands act on what `select_objects` picked.
  - Not sure which tool, action, setting or plugin operation does something:
    `search_tools {"query":"<a few words>"}` searches them all at once, with the Python API and the
    Design tab's CAD tools, each match with how to use it;
    `call_tool {"tool":"plugin.<plugin>.<name>","args":{…}}` runs an operation a plugin offers, which
    the tool list doesn't show.
- **Exact shapes** (these are what runs got wrong):
  - One camera per call: `set_view {"camera":"top"}`, then `set_view {"camera":"fit_selection"}`.
    Between the named directions: `set_view {"camera":"front","orbit":[30,20]}` (yaw, pitch in degrees),
    never a drag, which follows the user's mouse settings. Close in on a point of the scene:
    `set_view {"camera":"top","look_at":<point>,"zoom":2}`, the point in mm taken from the scene (an
    instance's `offset` from `list_objects`; on the Design tab, `describe_scene`'s `modeling_origin`),
    `zoom` in screenshot pixels per mm.
  - `export {"format":"gcode"|"sliced_3mf"|"3mf"|"stl","path":"/abs/…","overwrite":true}`; another plate
    `"plates":N`.
  - `select_objects {"objects":[0]}`; a part `select_objects {"volumes":[[0,1]]}`; nothing `select_objects {"none":true}`.
  - `transform {"object":0,"scale":1.5}` (relative), `"size":[80,null,null]`, `"plate":1`, a part with `"volume":1`.
  - `add_part {"object":0,"type":"negative","shape":"cylinder","size":[5,20],"position":[128,128,10]}`.
  - `auto_answer_dialogs {"rules":[{"match":"…","button":"…"}]}`; it replaces earlier rules. The regex
    is searched in the title, message and text: match a phrase (`"unsaved changes"`) or the title
    (`"^Preferences$"`), since a bare word (`"unsaved"`) also closes any dialog whose text holds it.
  - `set_widget_value` values are plain JSON (`"M5"`, `15`, `true`), never quoted twice.
- **Verify through state, not pictures:** `list_objects`, `list_plates`, `get_settings`, the `slice`
  reply, `get_slice_stats`, exported files. Take a screenshot only when the user wants an image.
- **Settings:**
  - Find a key, what a setting does, or which one controls an effect: `get_settings {"query":"stringing"}`
    (a few distinctive words) or `{"keys":[…]}`, with no scope: it looks through the process, filament
    and printer settings. Each item has its `scope` (the one `set_settings` needs), its value, and in
    `meta` the label, `description` (the full tooltip), `where` (page > group) and enum `values` with
    their `labels` (write the value). Check it before changing a setting you don't know.
  - Tuning (stronger, faster, less stringing): `get_settings {"where":"?"}` lists the process's
    page > group names; read the groups involved in full with `{"where":"Strength > Walls"}` (a group,
    not a whole page; add `"scope":"filament"` or `"printer"` for theirs).
  - `set_settings {"scope":"process"|"filament"|"printer","values":{...}}`, many keys per call; a key of
    another scope is refused, naming it. Other scopes: `"object","object":N` and
    `"volume","object":N,"volume":M` (overrides; `extruder` is the filament), `"project"` (plate type,
    slot colours, flushing, prime tower), `"plate","plate":N`.
  - Percents are strings (`"25%"`). Check the reply's `applied` and `changed`.
  - A per-variant value (meta `variants`, e.g. X1C speeds): one value writes the entry in use,
    `{"index":i,"value":v}` another.
- **Jobs:** a reply `{"status":"running"|"blocked_by_dialog","job":"jN"}` is still running: never
  repeat the call. Answer a dialog with `answer_dialog` (`get_app_state.dialogs` lists it), then
  `wait {"for":"job","job":"jN"}` (≤ 50 s, repeat it). Or put the call, its answers and the wait in
  one `run_replay`: `"await":false,"save_as":"x"`, the answers, then `wait` job `${jobs.x}`.
- **Dialogs:** a dialog's `message` (in `get_app_state.dialogs` and `blocked_by_dialog` replies) reads it
  as shown: its text, then its controls row by row, `[value]` fields, `[value ▾]` choices, `[x]`/`[ ]`
  checkboxes, `(*)`/`( )` options and tabs, `- row | cells` list rows, `[******]` secrets, ` (disabled)`.
  Read it instead of taking a screenshot.
  - `answer_dialog {"button":"<label>"}` presses a button; an option's label (`(*)`/`( )`) selects it
    and leaves the dialog open. A disabled button is refused: the message shows what it still needs.
    Set fields and checkboxes first with `set_widget_value {"target":{"label":…,"within":"<title>"},"value":…}`.
  - A native message box (`native: true`) takes `answer_dialog` on Windows and Linux; a file or
    folder picker, and any native dialog on macOS, only an `auto_answer_dialogs` rule registered
    before it opens. A picker's rule can hand it the file:
    `{"match":"<its prompt>","button":"ok","path":"/abs/file"}` (on Linux, or without `path`, the rule
    cancels the picker).
- **Indices shift** after arrange, split, cut, assemble, change of part type, order by and a CAD
  commit: re-read `list_objects`. Its `id` beside the index guards a call:
  `transform {"object":1,"object_id":732,…}` (`select_objects {"objects":[1],"object_ids":[732]}`) is refused
  when index 1 now names another object. **Paths are absolute**; replacing a file needs
  `"overwrite":true`.
- **Lists come in pages:** `{items, offset, total}`, with `next_offset` while more remain.

## Traps that decide tasks

- **Unsaved preset edits outlive the project.** `new_project {"discard_changes":true}` keeps them
  (the reply lists `modified`): add `"preset_changes":"discard"` for a clean start. `select_preset` with
  edits opens a prompt unless `"preset_changes":"transfer"|"discard"`.
- **A sliced file is not a project.** A "plate sliced file", "print from SD card" or a `.gcode.3mf`
  name means `export {"format":"sliced_3mf","plates":"all"|N,"path":"/abs/x.gcode.3mf"}` after
  slicing. `save_project` writes no G-code, whatever the file is called.
- **Slice numbers:** `filament_g` includes flush; `filaments[i]` (slot i+1) splits `model_g`,
  `support_g`, `flush_g`, `tower_g`. Feature times, per-layer times and the highest flow are in
  `get_slice_stats`. Any edit invalidates the slice: slice again before reading or exporting.
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
- **"Slicing is in progress"** from `export` or `get_slice_stats` right after an edit, a plate switch or
  `slice {"plates":"all"}`: `wait {"for":"idle"}`, then repeat.
- **Prompts to pre-answer:** "too small" (inch model) → yes; STEP "import parameters" → ok, "floating
  parts" → no; spiral mode → yes; opening a `.3mf` over modified presets: "unsaved changes" →
  Discard; after Design-tab edits, "save before continuing" → no (the project reads not dirty).
- **Object overrides:** `get_settings {"scope":"object","object":N}` lists the overrides with
  `effective` and `source`; `"values":{"brim_type":null}` removes one. Only keys the object list
  offers there are taken.
- **Clicks on the canvas** under a panel or notification are refused, naming what covers it:
  enlarge the window (`set_window {"size":[1600,1000]}`) or turn the camera.

## When no tool reaches it

If an area file lists a task under "Not possible", don't probe the GUI for it; tell the user.
Computing over objects, meshes and settings, or the same calls over many objects, is a Python script
when the person allows them (`references/python.md`); a fixed sequence of calls to send at once or
run again is a replay (`references/replays.md`). What an installed plugin offers is reached through
`run_action` and `call_tool`, and a command, tab or G-code step the user wants to keep in the app is
a plugin you write for them (`references/plugins.md`). A system printer or filament that isn't
installed is `select_preset` by name for this session; installing one for good has no verified route.
A few read-only details live only in the saved 3MF (`references/project-file.md`).

## Where the recipes are

The files are in `references/` beside this SKILL.md; in Claude Code the "Base directory" line above
gives its absolute path. Open them directly; don't search the disk for them. Read the area's file
before the first call that moves objects, adds parts, changes plates or presets, uses a gizmo or the
Design tab, runs Python or a replay, uses or writes a plugin, records a video, or exports anything
other than G-code. Tags: **[ref]**
verified on the current build, **[hand]** verified by hand only; `→` means the next call.

| Task | File |
|---|---|
| Open, import, scale, size, rotate, mirror, copies, arrange, fill, plates, order, assemble | `references/scene.md` |
| Slice numbers, preview data, export, save, plate settings, upload to a printer, video recordings | `references/output.md` |
| Setting keys; presets: select (hidden ones too), compare, save, rename, delete, import, export; nozzle, plate type, host, preferences | `references/settings.md` |
| Supports, blockers and enforcers, brim and ears, raft, elephant-foot compensation | `references/supports.md` |
| Parts, negative parts, modifiers, height ranges, per-object and per-part settings, change type, rename | `references/modifiers.md` |
| Filament slots, colours, flushing, part filaments, painting, pauses and filament changes, HueForge | `references/multicolor.md` |
| Cut, connectors, text, SVG, boolean, measure, simplify, seam, fuzzy skin, texture, variable layer height; any gizmo | `references/gizmos.md` |
| Calibration tests, reading and saving the result | `references/calibration.md` |
| Design (CAD) tab: sketches, extrude, holes, threads, fillets, patterns, variables, edits, commit; any request to model or design a part, which is built there (not as a mesh written by a script), so the user gets a design they can edit | `references/cad.md` |
| Python scripts and macros: computing over objects, meshes and settings, loops of tool calls, windows that ask the person, the `orca` API | `references/python.md` |
| Replays of tool calls: sending many calls at once, answering the dialogs they open, `expect` checks, `export_replay` | `references/replays.md` |
| Plugins: running an installed plugin's commands, pages and operations, the Plugins dialog, permission prompts; writing, installing, reloading and debugging a Python plugin | `references/plugins.md` |
| Speed Dial ids | `references/actions.md` |
| Reading a saved project file | `references/project-file.md` |
| Launching OrcaSlicer with MCP, reproducing issues, replays, tests, platform notes | `references/development.md` |

## GUI tools

- `list_widgets {"query":"…"}` finds widgets, list rows, menus, GL toolbar items, plate icons, ImGui
  items and the elements of web pages (the setup wizard, sign-in, plugin windows). Prefer locators to
  refs: `{"label":…}`, `{"option":"layer_height"}` (a setting's field), `{"imgui":…}`,
  `{"imgui_name":…}`, `{"gizmo":…}`, `{"gltoolbar":…}`, `{"menu":"File>Export>Export G-code"}`,
  `{"row":"Cube>Generic-Cylinder"}`, `{"plate_icon":"lock","plate":0}`, `{"tab":"General"}`,
  `{"web":"Next"}`, `{"class":…,"index":N}`, with `"within":"<dialog title>"` for a control in a
  dialog. list_widgets's description gives the full grammar.
- `list_windows` lists the open windows, modal or not, with refs: a window's ref is the `target` of
  `screenshot` (that window alone), `list_widgets` (only its widgets) and `set_window`, and its title
  the `within` of locators.
- Menus are never shown: a click that opens one replies `popup_menu`; the same call with
  `"popup_item":"Add Modifier>Cylinder"` runs an item (`ran`). Clicks on the model take points, not
  pixels: `mouse_gesture {"at":{"object":0,"point":[x,y,z]}}` (`references/gizmos.md`).
- When a call fails, `get_log {"level":"warning"}` says why.
