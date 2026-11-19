# Gizmos and painting

Seven commands open a gizmo on the object they name: `object_cut`, `object_lay_on_face`, `object_set_brim_ears`,
`object_paint {"type":"supports"|"seam"|"color"|"fuzzy_skin"}`, `object_measure`, `object_simplify` and
`part_boolean`. Given only the target (`{"object":0}`, also `instance` or `part`), each shows the Prepare view,
selects it and opens the gizmo; then drive the panel as a person does (fields, clicks on the model), and close it
with `object_select {"none":true}`. The same call again leaves it open. A gizmo that cannot open says why
(`needs_two_parts`, `needs_two_filaments`, `needs_one_part`...) and leaves the selection as it was.

Three of them also act from arguments, with no gizmo, as one undo step, closing an open gizmo first: a cut by a
plane, lay on face by a direction, brim ears by bed position. Typed neighbours: variable layer height is
`object_set_variable_layer_height`, parts and modifiers are `part_add`, part filament is `settings_set` with the
target `part`.

## Recipes

- Flat base by arguments: `object_cut {"object":0,"z":3,"keep":"upper"}` keeps the part above 3 mm, placed on its
  cut face; `{"z":10}` keeps both as two objects (`pieces` in the reply, with `open_edges` that `object_fix`
  repairs); `"plane":{"point":[x,y,z],"normal":[1,0,1]}` cuts on a tilted plane; `"to_parts":true` keeps both as
  parts of one object; `"upper"`/`"lower"`: `as_cut`, `place_on_cut`, `flip`. Connectors, the dovetail and a cut
  along a contour need the gizmo (`object_cut {"object":0}` opens it). [ref]
- Lay on a face by a direction: `object_lay_on_face {"object":0,"normal":[0,0,-1]}` (the object's own frame;
  `"frame":"world"` for the 3D view's); every instance turns with it. Picking one of the faces the gizmo finds needs
  the gizmo. [ref]
- Brim ears at bed positions: `object_set_brim_ears {"object":0,"points":[{"position":[90,90],"radius":3},{"position":[110,90]}]}`
  (radius 0.1 to 100 mm, the gizmo's default when left out); `"clear":true` removes them. They print only where the
  object's brim type is painted: `settings_set {"target":"object","object":0,"values":{"brim_type":"painted"}}`.
  Auto-generate needs the gizmo. [ref]

Through the gizmo's own panel: open it with its command (or `object_select {"objects":[0]}` and a click on its
icon); close with Done or `object_select {"none":true}`.

- Flat base: `widget_click {"target":{"gizmo":"Cut"}}` → `widget_set_value {"target":{"imgui_name":"move_Z"},"value":3}`
  (world Z) → `widget_set_value {"target":{"imgui":"Keep","imgui_name":"lower"},"value":false}` →
  `widget_click {"target":{"imgui":"Perform cut"}}` → `app_wait {"for":"idle"}`. [ref]
- Connectors: `window_set {"size":[1600,1000]}` → `view_set {"camera":"top"}` →
  `view_set {"camera":"fit_selection"}` → Cut → move_Z → `widget_click {"target":{"imgui":"Add connectors"}}`
  → `widget_click {"target":{"imgui":"Plug"}}` or `"Dowel"` (always: type and size carry over) →
  `widget_set_value {"target":{"imgui_name":"input_Size"},"value":<mm>}` →
  `widget_set_value {"target":{"imgui_name":"input_Depth"},"value":<mm>}` → per pin
  `mouse_gesture {"action":"move","at":{"object":0,"point":[x,y,zcut],"frame":"world"},"check":"point"}` then
  the same call without `action` → `widget_click {"target":{"imgui":"Confirm connectors"}}` → Perform cut →
  wait idle → `object_arrange {}` (the halves overlap). [ref]
- Engrave text: `view_set {"camera":"front"}` → `view_set {"camera":"fit_selection"}` →
  `widget_click {"target":{"gizmo":"Text"}}` → Text / Height / Depth as in `references/multicolor.md` →
  `widget_click {"target":{"imgui":"Cut"}}` → wait idle → Done → wait idle; the part is `negative`. [ref]
- SVG on top: `view_set {"camera":"top"}` → `view_set {"camera":"fit_selection"}` →
  `object_add_svg {"path":"/abs/star.svg"}` → wait idle →
  `widget_set_value {"target":{"imgui":"Depth"},"value":0.6}` → `widget_set_value {"target":{"imgui":"Size"},"value":50}`
  → wait idle → `widget_click {"target":{"gizmo":"SVG"}}` (close) →
  `settings_set {"target":"part","object":0,"part":1,"values":{"extruder":2}}` for a second colour. [ref]
- Bake engraving: [ref]
  1. `widget_click {"target":{"gizmo":"MeshBoolean"}}` → `widget_click {"target":{"imgui":"Difference","index":0}}`.
  2. `widget_click {"target":{"imgui_name":"select_source_volume"}}` →
     `mouse_gesture {"at":{"object":0,"part":0,"point":[x,y,z],"frame":"world"}}`.
  3. `widget_click {"target":{"imgui_name":"select_tool_volume"}}` →
     `mouse_gesture {"at":{"object":0,"part":1,"point":[x,y,z_top],"frame":"world"},"check":"point"}`.
     `"part":1` names the text part; without it the click selects the source again.
  4. `widget_set_value {"target":{"imgui":"Delete input"},"value":true}` → `widget_click {"target":{"imgui_name":"btn"}}`
     → `app_wait {"for":"idle"}`.
- Measure: [ref]
  1. Lay the part flat: `object_transform {"object":0,"rotate":[90,0,0]}` → `view_set {"camera":"top"}`.
  2. `widget_click {"target":{"gizmo":"Measure"}}`.
  3. For each hole, click 0.1 mm outside its rim on the visible face, in mesh coordinates
     (object-centred, before the instance's rotation): `mouse_gesture {"action":"move","at":{"object":0,
     "point":[cx + r + 0.1, …]}}`, then the same call without `action`. The reference used [-22.4,2,0]
     and [27.6,2,0] for Ø5 holes 50 mm apart on a 4 mm plate laid flat. When the bounding box doesn't
     give the hole centres, take them from the mesh with a Python script (`references/python.md`) or
     from the model file.
  4. Read `widget_list {"query":"Direct distance"}` → close → `object_transform {"object":0,"scale":<want/measured>}`.
- Simplify (no toolbar icon): `widget_click {"target":{"gizmo":"Simplify"}}` → wait idle →
  `widget_set_value {"target":{"imgui_name":"use_count"},"value":true}` →
  `widget_set_value {"target":{"imgui":"Decimate ratio"},"value":90.5}` (% removed) → wait idle →
  `widget_list {"query":"triangles"}` (predicted) → `widget_click {"target":{"imgui":"Apply"}}` → wait idle;
  `object_list` `triangles`. [ref]
- Seam (Cali Cat at the plate centre; mesh coordinates, object-centred): [ref]
  1. `view_set {"camera":"rear"}` → `view_set {"camera":"fit_selection"}`.
  2. `widget_click {"target":{"gizmo":"Seam"}}` → `widget_set_value {"target":{"imgui":"Brush size"},"value":2}`.
  3. `mouse_gesture {"action":"drag","at":{"object":0,"spot":"back"},"waypoints":[{"object":0,"point":[0,14.25,10]}],
     "to":{"object":0,"point":[0,14.25,-16.5]},"check":"point","steps":40}`; y = +bbox.size[1]/2 is the
     back face.
  4. `object_list` shows `painted.seam.enforcers` > 0.
  5. `view_set {"camera":"front"}` → `view_set {"camera":"fit_selection"}` → Brush size 6.
  6. The same drag with `"button":"right"`, `"at":{"object":0,"spot":"front"}` and a zigzag at
     y = −14.25 (x ±11, z 10 down to −15).
  7. `widget_click {"target":{"imgui":"Done"}}`.
- Fuzzy paint: `widget_click {"target":{"gizmo":"Fuzzy Skin Painting"}}` →
  `widget_click {"target":{"imgui_name":"Fill"}}` → `mouse_gesture {"at":{"object":0,"spot":"front"}}` → Done →
  `settings_set` thickness / point distance (`fuzzy_skin` `none`, the default, applies the paint). [ref]
- Texture: `window_set {"size":[1600,1300]}` → `view_set {"camera":"iso"}` →
  `widget_click {"target":{"gizmo":"Texture Displacement"}}` → wait idle →
  `widget_set_value {"target":{"imgui_name":"Connected area"},"value":true}` → sleep 1 s →
  `widget_click {"target":{"imgui_name":"Layer 1 image"}}` → sleep →
  `widget_click {"target":{"imgui_name":"Knurl"}}` (retry after 2 s) → check `painted.texture[0].name` →
  `mouse_gesture {"at":{"object":0,"spot":"front"}}` → `widget_set_value {"target":{"imgui_name":"Cylindrical"},"value":true}`
  → `widget_set_value {"target":{"imgui":"Tile size"},"value":4}` → `widget_set_value {"target":{"imgui":"Depth"},"value":0.35}`
  → `widget_click {"target":{"imgui_name":"bake"}}` → wait idle ×2 (~40 s) →
  `widget_click {"target":{"imgui_name":"close"}}`. [ref]
- Variable layer height: `object_set_variable_layer_height {"object":0,"action":"adaptive","quality":0.3}`
  (0 = quality … 1 = speed), then `{"action":"smooth","radius":5}`; `"reset"` removes it.
  `object_list` shows `layer_height_profile` {points, min_height, max_height}. [ref]

A sleep is `app_wait {"for":"dialog","timeout_s":2}`.

## Traps

- A point or ImGui item under a panel or notification is refused, naming it: enlarge the window
  (`window_set {"size":[1600,1000]}`) or turn the camera. No tool closes a notification; a
  warning one goes when its cause does (painted support enforcers with support off: turn
  `enable_support` on).
- In a painting gizmo, a stroke whose press misses the object is refused with the pixel where it
  first reaches the object: start it there. [ref] Elsewhere a drag off the model turns or pans the
  camera as the user's mouse settings say (the reply's `camera_moved`); turn it with `view_set`
  `camera` and `orbit` instead.
- `spot` = a ray through the bbox centre along an axis; it misses between letters and into holes.
  Off-surface points need `"check":"point"` and an axis camera.
- Texture: in a fresh app the first layer-image click only shows a tooltip; the library appears after
  thumbnails load, which `app_wait idle` ignores. Clear Resolution's Auto (`v2auto`) before typing it.
- Measure "Edit to scale": `"submit": false` + click "Scale all" (Return closes the popup). After
  `undo {}` the gizmo stays open: check `app_state.gizmo` first.
- "Add connectors" does nothing while a half isn't kept (Keep carries over). Connectors aren't
  flagged in `object_list`; count parts by type.

## Not possible

Typed cut, text, SVG, boolean, simplify, measure or painting; reading where paint is (facet counts
only); the Measure distance as data (read the panel text).

## Driving any gizmo

`tools/mcp/tests/gizmos.json` in an OrcaSlicer checkout drives all 17; copy its steps.

- Select the object (`object_select`), then `widget_click {"target":{"gizmo":"Seam"}}`. `widget_list` lists
  the gizmos of the shown 3D view, `[toolbar]` or `[no toolbar icon]` (Simplify, and SVG on a selected
  SVG part). A second click closes the gizmo (`"active": false`); Text on an object with no text
  creates a text part and replies `"created": true`. `app_state.gizmo` names the open one.
- `widget_list` lists each ImGui panel with its items and text, in reading order. A control is
  found by its label (a field with none takes the text left of it on its row, `{"imgui":"Brush size"}`)
  or its name (`{"imgui_name":"Fill"}` for an icon button, `{"imgui_name":"extruder_color_2"}`).
  Each item shows its kind and value: `slider`, `drag`, `input`, `multiline`, `combo`, `checkbox`,
  `radio`, `toggle`, `button`; `[no typing]` marks a slider that takes no typed value.
- `widget_set_value` sets a control by its kind and reads the value back: a combo by option text or 0-based
  index, a slider through the number field beside it, multiline text with `\n`, a checkbox, radio or
  toggle with true/false. A list that only draws what shows (Texture Displacement's library) scrolls
  with `mouse_gesture` `wheel` over it.
- Click the model at a point, not a pixel: `mouse_gesture {"at":{"object":0,"spot":"top"}}`, or
  `"point":[x,y,z]` in the part's mesh coordinates (`"frame":"world"` for scene coordinates; add
  `"part"` for a part, modifiers and negative parts included). It fails, naming what is in the way,
  when the point is hidden, below the bed or cut away; turn the camera with `view_set` (from below the
  bed hides nothing). `"check":"point"` skips that, for a point off the surface (a cut plane).
  Strokes: `"action":"drag"` with `"waypoints"` and `"to"`. `"modifiers":["shift"]` holds Shift (Measure's
  point picking); `"button":"right"` paints blockers or erases.
- The panel's fields set grabber values (Position, rotation, scale, cut height); grabbers are not
  dragged. File pickers inside a gizmo: `gizmo_pick_file` (`svg_change`, `svg_save_as`, `texture_import`).
- The assemble view: `view_set {"canvas":"assemble"}`, gizmo targets with `"canvas":"assemble"`, and
  `view_set {"canvas":"3d"}` to leave.
- Check the result in `object_list`: per part `painted` (supports, seam, fuzzy_skin, mmu, texture),
  `triangles`, `mesh_volume`, `text` (text, font), `svg`, `bbox`; per object `brim_ears`. After
  Simplify or a texture preview, `app_wait {"for":"idle"}` first.
