# Design (CAD) tab

The Design tab's commands are not in the tool list: every call is
`call_tool {"tool":"<name>","args":{...}}`, which the recipes below write short, as `<name> {args}`,
and its reply is `{"result":{…}}`. A replay step or a script's `orca.host.tools.call` names the
command itself (`{"tool":"extrude","args":{…}}`). The CAD feature must be on when OrcaSlicer starts
(Preferences > Features, then a restart, or `--cad` on its command line; `get_app_state.design_tab` says
so, and `cad_note` how to turn it on).
Look up a command's arguments and examples with `search_tools {"tool":"extrude"}`, find one with
`search_tools {"group":"Design","query":"sketch"}` (each match with its schema), and list every
name with `search_tools {"group":"Design"}`.

## Two forms

- Given arguments, a command runs at once, as the recipes do. Given none, a command with a row in
  the Design tab's right-click menu does what that row does for a person: it opens a
  feature card or a dialog, starts a sketch tool, acts at once, or declines. The reply is
  `{"result":{"interactive":true,"tool":…,"applies":…,"modal":…,"selection_kind":…,"busy":…,"status":…}}`
  and claims nothing about which: `busy` names the card, dialog or sketch left open (null when
  none) and `status` is the tab's status line afterwards. [ref]
  - A card: with a committed sketch, `extrude {}` opens the Extrude card
    (`"busy":"a feature card is open in the Design tab"`), from any tab, and every change command is
    refused ("Design tab busy: a feature card is open in the Design tab") until the card closes:
    `set_view {"tab":"design"}`, then `click {"target":{"label":"Cancel"}}` (the Design ribbon's
    Cancel, which no other tab shows). [ref]
  - A dialog: `sk_text {}` opens the modeless Text dialog and returns with it open (`"modal":false`,
    `busy` names it; `click {"target":{"label":"Cancel","within":"Text"}}` closes it), while a row
    whose dialog is modal (`colour`, `sk_svg`) holds the call, which replies `blocked_by_dialog` with a
    job first, as any command that asks does; `sk_svg`'s file picker is native, so register an
    `auto_answer_dialogs` rule before calling it (SKILL.md, Dialogs). [ref] `colour {}` opens the
    system's colour picker, which `answer_dialog` does not close on Windows: colour a body with
    `body_set {"body":…,"color":"#rrggbb"}` instead.
  - Declined: on an empty design `extrude {}` builds nothing, with `"busy":null` and
    `"status":"✗ Create a sketch, or pick a solid face, first"`.
  - A sketch tool: in a sketch `sk_rect {}` arms the rectangle tool (`busy` names the open sketch,
    `status` the tool's hint), for the person's clicks.
  - At once: a row that acts on the selection asks nothing. A Delete row (`delete`, `delete_body`,
    `sk_delete`) deletes what is selected; `sk_delete {}` with a sketch line selected removes it.
    Read the design back (`describe_scene`, `sketch_describe`) to see what such a call changed.
- A command that has both forms (`silent` in its schema) and whose other arguments all have
  defaults takes `{"silent":true}` alone to run with them: `extrude {"silent":true}` builds the
  default 20 × 20 × 10 mm box. One with a required argument is given that argument, which runs it
  silently too: `fillet {"silent":true}` is refused ("edge is required"),
  `fillet {"edge":[0],"radius":1,"body":0}` builds. [ref]
- Called with no arguments, a row runs only in its mode: a model tool in an open sketch is refused
  ("finish or cancel the sketch first", `mass_props {}`), a sketch tool (`sk_rect`, `sk_circle`…)
  outside one ("open a sketch first"). One that only the right-click menu offers is refused for a
  selection it doesn't take, with `accepts` and the `selection` in the error (`mass_props {}` with
  nothing selected); `describe_scene`'s `selection` is what is selected. [ref]
- Given arguments, a command goes by what it does, not by mode: a read runs in an open sketch too
  (`mass_props {"body":0}`); a change to the model is refused while a sketch, card or dialog is
  open ("Design tab busy: a sketch is open in the Design tab" for
  `fillet {"edge":[0],"radius":1,"body":0}` in a sketch); the sketch commands work on the open
  sketch. [ref]
- `search_tools {"tool":…}` shows which form a command has: `silent` in its schema for both; no
  arguments at all for one that only does what its row does (`sk_rect`, `sketch`, `delete`);
  neither for one with a silent form only, which runs whatever it is given (`sketch_begin`,
  `query_topology`, `commit`).

## Frames

- Base planes `"XY"`, `"XZ"`, `"YZ"` pass through `describe_scene.modeling_origin` (the bed centre,
  128,128 on the X1C) for sketches and features alike: their x/y are offsets from it. On `"XZ"` the
  sketch's v is world Z and an extrude goes +Y. `{"datum":i}` is a Plane feature, `{"body":b,"face":f}`
  a planar face. Replies carry the `frame` used.
- `query_topology`, `measure`, `describe_scene` bboxes are world coordinates. Imported STEP/STL keep
  the file's coordinates.

## Recipes

- Box or profile solid: `extrude {"width":60,"height":40,"distance":5}`;
  `{"plane":"XZ","profile":[[x,z],…],"distance":20}`; `"boolean":"new"|"union"`. [ref]
- Sketch → solid: `sketch_begin {"plane":"XY"}` → `sketch_add {"rect":[x0,y0,x1,y1]}` and/or
  `sketch_add {"entities":[…]}` → `sketch_commit` (reply: `sketch_index`, `closed_loops`, `buildable`)
  → `extrude {"sketch":0,"distance":8}` (`"loop":i` one region; by default it joins the body the
  sketch touches) or `revolve {"sketch":0}` (about the sketch's only construction line). [ref]
  - Entities: line `{"type":"line","p0":[x,y],"p1":[x,y]}`; circle `center`, `radius`; arc `center`,
    `radius`, `start_angle`, `end_angle` (radians, or `_deg` names); polygon
    `{"type":"polygon","center":[0,0],"sides":6,"across_flats":8}`; slot `c0`, `c1`, `width`; ellipse
    `center`, `radius`, `rminor`, `rotation_deg`; spline `ctrl`; `"construction":true` for an axis.
  - Corners: `sketch_fillet {"entities":[0,1],"radius":5}` (two lines meeting at a corner; a rect's
    lines are 0–3), `sketch_chamfer {"entities":[0,1],"distance":5}`. Also `sketch_trim`,
    `sketch_transform` (`move`, `rotate`, `array`, `polar_array`), `sketch_constrain`. [ref]
  - On a face: `sketch_begin {"plane":{"body":0,"face":10}}`.
- Upright round part: `revolve {"plane":"XZ","axis":1,"angle":360,"profile":[[r_in,0],[r_out,0],[r_out,h],[r_in,h]]}`. [ref]
- Holes: `hole {"diameter":3.4,"through":true,"x":25,"y":15}` (plane coords); blind
  `{"diameter":6.2,"depth":12,"x":0,"y":0}` from z 0 up; `hole_standard {"designation":"M3","through":true,
  "x":…,"y":…}`, `"style":2` countersink, `"style":1` counterbore; `"plane":"YZ"` drills +X; on a
  face `"plane":{"body":0,"face":0}` with `"at":{"body":0,"face":6}` (a bore whose axis it shares) or a
  world point. [ref]
- Thread: `query_topology {"kinds":["cylindrical"]}` → `thread {"face":6,"generation":g}` taps a
  bore with the nearest standard over its length (an M5 tap-drill bore → M5×0.8). On a plane:
  `thread {"plane":"XY","designation":"M5","internal":true,"length":15}`. Can return a job. [ref]
- Fillet / chamfer / shell: `query_topology {"body":0,"convex":false}` (inside edges), `{"direction":[0,0,1]}`,
  `{"at_extreme":"+z"}` (faces), `{"edge_kinds":["line"]}` → `fillet {"edge":[7],"radius":5,"body":0}`,
  `chamfer {"edge":[1,2],"distance":0.5}`, `shell {"thickness":2,"face":5,"body":0}` (open face). Ids
  expire at every rebuild: re-query, or pass `generation`. [ref]
- Circular pattern, fused: hub as body 0, the tooth as a new body (`"boolean":"new"`) →
  `pattern {"circular":true,"count":6,"angle":360}` (copies go into the last body) →
  `boolean {"op":"union","target":0,"tool":1}`. [ref] A bolt circle of holes: one `hole` per
  computed x, y.
- Variables: `set_variable {"name":"w","expr":"30"}` → `edit_feature {"feature":0,"field":"width","expr":"w"}`
  (a box extrude's width is on its Sketch, feature 0) → `set_variable {"name":"w","expr":"45"}`;
  `describe_scene` lists `variables` and each feature's `bound` fields. [ref]
- Edit an existing design: `describe_scene` → `describe_feature {"feature":1}` (`params`, `editable`,
  `ranges`) → `set_feature {"feature":1,"fields":{"distance":15}}`; a hole's `hole_diameter`. [ref]
- Spring: `helix {"radius":10,"pitch":5,"height":30}` → `sketch_begin {"plane":"XZ"}` →
  `sketch_add {"type":"circle","center":[10,0],"radius":1.5}` → `sketch_commit` →
  `sweep {"sketch":1,"path":0}`. Loft: sketches on `"XY"` and on `plane {"type":"offset","base":"XY","offset":40}`'s
  datum → `loft {"profiles":[1,2]}`. Split: `split {"plane":"XY","offset":10}` (`flip`,
  `keep_upper`/`keep_lower`).
- Imported parts: `import_step {"path":…}`, `import_mesh {"path":…}`; check with
  `mass_props {"body":0}` (`volume`), `measure {"a":{"face":14,"axis":true},"b":{"face":10}}`
  (`min_distance`), `query_topology` radii.
- Commit to the plate: `commit {}` (one object; `"mode":"bodies"` one per body;
  `"replace":[i]` deletes those Prepare objects in the same undo step, e.g. the stale object of an
  earlier commit) → `list_objects`. `export_step {"path":"/abs/x.step","overwrite":true}`. [ref]
- Move or turn a body: `body_transform {"body":0,"dx":30}` (`dy`, `dz`; `angle` about
  `axis_x/y/z` through `pivot_x/y/z`; `"copy":true` adds the moved copy as a new body). [ref]
- History: `undo {"history":"design"}` / `redo {"history":"design"}` (reply `history` and `result`
  with `undo_depth`, `redo_depth`; inside a sketch it takes back the last shape drawn); `feature_delete`,
  `feature_move {"feature":2,"delta":-1}`, `feature_set_enabled`, `feature_rename`;
  `body_set {"body":1,"name":"Lid","color":"#3080ff","visible":false}` (a hidden body is not
  committed), `body_delete`. `new_design {"discard_changes":true}` starts over.
- A gear or other generated outline: compute the points and `extrude {"profile":[…]}`.

## Traps

- A sketch line's ends are `p0` and `p1`; `start`/`end` is refused.
- The commands whose description says "A rejected rebuild is reported in the reply" (`fillet`,
  `chamfer`, `boolean`, `pattern`, `shell`, `mirror`, `body_transform`, `set_variable`,
  `edit_feature`, `import_*`, the surface tools…) answer `{"result":{"ok":false,"error":…}}` instead
  of failing: check `ok`.
- `undo` and `redo` without `history` act on the tab shown: driving the Design tab from Prepare, a
  bare `undo {}` takes back a project change. Pass `"history":"design"`.
- A command with a menu row, called with no arguments, does what the row does, not its silent
  form: it may open a card, decline, or act on the selection at once, and a Delete row deletes
  with no prompt (Two forms above). To build, pass the arguments, or `{"silent":true}` where
  nothing else is required.
- `run_action` reaches a Design tile only while the Design tab is shown in the tile's mode;
  `call_tool` reaches every Design command from any tab, the tab need not be shown.
- Face and edge ids from `query_topology` expire when the model rebuilds; deleting or moving features
  renumbers features and bodies.
- Design-only edits mark the project modified (`get_app_state.project.dirty`), so `new_project`
  and `open_files` after them need `"discard_changes":true`. [ref]
- Commit appends: without `replace`, the earlier committed object stays on the plate.

## Not possible

Feature patterns (bodies only); reopening a committed sketch for editing (`set_feature` edits its
feature's fields; otherwise delete and redraw); text or SVG in a sketch; sheet metal and drawings.
