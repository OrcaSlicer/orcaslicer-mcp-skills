# Scene: objects, parts of the scene, plates

Typed tools first: `open_files`, `transform`, `arrange`, `list_objects`, `list_plates`, and the
object commands of `references/actions.md`, which act on what `select` picked.

## Recipes

- Load: `open_files {"paths":["/abs/a.stl"]}`; a `.3mf` opens as a project unless `"as":"geometry"`.
  Handy models: `run_action {"id":"orca_command:add_handy_stanford_bunny:orca"}` (also `3dbenchy`,
  `voron_cube`, `cali_cat`, `orca_badge`, `autodesk_fdm_test`, `orca_tolerance_test`). [ref]
- Inch or metre model: `dialog_auto_answer {"rules":[{"match":"too small","button":"yes"}]}` →
  `open_files` (Yes converts to mm). [ref]
- STEP assembly → separate objects: auto-answer `{"match":"Step file import parameters","button":"ok"}`
  and `{"match":"floating parts","button":"no"}` → `open_files` → `select {"objects":[0]}` →
  `run_action {"id":"orca_command:obj_split_objects:orca"}` → `arrange {}`. [ref]
- Shape at a size, nothing selected: `select {"none":true}` →
  `run_action {"id":"orca_command:add_primitive_cube:orca","param":"40 40 10"}` (cylinder, cone,
  disc `"D H"`, sphere `"D"`, torus `"inner tube"`). [ref]
- Scale, size, rotate, move (`transform` drops the object onto the bed and registers its plate):
  - `transform {"object":0,"scale":1.5}` is relative (×1.5); `"mode":"absolute"` sets it. [ref]
  - `transform {"object":0,"size":[80,null,null]}` = 80 mm long in proportion;
    `"size":[30,20,10]` per axis. [ref]
  - Upright + centred: `transform {"object":0,"rotate":[90,0,0]}` (relative), then
    `transform {"object":0,"translate":[128,128,0],"mode":"absolute"}` (X1C bed centre). [ref]
  - Diagonal fit on a square bed: `transform {"object":0,"rotate":[0,0,-45]}` (X1C: −45°, not +45°,
    which hits the front-left exclusion area). [ref]
  - Smaller printer: `select_preset` printer (it re-picks process and filament silently) →
    `select {"objects":[0]}` → `run_action {"id":"orca_command:obj_fit_volume:orca"}`. [ref]
- Mirrored pair: `open_files` the file twice → `transform` `size` each → `select {"objects":[1]}` →
  `run_action {"id":"orca_command:obj_mirror_x:orca"}` → `arrange {}`; `list_objects` shows
  `mirrored`. [ref]
- N copies spaced: `select {"objects":[0]}` →
  `run_action {"id":"orca_command:obj_set_instances:orca","param":"6"}` (total count) →
  `arrange {"plates":0,"spacing":10,"rotate":false}`. [ref] Fill the plate:
  `arrange {"fill":0}`; the reply has `instances`. [ref]
- Separate named copies (each with its own settings): add the model once per copy, or set
  instances then `select {"objects":[0]}` →
  `run_action {"id":"orca_command:obj_instances_to_objects:orca"}`; then per object
  `select {"objects":[i]}` → `run_action {"id":"orca_command:obj_rename:orca","param":"infill-10"}`. [ref]
- Plates: `run_action {"id":"orca_command:plate_add:orca"}` →
  `transform {"object":2,"plate":1}` (same spot on the other plate) → `list_plates {}`. [ref]
- Arrange: `arrange {}` (every unlocked plate), `{"plates":1,"spacing":10,"rotate":false}` (one plate;
  the options stay saved as in the panel). It waits for the job and may renumber objects. [ref]
- Print order by object: see the trap below; order with
  `run_action {"id":"orca_command:obj_order_by_name:orca"}` (`_height`, `_size`, `_filament`; param
  `"all"` for every plate); the reply's `order` and `descending` say what it did. [ref]
- Assemble objects into one: `select {"objects":[1,2]}` →
  `run_action {"id":"orca_command:obj_assemble:orca"}` (the new object goes last). [ref]
- Split one object into parts: `run_action {"id":"orca_command:obj_split_parts:orca"}`.

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
