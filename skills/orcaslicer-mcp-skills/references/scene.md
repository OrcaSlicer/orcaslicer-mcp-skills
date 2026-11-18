# Scene: objects, parts of the scene, plates

Commands: `project_import`, `object_transform`, `object_arrange`, `object_list`, `plate_list`, and
the other `object_` and `plate_` commands, which act on the objects or plate they name, else on what
`object_select` picked or the current plate.

## How the scene is laid out

`object_list` is the Objects panel as a tree:

- `plates[i]` is `{index, name, objects}`; `outside` holds the objects on no plate. `{"plate":1}` lists
  that plate alone.
- An object has `instances` (its placements on that plate) and `parts`. An object with instances on
  two plates is listed under both; `total` counts it once.
- `parts` holds everything in the object, even when that is one part: parts, negative parts,
  modifiers, support blockers and enforcers and precise seams, told apart by `type`.
- `index` numbers an object in the project and a part in its object, whatever the plate. Those, and
  the `id` beside each, are what the other tools take: `"object":2`, `"part":1`, `"parts":[[2,1]]`.
- `plate_list` has each plate's settings and how many objects are on it (`objects`).

**Assembly.** When the person says "assembly" they mean one object with several parts: what Assemble
makes from several objects (`object_assemble`, named Assembly), what a STEP assembly opens as, and what
the Design tab commits for several bodies. `object_list` has no separate kind for it: it is an object
whose `parts` has more than one entry of `type` `part`. Work with it this way:

- The whole assembly: `object_transform {"object":N,…}`, `object_select {"objects":[N]}`, the target
  `object` of `settings_set`.
- One piece of it: `object_transform {"object":N,"part":M,…}`, `object_select {"parts":[[N,M]]}`,
  `settings_set {"target":"part","object":N,"part":M,…}` for its own filament and settings.
- Take it apart: `object_split {"object":N,"into":"objects"}`.

The assemble view (`view_set {"canvas":"assemble"}`) is something else: a 3D view that shows the
objects put together.

## Recipes

- Load: `project_import {"paths":["/abs/a.stl"]}`; a `.3mf` given to it adds its objects without its
  settings, and `project_open {"path":"/abs/p.3mf"}` opens it as a project. An import is one undo step
  and leaves the project with unsaved changes.
  Handy models: `object_add_handy_model {"model":"stanford_bunny"}` (also `3dbenchy`,
  `voron_cube`, `cali_cat`, `orca_badge`, `autodesk_fdm_test`, `orca_tolerance_test`). [ref]
- Inch or metre model: `dialog_auto_answer {"rules":[{"match":"too small","button":"yes"}]}` →
  `project_import` (Yes converts to mm). [ref]
- STEP assembly → separate objects: auto-answer `{"match":"Step file import parameters","button":"ok"}`
  and `{"match":"floating parts","button":"no"}` → `project_import` →
  `object_split {"object":0,"into":"objects"}` → `object_arrange {}`. [ref]
- Shape at a size: `object_add_primitive {"shape":"cube","size":[40,40,10]}` (cylinder, cone,
  disc `[D,H]`, sphere `[D]`, torus `[inner,tube]`; `"position"`, `"plate"`, `"name"` too). [ref]
- Scale, size, rotate, move (`object_transform` drops the object onto the bed and registers its plate):
  - `object_transform {"object":0,"scale":1.5}` is relative (×1.5); `"mode":"absolute"` sets it. [ref]
  - `object_transform {"object":0,"size":[80,null,null]}` = 80 mm long in proportion;
    `"size":[30,20,10]` per axis. [ref]
  - Upright + centred: `object_transform {"object":0,"rotate":[90,0,0]}` (relative), then
    `object_transform {"object":0,"translate":[128,128,0],"mode":"absolute"}` (X1C bed centre). [ref]
  - Diagonal fit on a square bed: `object_transform {"object":0,"rotate":[0,0,-45]}` (X1C: −45°, not
    +45°, which hits the front-left exclusion area). [ref]
  - Smaller printer: `preset_select` printer (it re-picks process and filament silently) →
    `object_scale_to_fit {"objects":[0]}`. [ref]
- Mirrored pair: `project_import` the file twice → `object_transform` `size` each →
  `object_mirror {"objects":[1],"axis":"x"}` → `object_arrange {}`; `object_list` shows
  `mirrored`. [ref]
- N copies spaced: `object_set_instances {"object":0,"count":6}` (total count) →
  `object_arrange {"plates":0,"spacing":10,"rotate":false}`. [ref] Fill the plate:
  `object_arrange {"fill_object":0}`; the reply's `added` counts the new instances. [ref]
- Separate named copies (each with its own settings): add the model once per copy, or set
  instances then `object_separate_instances {"object":0}`; then per object
  `object_rename {"object":i,"name":"infill-10"}`. [ref]
- Plates: `plate_add {}` → `object_transform {"object":2,"plate":1}` (same spot on the other plate) →
  `plate_list {}`. [ref]
- Arrange: `object_arrange {}` (every unlocked plate), `{"plates":1,"spacing":10,"rotate":false}` (one
  plate; the options stay saved as in the panel). It waits for the job and may renumber objects. [ref]
  - It shows the Prepare view first, so it runs from the assemble view, and with a gizmo open, which
    stays open. `fill_object` selects its object, so it is refused in the assemble view and while a
    gizmo is open. [ref]
  - A locked or an empty plate given by index is refused; `object_arrange {}` leaves a locked plate
    out and names it in `skipped_plates`. [ref]
- Print order by object: see the trap below; order with `object_order {"by":"name"}` (`height`,
  `size`, `filament`; `"descending":true`; `"all":true` for every plate; `"order":[2,0,1]` puts a
  plate's objects in the order given). `by` alone, as its Speed Dial action, reverses an order the
  objects already have; the reply's `order` and `descending` say what it did. [ref]
- Assemble objects into one: `object_assemble {"objects":[1,2]}` (the new object goes last). [ref]
- Split one object into parts: `object_split {"object":0,"into":"parts"}`.

## Traps

- By-object printing needs `extruder_clearance_radius` (68 mm on the X1C) between objects. With
  `print_sequence` `"by object"` set in the process, `object_arrange {}` with automatic spacing still
  left them "too close to others" at the slice; `object_arrange {"spacing":70}` fixed it. A plate whose
  own `print_sequence` (`plate_set_settings`) differs from the process is always arranged with
  automatic spacing (the reply's `note` says so): place its objects ≥ the radius apart with absolute
  `object_transform`s, plate 1's origin being 307.2 mm right of plate 0's on the X1C (read an offset
  from `object_list`). [ref]
- `object_list` rotation is folded Euler (−45° about Z reads `[180,-180,135]`); `bbox` is
  instance 0's; `mirrored` says left-handed, not which axis.
- `spot` targets aim along a ray through the bbox centre: a hole or gap there has no `spot`.
- Splitting into objects asks Split's question when pieces would float above the bed: give
  `"keep_z":true` (keep their height) or `false` (let them drop), or the call is refused
  (`auto_drop_question`).

## Not possible

Lay on face as a typed call (Place on Face gizmo or a computed `object_transform` `rotate`); several
files as one object (`project_import`, then `object_assemble`).
