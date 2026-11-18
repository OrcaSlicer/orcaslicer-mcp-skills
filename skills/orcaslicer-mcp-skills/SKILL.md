---
name: orcaslicer-mcp-skills
description: "Use OrcaSlicer through its MCP server (tools named mcp__orcaslicer__* or similar): load, place and arrange models, plates, parts and modifiers, height ranges, print/filament/printer settings and presets (save, delete, import), supports, multi-colour, layer pauses and filament changes, cut/text/SVG/boolean gizmos and painting, calibration tests, slicing with time, filament and preview numbers, G-code/3MF/STL export and upload, the Design (CAD) tab, dialogs, video recordings, Python scripts and macros, replays of tool calls, and plugins: running a plugin's commands and operations, and writing, installing, reloading and debugging Python plugins. Also for developing OrcaSlicer: launching it with MCP, reproducing issues, replays. Use whenever OrcaSlicer MCP tools are available or the user wants something done in OrcaSlicer."
---

# OrcaSlicer through MCP

## Start

1. MCP tools may be deferred. Where the client loads them on demand (ToolSearch in Claude Code),
   load what the task needs in one call, for example
   `select:mcp__orcaslicer__app_state,mcp__orcaslicer__object_list,mcp__orcaslicer__settings_set,mcp__orcaslicer__settings_get,mcp__orcaslicer__plate_slice,mcp__orcaslicer__project_export`;
   add `project_import`, `object_transform`, `part_add`, `object_arrange`, `object_select` and the other
   `object_` commands for scene work, `preset_select`, `preset_save`, `preset_list` for presets,
   `tool_search` and `tool_call` when unsure what does something and for the commands the tool list
   doesn't show (the Design tab's, plugin operations), `script_run`, `script_search_api`, `replay_run`
   for scripts and replays, and `widget_list`, `widget_click`, `widget_set_value`, `mouse_gesture`,
   `app_wait`, `dialog_answer`, `dialog_auto_answer` only for gizmos and dialogs: every loaded schema
   costs on every call.
2. Call `app_state`: ready, tab, presets, objects, plates, the `selection`, open `dialogs`, the open
   `gizmo`, `jobs`, whether Python scripts run unrestricted (`python.scripts_unrestricted`) and whether
   the Design tab is on (`design_tab`; when its commands don't run, `cad_note` says how the person
   turns it on).
3. **No OrcaSlicer tools at all** means the app is not connected: the user turns on Preferences >
   General > AI features > MCP server and adds it ("Add to an agent"); in an OrcaSlicer checkout, read
   `references/development.md`.

## Work efficiently

- **Commands first,** and the GUI tools (`widget_list`, `widget_click`, `widget_set_value`,
  `keyboard_press`, `mouse_gesture`) last.
  - A menu row is a call of its command with the row's arguments: Mirror > Along X Axis is
    `object_mirror {"axis":"x"}`, a cube of a size `object_add_primitive {"shape":"cube","size":[40,40,10]}`.
    `references/actions.md` lists every Speed Dial tile a person can pick with the command and
    arguments it runs; a command that takes objects or parts acts on those it names, else on what
    `object_select` picked. Some tiles are the person's own and have a command for agents: Action
    Replay `replay_run`, Run Script `script_run`, Take Screenshot (a PNG where recordings go,
    Downloads by default) `window_screenshot`, the calibration dialogs `calibration_generate_test`
    with the test's values. Leave the wiki, YouTube and plugin hub pages (`app_open_web_page`),
    About (`window_open {"window":"about"}`) and the configuration folder (`app_show_config_folder`)
    to the person. A Design tile runs through `action_run` only while the Design tab is shown in the
    tile's mode (in a sketch or outside one); `tool_call` runs its command from any tab
    (`references/cad.md`).
  - **A tile's arguments are the person's form.** A call whose arguments are exactly those of one of
    the command's tiles (`{}` for a tile that has none; Go to tab and Go to layer, which collect a
    value, aside) does what a person's pick of that tile does: one that asks opens its dialog
    (`blocked_by_dialog`), one that acts on the selection acts on it, and a Design command with a
    menu row does what the row does, which can be to open a card, to decline, or to act at once on
    the selection (a Delete row deletes it) (`references/cad.md`). Any other call runs from its
    arguments and opens no dialog for them:
    `project_export {"format":"gcode"}` opens the file dialog,
    `project_export {"format":"gcode","path":"/abs/x.gcode"}` writes the file.
    `"silent":true` runs one whose schema has `silent` from its arguments, alone where its other
    arguments all have defaults (`project_new {"silent":true}`); one with a required argument is
    given that argument instead (`project_save {"silent":true}` is refused, "path is required"). A
    command without `silent` refuses it as an unknown argument (some Design commands ignore it:
    `references/cad.md`). `recording_stop {"silent":true}` waits for the file and reports it,
    `recording_stop {}` is the person's Stop.
  - Three kinds of call fall outside that rule. A command with no tile has no person's form: called
    with no arguments it runs, or is refused for the argument it requires (`preset_select {}`). One
    whose dialog is what it does opens it with arguments too: `preset_export {"path":"/abs/dir"}`
    opens the export dialog, and the path only spares it the folder question (`printer_send`,
    `window_open` and `app_benchmark_3d_scene` open theirs too). And what a call sets off can still
    ask: a STEP file's import options during `project_import`, spiral vase's confirmation from
    `settings_set`, a message when `printer_install` selects a new printer (`blocked_by_dialog`).
  - Not sure which command, action, setting or plugin operation does something:
    `tool_search {"query":"<a few words>"}` searches them all at once, with the Python API, each
    match with how to use it. It finds every command, also those the tool list doesn't show (three
    menu rows that take no arguments, the Design tab's and the operations plugins offer,
    `plugin.<plugin>.<name>`), with such a one's input schema; `{"tool":"<name>"}` gives one's
    description, schema and examples, `{"group":"Design"}` the names in a group.
    `tool_call {"tool":"<name>","args":{…}}` runs any of them.
- **Exact shapes** (these are what runs got wrong):
  - One camera per call: `view_set {"camera":"top"}`, then `view_set {"camera":"fit_selection"}`.
    Between the named directions: `view_set {"camera":"front","orbit":[30,20]}` (yaw, pitch in degrees),
    never a drag, which follows the user's mouse settings. Close in on a point of the scene:
    `view_set {"camera":"top","look_at":<point>,"zoom":2}`, the point in mm taken from the scene (an
    instance's `offset` from `object_list`; on the Design tab, `describe_design`'s `modeling_origin`),
    `zoom` in screenshot pixels per mm.
  - `project_export {"format":"gcode"|"sliced_3mf"|"3mf"|"stl","path":"/abs/…","overwrite":true}`; another plate
    `"plates":N`.
  - `object_select {"objects":[0]}`; a part `object_select {"parts":[[0,1]]}`; nothing `object_select {"none":true}`.
  - `object_transform {"object":0,"scale":1.5}` (relative), `"size":[80,null,null]`, `"plate":1`, a part with `"part":1`.
  - `part_add {"object":0,"type":"negative","shape":"cylinder","size":[5,20],"position":[128,128,10]}`.
  - `dialog_auto_answer {"rules":[{"match":"…","button":"…"}]}`; it replaces earlier rules. The regex
    is searched in the title, message and text: match a phrase (`"unsaved changes"`) or the title
    (`"^Preferences$"`), since a bare word (`"unsaved"`) also closes any dialog whose text holds it.
  - `widget_set_value` values are plain JSON (`"M5"`, `15`, `true`), never quoted twice.
- **The scene is a tree:** `object_list` gives `plates[i].objects[j]` and `outside` (objects on no
  plate), each object with its `instances` and `parts`. An object's `index` and `id` are the project's,
  whatever the plate. An "assembly" is one object with several parts (`references/scene.md`).
- **Verify through state, not pictures:** `object_list`, `plate_list`, `settings_get`, the `plate_slice`
  reply, `plate_slice_stats`, exported files. Take a screenshot only when the user wants an image.
- **Settings:**
  - Find a key, what a setting does, or which one controls an effect: `settings_get {"query":"stringing"}`
    (a few distinctive words) or `{"keys":[…]}`, with no type: it looks through the process, filament
    and printer settings. Each item has its `type` (the one `settings_set` needs), its value, and in
    `meta` the label, `description` (the full tooltip), `where` (page > group) and enum `values` with
    their `labels` (write the value). Check it before changing a setting you don't know.
  - Tuning (stronger, faster, less stringing): `settings_list {"type":"process","where":"?"}` names no
    group, so its error lists the process's page > group names (`groups`); read the groups involved in
    full with `settings_list {"type":"process","where":"Strength > Walls"}` (a group, not a whole page;
    `"type":"filament"` or `"printer"` for theirs).
  - `settings_set {"type":"process"|"filament"|"printer","values":{...}}`, many keys per call; a key of
    another type is refused, naming it. Overrides take a `target`: `"object","object":N`,
    `"part","object":N,"part":M` (`extruder` is the filament) and
    `"height_range","object":N,"min_z":…,"max_z":…`. The project's own settings (plate type, slot
    colours, flushing, prime tower) are `project_set_settings {"values":{…}}`, a plate's
    `plate_set_settings {"plate":N,"values":{…}}`, the app's preferences `preferences_set {"values":{…}}`.
  - Percents are strings (`"25%"`). Check the reply's `applied` and `changed`.
  - A per-variant value (meta `variants`, e.g. X1C speeds): one value writes the entry in use,
    `{"index":i,"value":v}` another.
- **Jobs:** a reply `{"status":"running"|"blocked_by_dialog","job":"jN"}` is still running: never
  repeat the call. Answer a dialog with `dialog_answer` (`app_state.dialogs` lists it), then
  `app_wait {"for":"job","job":"jN"}` (≤ 50 s, repeat it). Or put the call, its answers and the wait in
  one `replay_run`: `"await":false,"save_as":"x"`, the answers, then `app_wait` job `${jobs.x}`.
- **Dialogs:** a dialog's `message` (in `app_state.dialogs` and `blocked_by_dialog` replies) reads it
  as shown: its text, then its controls row by row, `[value]` fields, `[value ▾]` choices, `[x]`/`[ ]`
  checkboxes, `(*)`/`( )` options and tabs, `- row | cells` list rows, `[******]` secrets, ` (disabled)`.
  Read it instead of taking a screenshot.
  - `dialog_answer {"button":"<label>"}` presses a button; an option's label (`(*)`/`( )`) selects it
    and leaves the dialog open. A disabled button is refused: the message shows what it still needs.
    Set fields and checkboxes first with `widget_set_value {"target":{"label":…,"within":"<title>"},"value":…}`.
  - A native message box (`native: true`) takes `dialog_answer` on Windows and Linux; a file or
    folder picker, and any native dialog on macOS, only a `dialog_auto_answer` rule registered
    before it opens. A picker's rule can hand it the file:
    `{"match":"<its prompt>","button":"ok","path":"/abs/file"}` (on Linux, or without `path`, the rule
    cancels the picker).
- **Refusals:** a failed call's error is its reason; where the app refused what was asked, a second
  line of JSON details follows with a stable `code`: `The project has no objects.` then
  `{"code":"no_objects"}`. A tile runs its command, so `action_run` relays what the command's call
  would say. Tell cases apart by the `code`, not the wording.
- **Indices shift** after arrange, split, cut, assemble, change of part type, order by and a CAD
  commit: re-read `object_list`. Its `id` beside the index guards a call:
  `object_transform {"object":1,"object_id":732,…}` (`object_select {"objects":[1],"object_ids":[732]}`) is refused
  when index 1 now names another object. **Paths are absolute**; replacing a file needs
  `"overwrite":true`.
- **Lists that take `limit` come in pages:** `{items, offset, total}` (`object_list` has `plates` and
  `outside` in place of `items`), with `next_offset` while more remain.

## Traps that decide tasks

- **Unsaved preset edits outlive the project.** `project_new {"discard_changes":true}` keeps them
  (the reply lists `modified`): add `"preset_changes":"discard"` for a clean start. `preset_select` with
  edits opens a prompt unless `"preset_changes":"transfer"|"discard"`.
- **A sliced file is not a project.** A "plate sliced file", "print from SD card" or a `.gcode.3mf`
  name means `project_export {"format":"sliced_3mf","plates":"all"|N,"path":"/abs/x.gcode.3mf"}` after
  slicing. `project_save` writes no G-code and refuses a name ending in `.gcode.3mf`.
- **Slice numbers:** `filament_g` includes flush; `filaments[i]` (slot i+1) splits `model_g`,
  `support_g`, `flush_g`, `tower_g`. Feature times, per-layer times and the highest flow are in
  `plate_slice_stats`. Any edit invalidates the slice: slice again before reading or exporting.
- **Values whose meaning the label hides:**
  - `elefant_foot_compensation` is per side: a 20.4 mm first layer on a 20.0 mm part means 0.2.
  - With `brim_type` `brim_ears`, `brim_width` is the ear radius.
  - "N° from vertical" is `support_threshold_angle` = 90 − N.
  - "Manual supports" means `support_type` `normal(manual)` or `tree(manual)`.
  - `flush_into_objects` prints the objects in whatever filament is loaded; use `flush_into_infill`.
  - The print purges `flush_volumes_matrix` × `flush_multiplier` (0.3 on Bambu printers): a flush
    of V mm³ is the entry V / `flush_multiplier`.
- **Placement:** on the X1C rotate −45°, not +45° (front-left exclusion area). To fit a smaller
  printer use `object_scale_to_fit`, not a hand-made scale. Printing by object needs
  `extruder_clearance_radius` (68 mm on the X1C) between objects: `references/scene.md`.
- **"Slicing is in progress"** from `project_export` or `plate_slice_stats` right after an edit, a plate
  switch or `plate_slice {"plates":"all"}`: `app_wait {"for":"idle"}`, then repeat.
- **Prompts to pre-answer:** "too small" (inch model) → yes; STEP "import parameters" → ok, "floating
  parts" → no; spiral mode → yes; opening a `.3mf` over modified presets: "unsaved changes" →
  Discard.
- **Object overrides:** `settings_get {"target":"object","object":N}` lists the overrides with
  `effective` and `source`; `settings_set` with the same target and `"values":{"brim_type":null}`
  removes one. Only keys the object list offers there are taken.
- **Clicks on the canvas** under a panel or notification are refused, naming what covers it:
  enlarge the window (`window_set {"size":[1600,1000]}`) or turn the camera.

## When no tool reaches it

If an area file lists a task under "Not possible", don't probe the GUI for it; tell the user.
Computing over objects, meshes and settings, or the same calls over many objects, is a Python script
when the person allows them (`references/python.md`); a fixed sequence of calls to send at once or
run again is a replay (`references/replays.md`). What an installed plugin offers is reached through
`action_run` and `tool_call`, and a command, tab or G-code step the user wants to keep in the app is
a plugin you write for them (`references/plugins.md`). A system printer or filament that isn't
installed is `preset_select` by name for this session, and `printer_install` or `filament_install`
installs it for good (`references/settings.md`).
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
| Replays of tool calls: sending many calls at once, answering the dialogs they open, `expect` checks, `replay_export` | `references/replays.md` |
| Plugins: running an installed plugin's commands, pages and operations, turning plugins on and off, permission prompts; writing, installing, reloading and debugging a Python plugin | `references/plugins.md` |
| Speed Dial tiles: ids, the command and arguments each runs, what each does | `references/actions.md` |
| Reading a saved project file | `references/project-file.md` |
| Launching OrcaSlicer with MCP, reproducing issues, replays, tests, platform notes | `references/development.md` |

## GUI tools

- `widget_list {"query":"…"}` finds widgets, list rows, menus, GL toolbar items, plate icons, ImGui
  items and the elements of web pages (the setup wizard, sign-in, plugin windows). Prefer locators to
  refs: `{"label":…}`, `{"option":"layer_height"}` (a setting's field), `{"imgui":…}`,
  `{"imgui_name":…}`, `{"gizmo":…}`, `{"gltoolbar":…}`, `{"menu":"File>Export>Export G-code"}`,
  `{"row":"Cube>Generic-Cylinder"}`, `{"plate_icon":"lock","plate":0}`, `{"tab":"General"}`,
  `{"web":"Next"}`, `{"class":…,"index":N}`, with `"within":"<dialog title>"` for a control in a
  dialog. widget_list's description gives the full grammar.
- `window_list` lists the open windows, modal or not, with refs: a window's ref is the `target` of
  `window_screenshot` (that window alone), `widget_list` (only its widgets) and `window_set`, and its
  title the `within` of locators.
- Menus are never shown: a click that opens one replies `popup_menu`; the same call with
  `"popup_item":"Add Modifier>Cylinder"` runs an item (`ran`). Clicks on the model take points, not
  pixels: `mouse_gesture {"at":{"object":0,"point":[x,y,z]}}` (`references/gizmos.md`).
- When a call fails, `app_log {"level":"warning"}` says why.
