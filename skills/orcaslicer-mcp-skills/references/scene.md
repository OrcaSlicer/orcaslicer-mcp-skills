# Scene: objects, parts of the scene, plates

Typed tools first: `open_files`, `transform`, `arrange`, `list_objects`, `list_plates`, and the
object commands of `references/actions.md`, which act on what `select_objects` picked.

## How the scene is laid out

`list_objects` is the Objects panel as a tree:

- `plates[i]` is `{index, name, objects}`; `outside` holds the objects on no plate. `{"plate":1}` lists
  that plate alone.
- An object has `instances` (its placements on that plate) and `parts`. An object with instances on
  two plates is listed under both; `total` counts it once.
- `parts` holds everything in the object, even when that is one part: parts, negative parts,
  modifiers, support blockers and enforcers and precise seams, told apart by `type`.
- `index` numbers an object in the project and a part in its object, whatever the plate. Those, and
  the `id` beside each, are what the other tools take: `"object":2`, `"part":1`, `"parts":[[2,1]]`.
- `list_plates` has each plate's settings and how many objects are on it (`objects`).

**Assembly.** When the person says "assembly" they mean one object with several parts: what Assemble
makes from several objects (`obj_assemble`, named Assembly), what a STEP assembly opens as, and what
the Design tab commits for several bodies. `list_objects` has no separate kind for it: it is an object
whose `parts` has more than one entry of `type` `part`. Work with it this way:

- The whole assembly: `transform {"object":N,…}`, `select_objects {"objects":[N]}`, the object scope of
  `set_settings`.
- One piece of it: `transform {"object":N,"part":M,…}`, `select_objects {"parts":[[N,M]]}`,
  `set_settings {"scope":"part","object":N,"part":M,…}` for its own filament and settings.
- Take it apart: `select_objects {"objects":[N]}` → `run_action {"id":"obj_split_objects"}`.

The assemble view (`set_view {"canvas":"assemble"}`) is something else: a 3D view that shows the
objects put together.

## Recipes

- Load: `open_files {"paths":["/abs/a.stl"]}`; a `.3mf` opens as a project unless `"as":"geometry"`,
  which adds its objects without its settings, as
  `run_action {"id":"import_file","args":{"path":"/abs/p.3mf"}}` does. An import is one undo step and leaves
  the project with unsaved changes.
  Handy models: `run_action {"id":"add_handy_stanford_bunny"}` (also `3dbenchy`,
  `voron_cube`, `cali_cat`, `orca_badge`, `autodesk_fdm_test`, `orca_tolerance_test`). [ref]
- Inch or metre model: `auto_answer_dialogs {"rules":[{"match":"too small","button":"yes"}]}` →
  `open_files` (Yes converts to mm). [ref]
- STEP assembly → separate objects: auto-answer `{"match":"Step file import parameters","button":"ok"}`
  and `{"match":"floating parts","button":"no"}` → `open_files` → `select_objects {"objects":[0]}` →
  `run_action {"id":"obj_split_objects"}` → `arrange {}`. [ref]
- Shape at a size, nothing selected: `select_objects {"none":true}` →
  `run_action {"id":"add_primitive_cube","args":{"size":[40,40,10]}}` (cylinder, cone,
  disc `[D,H]`, sphere `[D]`, torus `[inner,tube]`). [ref]
- Scale, size, rotate, move (`transform` drops the object onto the bed and registers its plate):
  - `transform {"object":0,"scale":1.5}` is relative (×1.5); `"mode":"absolute"` sets it. [ref]
  - `transform {"object":0,"size":[80,null,null]}` = 80 mm long in proportion;
    `"size":[30,20,10]` per axis. [ref]
  - Upright + centred: `transform {"object":0,"rotate":[90,0,0]}` (relative), then
    `transform {"object":0,"translate":[128,128,0],"mode":"absolute"}` (X1C bed centre). [ref]
  - Diagonal fit on a square bed: `transform {"object":0,"rotate":[0,0,-45]}` (X1C: −45°, not +45°,
    which hits the front-left exclusion area). [ref]
  - Smaller printer: `select_preset` printer (it re-picks process and filament silently) →
    `select_objects {"objects":[0]}` → `run_action {"id":"obj_fit_volume"}`. [ref]
- Mirrored pair: `open_files` the file twice → `transform` `size` each → `select_objects {"objects":[1]}` →
  `run_action {"id":"obj_mirror_x"}` → `arrange {}`; `list_objects` shows
  `mirrored`. [ref]
- N copies spaced: `select_objects {"objects":[0]}` →
  `run_action {"id":"obj_set_instances","args":{"count":6}}` (total count) →
  `arrange {"plates":0,"spacing":10,"rotate":false}`. [ref] Fill the plate:
  `arrange {"fill":0}`; the reply has `instances`. [ref]
- Separate named copies (each with its own settings): add the model once per copy, or set
  instances then `select_objects {"objects":[0]}` →
  `run_action {"id":"obj_instances_to_objects"}`; then per object
  `select_objects {"objects":[i]}` → `run_action {"id":"obj_rename","args":{"name":"infill-10"}}`. [ref]
- Plates: `run_action {"id":"plate_add"}` →
  `transform {"object":2,"plate":1}` (same spot on the other plate) → `list_plates {}`. [ref]
- Arrange: `arrange {}` (every unlocked plate), `{"plates":1,"spacing":10,"rotate":false}` (one plate;
  the options stay saved as in the panel). It waits for the job and may renumber objects. [ref]
  - It shows the Prepare view first, so it runs from the assemble view, and with a gizmo open, which
    stays open, as `run_action {"id":"obj_arrange"}` does. `fill` selects its object, so it is refused
    in the assemble view and while a gizmo is open. [ref]
  - A locked or an empty plate given by index is refused; `arrange {}` leaves a locked plate out and
    names it in `skipped_plates`. [ref]
- Print order by object: see the trap below; order with
  `run_action {"id":"obj_order_by_name"}` (`_height`, `_size`, `_filament`;
  `"args":{"all":true}` for every plate); the reply's `order` and `descending` say what it did. [ref]
- Assemble objects into one: `select_objects {"objects":[1,2]}` →
  `run_action {"id":"obj_assemble"}` (the new object goes last). [ref]
- Split one object into parts: `run_action {"id":"obj_split_parts"}`.

## Traps

- By-object printing needs `extruder_clearance_radius` (68 mm on the X1C) between objects. With
  `print_sequence` `"by object"` set in the process, `arrange {}` with automatic spacing still left
  them "too close to others" at the slice; `arrange {"spacing":70}` fixed it. A plate whose own
  `print_sequence` (plate scope) differs from the process is always arranged with automatic
  spacing (the reply's `note` says so): place its objects ≥ the radius apart with absolute
  `transform`s, plate 1's origin being 307.2 mm right of plate 0's on the X1C (read an offset from
  `list_objects`). [ref]
- `list_objects` rotation is folded Euler (−45° about Z reads `[180,-180,135]`); `bbox` is
  instance 0's; `mirrored` says left-handed, not which axis.
- `spot` targets aim along a ray through the bbox centre: a hole or gap there has no `spot`.

## Not possible

Lay on face as a typed call (Place on Face gizmo or a computed `transform rotate`); orient with a
result (`obj_orient` replies only success); several files as one object (`open_files`, then
`obj_assemble`).
