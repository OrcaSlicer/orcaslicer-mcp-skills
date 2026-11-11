# Gizmos and painting

Gizmos have no typed tools: drive them as a person does (icon, panel fields, clicks on the model).
Typed neighbours: variable layer height is `set_variable_layer_height`, parts and modifiers are
`add_part`, part filament is `set_settings` volume scope.

## Recipes

`select_objects {"objects":[0]}` first; close with Done or a second click on the gizmo.

- Flat base: `click {"target":{"gizmo":"Cut"}}` → `set_widget_value {"target":{"imgui_name":"move_Z"},"value":3}`
  (world Z) → `set_widget_value {"target":{"imgui":"Keep","imgui_name":"lower"},"value":false}` →
  `click {"target":{"imgui":"Perform cut"}}` → `wait {"for":"idle"}`. [ref]
- Connectors: `set_window {"size":[1600,1000]}` → `set_view {"camera":"top"}` →
  `set_view {"camera":"fit_selection"}` → Cut → move_Z → `click {"target":{"imgui":"Add connectors"}}`
  → `click {"target":{"imgui":"Plug"}}` or `"Dowel"` (always: type and size carry over) →
  `set_widget_value {"target":{"imgui_name":"input_Size"},"value":<mm>}` →
  `set_widget_value {"target":{"imgui_name":"input_Depth"},"value":<mm>}` → per pin
  `mouse_gesture {"action":"move","at":{"object":0,"point":[x,y,zcut],"frame":"world"},"check":"point"}` then
  the same call without `action` → `click {"target":{"imgui":"Confirm connectors"}}` → Perform cut →
  wait idle → `arrange {}` (the halves overlap). [ref]
- Engrave text: `set_view {"camera":"front"}` → `set_view {"camera":"fit_selection"}` →
  `click {"target":{"gizmo":"Text"}}` → Text / Height / Depth as in `references/multicolor.md` →
  `click {"target":{"imgui":"Cut"}}` → wait idle → Done → wait idle; the part is `negative`. [ref]
- SVG on top: `set_view {"camera":"top"}` → `set_view {"camera":"fit_selection"}` →
  `run_action {"id":"add_primitive_svg","param":"/abs/star.svg"}` → wait idle →
  `set_widget_value {"target":{"imgui":"Depth"},"value":0.6}` → `set_widget_value {"target":{"imgui":"Size"},"value":50}`
  → wait idle → `click {"target":{"gizmo":"SVG"}}` (close) →
  `set_settings {"scope":"volume","object":0,"volume":1,"values":{"extruder":2}}` for a second colour. [ref]
- Bake engraving: [ref]
  1. `click {"target":{"gizmo":"MeshBoolean"}}` → `click {"target":{"imgui":"Difference","index":0}}`.
  2. `click {"target":{"imgui_name":"select_source_volume"}}` →
     `mouse_gesture {"at":{"object":0,"volume":0,"point":[x,y,z],"frame":"world"}}`.
  3. `click {"target":{"imgui_name":"select_tool_volume"}}` →
     `mouse_gesture {"at":{"object":0,"volume":1,"point":[x,y,z_top],"frame":"world"},"check":"point"}`.
     `"volume":1` names the text part; without it the click selects the source again.
  4. `set_widget_value {"target":{"imgui":"Delete input"},"value":true}` → `click {"target":{"imgui_name":"btn"}}`
     → `wait {"for":"idle"}`.
- Measure: [ref]
  1. Lay the part flat: `transform {"object":0,"rotate":[90,0,0]}` → `set_view {"camera":"top"}`.
  2. `click {"target":{"gizmo":"Measure"}}`.
  3. For each hole, click 0.1 mm outside its rim on the visible face, in mesh coordinates
     (object-centred, before the instance's rotation): `mouse_gesture {"action":"move","at":{"object":0,
     "point":[cx + r + 0.1, …]}}`, then the same call without `action`. The reference used [-22.4,2,0]
     and [27.6,2,0] for Ø5 holes 50 mm apart on a 4 mm plate laid flat. When the bounding box doesn't
     give the hole centres, take them from the mesh with a Python script (`references/python.md`) or
     from the model file.
  4. Read `list_widgets {"query":"Direct distance"}` → close → `transform {"object":0,"scale":<want/measured>}`.
- Simplify (no toolbar icon): `click {"target":{"gizmo":"Simplify"}}` → wait idle →
  `set_widget_value {"target":{"imgui_name":"use_count"},"value":true}` →
  `set_widget_value {"target":{"imgui":"Decimate ratio"},"value":90.5}` (% removed) → wait idle →
  `list_widgets {"query":"triangles"}` (predicted) → `click {"target":{"imgui":"Apply"}}` → wait idle;
  `list_objects` `triangles`. [ref]
- Seam (Cali Cat at the plate centre; mesh coordinates, object-centred): [ref]
  1. `set_view {"camera":"rear"}` → `set_view {"camera":"fit_selection"}`.
  2. `click {"target":{"gizmo":"Seam"}}` → `set_widget_value {"target":{"imgui":"Brush size"},"value":2}`.
  3. `mouse_gesture {"action":"drag","at":{"object":0,"spot":"back"},"waypoints":[{"object":0,"point":[0,14.25,10]}],
     "to":{"object":0,"point":[0,14.25,-16.5]},"check":"point","steps":40}`; y = +bbox.size[1]/2 is the
     back face.
  4. `list_objects` shows `painted.seam.enforcers` > 0.
  5. `set_view {"camera":"front"}` → `set_view {"camera":"fit_selection"}` → Brush size 6.
  6. The same drag with `"button":"right"`, `"at":{"object":0,"spot":"front"}` and a zigzag at
     y = −14.25 (x ±11, z 10 down to −15).
  7. `click {"target":{"imgui":"Done"}}`.
- Fuzzy paint: `click {"target":{"gizmo":"Fuzzy Skin Painting"}}` →
  `click {"target":{"imgui_name":"Fill"}}` → `mouse_gesture {"at":{"object":0,"spot":"front"}}` → Done →
  `set_settings` thickness / point distance (`fuzzy_skin` `none`, the default, applies the paint). [ref]
- Texture: `set_window {"size":[1600,1300]}` → `set_view {"camera":"iso"}` →
  `click {"target":{"gizmo":"Texture Displacement"}}` → wait idle →
  `set_widget_value {"target":{"imgui_name":"Connected area"},"value":true}` → sleep 1 s →
  `click {"target":{"imgui_name":"Layer 1 image"}}` → sleep →
  `click {"target":{"imgui_name":"Knurl"}}` (retry after 2 s) → check `painted.texture[0].name` →
  `mouse_gesture {"at":{"object":0,"spot":"front"}}` → `set_widget_value {"target":{"imgui_name":"Cylindrical"},"value":true}`
  → `set_widget_value {"target":{"imgui":"Tile size"},"value":4}` → `set_widget_value {"target":{"imgui":"Depth"},"value":0.35}`
  → `click {"target":{"imgui_name":"bake"}}` → wait idle ×2 (~40 s) →
  `click {"target":{"imgui_name":"close"}}`. [ref]
- Variable layer height: `set_variable_layer_height {"object":0,"action":"adaptive","quality":0.3}`
  (0 = quality … 1 = speed), then `{"action":"smooth","radius":5}`; `"reset"` removes it.
  `list_objects` shows `layer_height_profile` {points, min_height, max_height}. [ref]

A sleep is `wait {"for":"dialog","timeout_s":2}`.

## Traps

- A point or ImGui item under a panel or notification is refused, naming it: enlarge the window
  (`set_window {"size":[1600,1000]}`) or turn the camera. No tool closes a notification; a
  warning one goes when its cause does (painted support enforcers with support off: turn
  `enable_support` on).
- In a painting gizmo, a stroke whose press misses the object is refused with the pixel where it
  first reaches the object: start it there. [ref] Elsewhere a drag off the model turns or pans the
  camera as the user's mouse settings say (the reply's `camera_moved`); turn it with `set_view`
  `camera` and `orbit` instead.
- `spot` = a ray through the bbox centre along an axis; it misses between letters and into holes.
  Off-surface points need `"check":"point"` and an axis camera.
- Texture: in a fresh app the first layer-image click only shows a tooltip; the library appears after
  thumbnails load, which `wait idle` ignores. Clear Resolution's Auto (`v2auto`) before typing it.
- Measure "Edit to scale": `"submit": false` + click "Scale all" (Return closes the popup). After
  `run_action {"id":"undo"}` the gizmo stays open: check `get_app_state.gizmo` first.
- "Add connectors" does nothing while a half isn't kept (Keep carries over). Connectors aren't
  flagged in `list_objects`; count parts by type.

## Not possible

Typed cut, text, SVG, boolean, simplify, measure or painting; reading where paint is (facet counts
only); the Measure distance as data (read the panel text).

## Driving any gizmo

`tools/mcp/tests/gizmos.json` in an OrcaSlicer checkout drives all 17; copy its steps.

- Select the object (`select_objects`), then `click {"target":{"gizmo":"Seam"}}`. `list_widgets` lists
  the gizmos of the shown 3D view, `[toolbar]` or `[no toolbar icon]` (Simplify, and SVG on a selected
  SVG part). A second click closes the gizmo (`"active": false`); Text on an object with no text
  creates a text part and replies `"created": true`. `get_app_state.gizmo` names the open one.
- `list_widgets` lists each ImGui panel with its items and text, in reading order. A control is
  found by its label (a field with none takes the text left of it on its row, `{"imgui":"Brush size"}`)
  or its name (`{"imgui_name":"Fill"}` for an icon button, `{"imgui_name":"extruder_color_2"}`).
  Each item shows its kind and value: `slider`, `drag`, `input`, `multiline`, `combo`, `checkbox`,
  `radio`, `toggle`, `button`; `[no typing]` marks a slider that takes no typed value.
- `set_widget_value` sets a control by its kind and reads the value back: a combo by option text or 0-based
  index, a slider through the number field beside it, multiline text with `\n`, a checkbox, radio or
  toggle with true/false. A list that only draws what shows (Texture Displacement's library) scrolls
  with `mouse_gesture` `wheel` over it.
- Click the model at a point, not a pixel: `mouse_gesture {"at":{"object":0,"spot":"top"}}`, or
  `"point":[x,y,z]` in the part's mesh coordinates (`"frame":"world"` for scene coordinates; add
  `"volume"` for a part, modifiers and negative parts included). It fails, naming what is in the way,
  when the point is hidden, below the bed or cut away; turn the camera with `set_view` (from below the
  bed hides nothing). `"check":"point"` skips that, for a point off the surface (a cut plane).
  Strokes: `"action":"drag"` with `"waypoints"` and `"to"`. `"modifiers":["shift"]` holds Shift (Measure's
  point picking); `"button":"right"` paints blockers or erases.
- The panel's fields set grabber values (Position, rotation, scale, cut height); grabbers are not
  dragged. File pickers inside a gizmo: `pick_gizmo_file` (`svg_change`, `svg_save_as`, `texture_import`).
- The assemble view: `set_view {"canvas":"assemble"}`, gizmo targets with `"canvas":"assemble"`, and
  `set_view {"canvas":"3d"}` to leave.
- Check the result in `list_objects`: per part `painted` (supports, seam, fuzzy_skin, mmu, texture),
  `triangles`, `mesh_volume`, `text` (text, font), `svg`, `bbox`; per object `brim_ears`. After
  Simplify or a texture preview, `wait {"for":"idle"}` first.
