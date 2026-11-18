# Design (CAD) tab

The Design tab's commands are not in the tool list: every call is
`tool_call {"tool":"<name>","args":{...}}`, which the recipes below write short, as `<name> {args}`,
and its reply is `{"result":{…}}`. A replay step or a script's `orca.host.tools.call` names the
command itself (`{"tool":"extrude","args":{…}}`). The CAD feature must be on when OrcaSlicer starts
(Preferences > Features, then a restart, or `--cad` on its command line; `app_state.design_tab` says
so, and `cad_note` how to turn it on).
Look up a command's arguments and examples with `tool_search {"tool":"extrude"}`, find one with
`tool_search {"group":"Design","query":"sketch"}` (each match with its schema), and list every
name with `tool_search {"group":"Design"}`; `tool_search`' own description lists them by heading.

## Names

- The names use the industry's terminology, as Fusion 360, Onshape and SolidWorks use it: `extrude`,
  `fillet`, `shell`, `split`, `align`, `sketch_finish`, `feature_suppress`, `surface_patch`,
  `sketch_blend_curve`. No prefix is the 3D model; `sketch_` works in the open sketch, `surface_` makes
  sheet bodies, `construct_` reference planes, axes and coordinate systems, `feature_` one entry of the
  history, `body_` one body (`body_set`, `body_place`, `body_describe`, `body_mass_properties`).
- One argument picks the variant: `type` for what kind of thing a command makes (`sketch_draw`'s shape,
  `hole`'s seat, `pattern`'s layout, `align`'s joint, `thicken`'s face or surface), `operation` for what the
  result does to the bodies already there: `new_body`, `join`, `cut`, `intersect` (`extrude` also
  `auto`, the body the profile touches). A command reads a guessed `type` as its `operation` and the
  other way round. [ref] An argument a command does not take is refused where its schema closes its
  list of arguments (`extrude`, `fillet`, `hole`, `rename_selected`); many others (`helix`, `shell`,
  `body_transform`, `construct_plane`, the reads `describe_design`, `body_describe`) ignore it, so
  check each name against `tool_search {"tool":…}` before relying on it.

## Two forms

- Given arguments, a command runs at once, as the recipes do. Called with exactly the arguments that
  choose one of its items in the Design tab's right-click menu (none for the command's own item), it
  does what that item does for a person: it opens a feature card or a dialog, starts a sketch tool,
  acts at once, or declines. The reply is
  `{"result":{"interactive":true,"tool":…,"applies":…,"modal":…,"selection_kind":…,"busy":…,"status":…}}`
  and claims nothing about which: `busy` names the card, dialog or sketch left open (null when
  none) and `status` is the tab's status line afterwards. A command's description ends with the
  argument sets that open its items. [ref]
  - A card: with a committed sketch, `extrude {}` opens the Extrude card
    (`"busy":"a feature card is open in the Design tab"`), from any tab, and every change command is
    refused ("Design tab busy: a feature card is open in the Design tab") until the card closes:
    `confirm_tool {}` confirms it as Enter does, `cancel_tool {}` cancels it. With two bodies,
    `boolean {}` opens the Boolean card on Join and `boolean {"operation":"cut"}` on Subtract,
    whatever is selected. [ref]
  - A dialog: in a sketch `sketch_text {}` opens the modeless Text dialog and returns with it open
    (`"modal":false`, `busy` names it): `cancel_tool {}` closes it as its Cancel does, and once text is
    typed in it `confirm_tool {}` takes its OK, which opens the text's placement (`"open":"insert"`), while
    an item whose dialog is modal (`body_set {}`, the Color item, with a body selected, and
    `sketch_insert_svg {}`) holds the call, which replies `blocked_by_dialog` with a job first, as any
    command that asks does; the SVG file picker is native, so register a `dialog_auto_answer` rule
    before calling it (SKILL.md, Dialogs). [ref] `body_set {}` opens the system's colour picker, which
    `dialog_answer` does not close on Windows: colour a body with `body_set {"body":…,"color":"#rrggbb"}`
    instead.
  - Declined: on an empty design `extrude {}` builds nothing, with `"busy":null` and
    `"status":"✗ Create a sketch, or pick a solid face, first"`. [ref]
  - A sketch tool: in a sketch `sketch_draw {"type":"rectangle_2_point"}` arms the rectangle tool
    (`busy` names the open sketch, `status` the tool's hint), for the person's clicks; each `type`
    arms its own tool, and with entities selected `sketch_pattern {"type":"circular"}` the Polar array. [ref]
  - At once: an item that acts on the selection asks nothing. A Delete item (`feature_delete {}`,
    `body_delete {}`, `sketch_delete {}`) deletes what is selected; `sketch_delete {}` with a sketch line
    selected removes it. Read the design back (`describe_design`, `sketch_describe`) to see what such a
    call changed.
- So a call meant to run must give more than an item's arguments: `boolean {"operation":"cut"}` alone
  opens the card, `boolean {"operation":"cut","target":0,"tool":1}` cuts. A command that has both
  forms (`silent` in its schema) and whose other arguments all have defaults takes `{"silent":true}`
  alone to run with them: `extrude {"silent":true}` builds the default 20 × 20 × 10 mm box. One with a
  required argument is given that argument, which runs it silently too: `fillet {"silent":true}` is
  refused ("edges is required"), `fillet {"edges":[0],"radius":1,"body":0}` builds. [ref]
- Called to open a tool, an item runs only in its mode: a model tool in an open sketch is refused
  ("finish or cancel the sketch first", `body_mass_properties {}`), a sketch tool outside one ("open a
  sketch first"). One that only the right-click menu offers is refused for a selection it doesn't take,
  with `accepts` and the `selection` in the error (`body_mass_properties {}` with nothing selected);
  `describe_design`'s `selection` is what is selected. [ref]
- Given arguments, a command goes by what it does, not by mode: a read runs in an open sketch too
  (`body_mass_properties {"body":0}`); a change to the model is refused while a sketch, card or dialog
  is open ("Design tab busy: a sketch is open in the Design tab" for
  `fillet {"edges":[0],"radius":1,"body":0}` in a sketch); the sketch commands work on the open
  sketch. [ref]
- `tool_search {"tool":…}` shows which form a command has: `silent` in its schema for both; no
  arguments at all for one that only does what its menu item does (`rename_selected`,
  `zoom_to_selection`);
  neither for one with a silent form only, which runs whatever it is given (`describe_design`,
  `body_describe`, `commit_to_plate`), or opens an item's tool only with that item's `type`
  (`sketch_draw`, `sketch_transform`, `sketch_pattern`).
- Select for the menu items: `select_geometry {"body":0,"face":5}` (also `edges`, `vertex`, `plane`,
  `sketch` with `loop`, `feature`; `{}` clears) selects as a click does, so the items that take a face act
  on it, `zoom_to_selection {}` among them; `rename_selected {}` takes a body (`{"body":0}`) or a loop.
  A Feature-tree row selected with `select_geometry {"feature":i}` is what `feature_edit {}`,
  `feature_delete {}`, `rename_selected {}` and `sketch_constrain_selected {}` act on. [ref]

## Frames

- Base planes `"XY"`, `"XZ"`, `"YZ"` pass through `describe_design.modeling_origin` (the bed centre,
  128,128 on the X1C) for sketches and features alike: their x/y are offsets from it. On `"XZ"` the
  sketch's v is world Z and an extrude goes +Y. `{"datum":i}` is a Plane feature, `{"body":b,"face":f}`
  a planar face. Replies carry the `frame` used.
- `body_describe`, `measure`, `describe_design` bboxes are world coordinates. Imported STEP/STL keep
  the file's coordinates.

## Recipes

- Box or profile solid: `extrude {"width":60,"height":40,"distance":5}`;
  `{"plane":"XZ","profile":[[x,z],…],"distance":20}`; `"operation":"new_body"|"join"|"cut"`. [ref]
- Sketch → solid: `sketch_create {"plane":"XY"}` →
  `sketch_draw {"type":"rectangle_2_point","p0":[x0,y0],"p1":[x1,y1]}` and/or
  `sketch_draw {"entities":[…]}` → `sketch_finish` (reply: `sketch_index`, `closed_loops`, `buildable`)
  → `extrude {"sketch":0,"distance":8}` (`"loop":i` one region; by default it joins the body the
  sketch touches) or `revolve {"sketch":0}` (about the sketch's only construction line). [ref]
  - Shapes, each `{"type":…}` with its fields: `line` `p0`, `p1`; `polyline` `points`;
    `rectangle_2_point` `p0`, `p1`; `rectangle_center` `center`, `corner`; `rectangle_3_point` `p0`,
    `p1`, `width`; `rectangle_rounded` `p0`, `p1`, `radius`; `circle_center` `center`, `radius`;
    `circle_2_point` `p0`, `p1`; `circle_3_point` `p0`, `p1`, `p2`; `arc_3_point` `p0`, `p1`, `on_arc`;
    `arc_center` `center`, `radius`, `start_angle`, `end_angle` (radians, or `_deg` names); `arc_tangent`
    `end` and `entity`; `slot` `c0`, `c1`, `width`; `slot_arc` `center`, `start`, `end`, `width`; `ellipse`
    `center`, `radius`, `rminor`, `rotation_deg`; `ellipse_arc`; `spline` `ctrl`; `polygon`
    `{"type":"polygon","center":[0,0],"sides":6,"across_flats":8}`; `point` `p`; `"construction":true`
    for an axis. Each is built as its tool builds it, with the constraints its tool gives it at any slant, and
    one undo takes it off, a polyline its last line only. A constraint between a shape and the sketch's axes
    or what is already in the sketch, or between plain lines, is inferred only where it holds exactly, so
    that none moves a point given. [ref]
  - Corners: `sketch_fillet {"entities":[0,1],"radius":5}` (two lines meeting at a corner; a
    rectangle's lines are 0–3), `sketch_chamfer {"entities":[0,1],"distance":5}`. Also
    `sketch_trim {"at":[x,y]}`, `sketch_extend {"at":[x,y]}`,
    `sketch_transform {"type":"move","dx":5,"entities":[0]}` (`rotate`, `scale`),
    `sketch_pattern {"type":"circular","count":6,"angle_deg":360,"pivot":[0,0],"entities":[0]}`
    (`linear` with `dx`, `dy`), `sketch_mirror {"axis":"y_axis","entities":[0]}`,
    `sketch_offset {"distance":2,"entities":[0]}`, `sketch_constrain`, `sketch_dimension`. [ref]
  - On a face: `sketch_create {"plane":{"body":0,"face":10}}`.
  - Change a committed sketch: `sketch_edit {"sketch":0}` → the sketch commands → `sketch_finish`, which
    replaces it in one undo step (`replaced: true`) and rebuilds what is built on it; `sketch_cancel`
    drops the changes. [ref]
  - Text and SVG: `sketch_text {"text":"ORCA","height":10,"plane":"XY"}` (on the model, a face plane
    too, `font` a face name); `sketch_insert_svg {"path":"/abs/art.svg"}` into the open sketch, or on
    the model with `plane`. [ref]
- Upright round part: `revolve {"plane":"XZ","axis":1,"angle":360,"profile":[[r_in,0],[r_out,0],[r_out,h],[r_in,h]]}`. [ref]
- Holes: `hole {"diameter":3.4,"through":true,"x":25,"y":15}` (plane coords); blind
  `{"diameter":6.2,"depth":12,"x":0,"y":0}` from z 0 up;
  `hole {"designation":"M3","through":true,"x":…,"y":…}` a standard clearance hole,
  `"type":"countersink"` or `"counterbore"` with its seat (`csink_*` or `cbore_*`, which another type
  refuses); `"plane":"YZ"` drills +X; on a
  face `"plane":{"body":0,"face":0}` with `"at":{"body":0,"face":6}` (a bore whose axis it shares) or a
  world point. [ref]
- Thread: `body_describe {"kinds":["cylindrical"]}` → `thread {"face":6,"generation":g}` taps a
  bore with the nearest standard over its length (an M5 tap-drill bore → M5×0.8). On a plane:
  `thread {"plane":"XY","designation":"M5","internal":true,"length":15}`. Can return a job. [ref]
- Fillet / chamfer / shell: `body_describe {"body":0,"convex":false}` (inside edges),
  `{"direction":[0,0,1]}`, `{"at_extreme":"+z"}` (faces), `{"edge_kinds":["line"]}` →
  `fillet {"edges":[7],"radius":5,"body":0}`, `chamfer {"edges":[1,2],"distance":0.5}`, or a card's
  Edges choice, `fillet {"edges":"top","radius":1}` (`bottom`, `lateral`, `all`);
  `shell {"thickness":2,"face":5,"body":0}` (open face). Ids expire at every rebuild: re-query, or
  pass `generation`. [ref]
- Circular pattern, fused: hub as body 0, the tooth as a new body (`"operation":"new_body"`) →
  `pattern {"type":"circular","count":6,"angle":360}` (copies go into the last body) →
  `boolean {"operation":"join","target":0,"tool":1}`. [ref] A bolt circle of holes: one `hole` per
  computed x, y.
- Variables: `set_variable {"name":"w","expr":"30"}` →
  `feature_set_expression {"feature":0,"field":"width","expr":"w"}` (a box extrude's width is on its
  Sketch, feature 0) → `set_variable {"name":"w","expr":"45"}`; `describe_design` lists `variables` and
  each feature's `bound` fields. [ref]
- Edit an existing design: `describe_design` → `feature_describe {"feature":1}` (`params`, `editable`,
  `ranges`, `refs`) → `feature_edit {"feature":1,"fields":{"distance":15}}`; a hole's `hole_diameter`.
  What a feature refers to changes with `refs`, its ids read in the model just before the feature:
  `body_describe {"body":0,"before":2}` → `feature_edit {"feature":2,"refs":{"edges":[4]}}`
  (`feature_describe`'s `ids_of` says which model a feature's ids are in). [ref]
- Spring: `helix {"radius":10,"pitch":5,"height":30}` → `sketch_create {"plane":"XZ"}` →
  `sketch_draw {"type":"circle_center","center":[10,0],"radius":1.5}` → `sketch_finish` →
  `sweep {"sketch":1,"path":0}`. Loft: sketches on `"XY"` and on
  `construct_plane {"type":"offset","base":"XY","offset":40}`'s datum → `loft {"profiles":[1,2]}`.
  Split: `split {"plane":"XY","offset":10}` (`flip`, `keep_upper`/`keep_lower`).
- Imported parts: `import_step {"path":…}`, `import_mesh {"path":…}`; check with
  `body_mass_properties {"body":0}` (`volume`), `measure {"a":{"face":14,"axis":true},"b":{"face":10}}`
  (`min_distance`), `body_describe` radii.
- Commit to the plate: `commit_to_plate {}` (one object; `"mode":"bodies"` one per body;
  `"replace":[i]` deletes those Prepare objects in the same undo step, e.g. the stale object of an
  earlier commit) → `object_list`. `export_step {"path":"/abs/x.step","overwrite":true}`. [ref]
- Move or turn a body as a feature: `body_transform {"body":0,"dx":30}` (`dy`, `dz`; `angle` about
  `axis_x/y/z` through `pivot_x/y/z`; `"copy":true` adds the moved copy as a new body). Place it for the
  plate only, outside the history: `body_place {"body":0,"move":[30,0,0]}` (`rotate`, `face` to lay on
  the bed, `"reset":true`); `describe_design` gives each body's `placement`. [ref]
- History: `undo {"history":"design"}` / `redo {"history":"design"}` (reply `history` and `result`
  with `undo_depth`, `redo_depth`; inside a sketch it takes back the last shape drawn); `feature_delete`,
  `feature_move {"feature":2,"delta":-1}`, `feature_suppress {"feature":2,"suppressed":true}`,
  `feature_rename`; `body_set {"body":1,"name":"Lid","color":"#3080ff","visible":false}` (a hidden body
  is not committed), `body_delete`. `new_design {"discard_changes":true}` starts over.
- A gear or other generated outline: compute the points and `extrude {"profile":[…]}`.

## Traps

- A sketch line's ends are `p0` and `p1`; `start`/`end` is refused.
- The commands whose description says "A rejected rebuild is reported in the reply" (`boolean`,
  `shell`, `mirror`, `draft`, `body_transform`, `set_variable`, `feature_set_expression`, `import_*`,
  the surface tools, `sketch_finish` of a re-opened sketch…) answer
  `{"result":{"ok":false,"error":…}}` instead of failing: check `ok`. The others, `fillet`, `chamfer`,
  `pattern` and `extrude` among them, fail the call with the kernel's reason. `confirm_tool` answers
  `"ok":false` with the reason in `status` when the Confirm does not take.
- `undo` and `redo` without `history` act on the tab shown: driving the Design tab from Prepare, a
  bare `undo {}` takes back a project change. Pass `"history":"design"`.
- A call whose arguments are only those that choose a menu item does what the item does, not its
  silent form: it may open a card, decline, or act on the selection at once, and a Delete item deletes
  with no prompt (Two forms above). To build, pass the arguments, or `{"silent":true}` where
  nothing else is required.
- `action_run` reaches a Design tab action only while the Design tab is shown in the action's mode;
  `tool_call` reaches every Design command from any tab, the tab need not be shown. An action of an item
  folded into a command (Polar array, `sk_array_polar`) runs that command with the item's arguments.
- Face and edge ids from `body_describe` expire when the model rebuilds; deleting, moving or
  suppressing features renumbers features and bodies.
- Design-only edits mark the project modified (`app_state.project.dirty`), so `project_new`
  and `project_open` after them need `"discard_changes":true`. [ref]
- Commit appends: without `replace`, the earlier committed object stays on the plate.

## Not possible

Feature patterns (bodies only); sheet metal and drawings.
