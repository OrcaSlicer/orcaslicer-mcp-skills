# Gizmos and painting

Gizmos have no typed tools: drive them as a person does (icon, panel fields, clicks on the model).
Typed neighbours: variable layer height is `set_layer_height_profile`, parts and modifiers are
`add_part`, part filament is `set_settings` volume scope.

## Recipes

`select {"objects":[0]}` first; close with Done or a second click on the gizmo.

- Flat base: `ui_click {"target":{"gizmo":"Cut"}}` → `ui_input {"target":{"imgui_name":"move_Z"},"value":3}`
  (world Z) → `ui_input {"target":{"imgui":"Keep","imgui_name":"lower"},"value":false}` →
  `ui_click {"target":{"imgui":"Perform cut"}}` → `wait {"for":"idle"}`. [ref]
- Connectors: `set_view {"window_size":[1600,1000]}` → `set_view {"camera":"top"}` →
  `set_view {"camera":"fit_selection"}` → Cut → move_Z → `ui_click {"target":{"imgui":"Add connectors"}}`
  → `ui_click {"target":{"imgui":"Plug"}}` or `"Dowel"` (always: type and size carry over) →
  `ui_input {"target":{"imgui_name":"input_Size"},"value":<mm>}` →
  `ui_input {"target":{"imgui_name":"input_Depth"},"value":<mm>}` → per pin
  `ui_mouse {"action":"move","at":{"object":0,"point":[x,y,zcut],"frame":"world"},"check":"point"}` then
  the same call without `action` → `ui_click {"target":{"imgui":"Confirm connectors"}}` → Perform cut →
  wait idle → `arrange {}` (the halves overlap). [ref]
- Engrave text: `set_view {"camera":"front"}` → `set_view {"camera":"fit_selection"}` →
  `ui_click {"target":{"gizmo":"Text"}}` → Text / Height / Depth as in `references/multicolor.md` →
  `ui_click {"target":{"imgui":"Cut"}}` → wait idle → Done → wait idle; the part is `negative`. [ref]
- SVG on top: `set_view {"camera":"top"}` → `set_view {"camera":"fit_selection"}` →
  `run_action {"id":"orca_command:add_primitive_svg:orca","param":"/abs/star.svg"}` → wait idle →
  `ui_input {"target":{"imgui":"Depth"},"value":0.6}` → `ui_input {"target":{"imgui":"Size"},"value":50}`
  → wait idle → `ui_click {"target":{"gizmo":"SVG"}}` (close) →
  `set_settings {"scope":"volume","object":0,"volume":1,"values":{"extruder":2}}` for a second colour. [ref]
- Bake engraving: [ref]
  1. `ui_click {"target":{"gizmo":"MeshBoolean"}}` → `ui_click {"target":{"imgui":"Difference","index":0}}`.
  2. `ui_click {"target":{"imgui_name":"select_source_volume"}}` →
     `ui_mouse {"at":{"object":0,"volume":0,"point":[x,y,z],"frame":"world"}}`.
  3. `ui_click {"target":{"imgui_name":"select_tool_volume"}}` →
     `ui_mouse {"at":{"object":0,"volume":1,"point":[x,y,z_top],"frame":"world"},"check":"point"}`.
     `"volume":1` names the text part; without it the click selects the source again.
  4. `ui_input {"target":{"imgui":"Delete input"},"value":true}` → `ui_click {"target":{"imgui_name":"btn"}}`
     → `wait {"for":"idle"}`.
- Measure: [ref]
  1. Lay the part flat: `transform {"object":0,"rotate":[90,0,0]}` → `set_view {"camera":"top"}`.
  2. `ui_click {"target":{"gizmo":"Measure"}}`.
  3. For each hole, click 0.1 mm outside its rim on the visible face, in mesh coordinates
     (object-centred, before the instance's rotation): `ui_mouse {"action":"move","at":{"object":0,
     "point":[cx + r + 0.1, …]}}`, then the same call without `action`. The reference used [-22.4,2,0]
     and [27.6,2,0] for Ø5 holes 50 mm apart on a 4 mm plate laid flat. When the bounding box doesn't
     give the hole centres, take them from the mesh file with a short script.
  4. Read `ui_snapshot {"query":"Direct distance"}` → close → `transform {"object":0,"scale":<want/measured>}`.
- Simplify (no toolbar icon): `ui_click {"target":{"gizmo":"Simplify"}}` → wait idle →
  `ui_input {"target":{"imgui_name":"use_count"},"value":true}` →
  `ui_input {"target":{"imgui":"Decimate ratio"},"value":90.5}` (% removed) → wait idle →
  `ui_snapshot {"query":"triangles"}` (predicted) → `ui_click {"target":{"imgui":"Apply"}}` → wait idle;
  `list_objects` `triangles`. [ref]
- Seam (Cali Cat at the plate centre; mesh coordinates, object-centred): [ref]
  1. `set_view {"camera":"rear"}` → `set_view {"camera":"fit_selection"}`.
  2. `ui_click {"target":{"gizmo":"Seam"}}` → `ui_input {"target":{"imgui":"Brush size"},"value":2}`.
  3. `ui_mouse {"action":"drag","at":{"object":0,"spot":"back"},"path":[{"object":0,"point":[0,14.25,10]}],
     "to":{"object":0,"point":[0,14.25,-16.5]},"check":"point","steps":40}`; y = +bbox.size[1]/2 is the
     back face.
  4. `list_objects` shows `painted.seam.enforcers` > 0.
  5. `set_view {"camera":"front"}` → `set_view {"camera":"fit_selection"}` → Brush size 6.
  6. The same drag with `"button":"right"`, `"at":{"object":0,"spot":"front"}` and a zigzag at
     y = −14.25 (x ±11, z 10 down to −15).
  7. `ui_click {"target":{"imgui":"Done"}}`.
- Fuzzy paint: `ui_click {"target":{"gizmo":"Fuzzy Skin Painting"}}` →
  `ui_click {"target":{"imgui_name":"Fill"}}` → `ui_mouse {"at":{"object":0,"spot":"front"}}` → Done →
  `set_settings` thickness / point distance (`fuzzy_skin` `none`, the default, applies the paint). [ref]
- Texture: `set_view {"window_size":[1600,1300]}` → `set_view {"camera":"iso"}` →
  `ui_click {"target":{"gizmo":"Texture Displacement"}}` → wait idle →
  `ui_input {"target":{"imgui_name":"Connected area"},"value":true}` → sleep 1 s →
  `ui_click {"target":{"imgui_name":"Layer 1 image"}}` → sleep →
  `ui_click {"target":{"imgui_name":"Knurl"}}` (retry after 2 s) → check `painted.texture[0].name` →
  `ui_mouse {"at":{"object":0,"spot":"front"}}` → `ui_input {"target":{"imgui_name":"Cylindrical"},"value":true}`
  → `ui_input {"target":{"imgui":"Tile size"},"value":4}` → `ui_input {"target":{"imgui":"Depth"},"value":0.35}`
  → `ui_click {"target":{"imgui_name":"bake"}}` → wait idle ×2 (~40 s) →
  `ui_click {"target":{"imgui_name":"close"}}`. [ref]
- Variable layer height: `set_layer_height_profile {"object":0,"action":"adaptive","quality":0.3}`
  (0 = quality … 1 = speed), then `{"action":"smooth","radius":5}`; `"reset"` removes it.
  `list_objects` shows `layer_height_profile` {points, min_height, max_height}. [ref]

A sleep is `wait {"for":"dialog","timeout_s":2}`.

## Traps

- A point or ImGui item under a panel or notification is refused, naming it: enlarge the window
  (`set_view {"window_size":[1600,1000]}`) or turn the camera. No tool closes a notification; a
  warning one goes when its cause does (painted support enforcers with support off: turn
  `enable_support` on).
- A drag starting off the model turns (left) or pans (right) the camera, with no error.
- `spot` = a ray through the bbox centre along an axis; it misses between letters and into holes.
  Off-surface points need `"check":"point"` and an axis camera.
- Texture: in a fresh app the first layer-image click only shows a tooltip; the library appears after
  thumbnails load, which `wait idle` ignores. Clear Resolution's Auto (`v2auto`) before typing it.
- Measure "Edit to scale": `"submit": false` + click "Scale all" (Return closes the popup). After
  `run_action {"id":"orca_command:undo:orca"}` the gizmo stays open: check `get_state.gizmo` first.
- "Add connectors" does nothing while a half isn't kept (Keep carries over). Connectors aren't
  flagged in `list_objects`; count parts by type.

## Not possible

Typed cut, text, SVG, boolean, simplify, measure or painting; reading where paint is (facet counts
only); the Measure distance as data (read the panel text).

## Driving any gizmo

`tools/mcp/tests/gizmos.json` in an OrcaSlicer checkout drives all 17; copy its steps.

- Select the object (`select`), then `ui_click {"target":{"gizmo":"Seam"}}`. `ui_snapshot` lists
  the gizmos of the shown 3D view, `[toolbar]` or `[no toolbar icon]` (Simplify, and SVG on a selected
  SVG part). A second click closes the gizmo (`"active": false`); Text on an object with no text
  creates a text part and replies `"created": true`. `get_state.gizmo` names the open one.
- `ui_snapshot` lists each ImGui panel with its items and text, in reading order. A control is
  found by its label (a field with none takes the text left of it on its row, `{"imgui":"Brush size"}`)
  or its name (`{"imgui_name":"Fill"}` for an icon button, `{"imgui_name":"extruder_color_2"}`).
  Each item shows its kind and value: `slider`, `drag`, `input`, `multiline`, `combo`, `checkbox`,
  `radio`, `toggle`, `button`; `[no typing]` marks a slider that takes no typed value.
- `ui_input` sets a control by its kind and reads the value back: a combo by option text or 0-based
  index, a slider through the number field beside it, multiline text with `\n`, a checkbox, radio or
  toggle with true/false. A list that only draws what shows (Texture Displacement's library) scrolls
  with `ui_mouse` `wheel` over it.
- Click the model at a point, not a pixel: `ui_mouse {"at":{"object":0,"spot":"top"}}`, or
  `"point":[x,y,z]` in the part's mesh coordinates (`"frame":"world"` for scene coordinates; add
  `"volume"` for a part, modifiers and negative parts included). It fails, naming what is in the way,
  when the point is hidden, below the bed or cut away; turn the camera with `set_view` (from below the
  bed hides nothing). `"check":"point"` skips that, for a point off the surface (a cut plane).
  Strokes: `"action":"drag"` with `"path"` and `"to"`. `"modifiers":["shift"]` holds Shift (Measure's
  point picking); `"button":"right"` paints blockers or erases.
- The panel's fields set grabber values (Position, rotation, scale, cut height); grabbers are not
  dragged. File pickers inside a gizmo: `gizmo_file` (`svg_change`, `svg_save_as`, `texture_import`).
- The assemble view: `set_view {"view":"assemble"}`, gizmo targets with `"canvas":"assemble"`, and
  `set_view {"view":"3d"}` to leave.
- Check the result in `list_objects`: per part `painted` (supports, seam, fuzzy_skin, mmu, texture),
  `triangles`, `mesh_volume`, `text` (text, font), `svg`, `bbox`; per object `brim_ears`. After
  Simplify or a texture preview, `wait {"for":"idle"}` first.
