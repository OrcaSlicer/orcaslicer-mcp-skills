# Design (CAD) tab

Every call is `cad_call {"method":"<name>","params":{...}}`. The CAD feature must be on
(Preferences > Features, then a restart). Look up params with
`cad_call {"method":"describe_tools","params":{"method":["extrude","hole"]}}` or
`{"query":"sketch"}`, never the whole list.

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
- Variables: `set_variable {"name":"w","expr":"30"}` → `set_feature_expr {"feature":0,"field":"width","expr":"w"}`
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
  `mass_properties {"body":0}` (`volume`), `measure {"a":{"face":14,"axis":true},"b":{"face":10}}`
  (`min_distance`), `query_topology` radii.
- Commit to the plate: `commit {}` (one object; `"mode":"bodies"` one per body;
  `"replace":[i]` deletes those Prepare objects in the same undo step, e.g. the stale object of an
  earlier commit) → `list_objects`. `export_step {"path":"/abs/x.step","overwrite":true}`. [ref]
- History: `undo` / `redo` (the Design history; `run_action undo` is Prepare's); `feature_delete`,
  `feature_move {"feature":2,"delta":-1}`, `feature_set_enabled`, `feature_rename`;
  `body_set {"body":1,"name":"Lid","color":"#3080ff","visible":false}` (a hidden body is not
  committed), `body_delete`. `new_design {"discard_changes":true}` starts over.
- A gear or other generated outline: compute the points and `extrude {"profile":[…]}`.

## Traps

- `sketch_add` ignores unknown keys: a line given `start`/`end` becomes a zero-length line, and the
  extrude then fails ("BRep_API: command not done"). Use `p0`/`p1`.
- The methods in `describe_tools`' `legacy_ok_false` (`fillet`, `chamfer`, `boolean`, `pattern`,
  `shell`, `mirror`, `set_variable`, `set_feature_expr`, `import_*`…) answer `{"ok":false,"error":…}`
  instead of failing: check `ok`.
- Face and edge ids from `query_topology` expire when the model rebuilds; deleting or moving features
  renumbers features and bodies.
- Design-only edits leave `get_state.project.dirty` false, yet opening another project with
  `open_files {"discard_changes":true}` can still ask "save before continuing": auto-answer
  `{"match":"save before continuing","button":"no"}` first.
- Commit appends: without `replace`, the earlier committed object stays on the plate.

## Not possible

Feature patterns (bodies only); reopening a committed sketch for editing (`set_feature` edits its
feature's fields; otherwise delete and redraw); text or SVG in a sketch; sheet metal and drawings.
