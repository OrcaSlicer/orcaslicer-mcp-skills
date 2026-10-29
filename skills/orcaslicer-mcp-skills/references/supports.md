# Supports and adhesion

## Recipes

- Tree from build plate: `set_settings {"scope":"print","values":{"enable_support":true,
  "support_type":"tree(auto)","support_on_build_plate_only":true}}` → slice. Types: `normal(auto)`,
  `tree(auto)`, `normal(manual)`, `tree(manual)` ("manual" = only where enforcers are). [ref]
- Normal snug, two calls, type first: `{"support_type":"normal(auto)"}` → `{"support_style":"snug",
  "support_threshold_angle":40,"support_top_z_distance":0.25,"support_object_xy_distance":0.6,
  "support_interface_top_layers":3}`; check `applied.support_style`. "50° from vertical" = threshold
  40. [ref]
- Per object: `set_settings {"scope":"object","object":0,"values":{"enable_support":true}}`; a brim
  for one object `{"brim_type":"outer_only","brim_width":8}`; clear an override with
  `{"brim_type":null}`; no brim `"no_brim"`. [ref]
- Support blocker or enforcer as a volume (no painting): `add_part {"object":0,"type":"support_blocker",
  "shape":"cube","size":[20,20,5],"position":[x,y,z]}` where `position` is the box centre on the bed;
  size it to cover the space under the overhang, from the bbox in `list_objects`.
  `"support_enforcer"` likewise. [ref]
- Mouse ears: `{"brim_type":"brim_ears","brim_width":8}` (`brim_width` is the ear radius). [ref]
- Brim Ears tool: `select` → `set_view {"camera":"iso"}` → `set_view {"camera":"fit_selection"}` →
  `ui_click {"target":{"gizmo":"Brim Ears"}}` → `ui_click {"target":{"imgui_name":"Set brim type to painted"}}`
  → `ui_input {"target":{"imgui":"Brim ear radius"},"value":5}` → `ui_click {"target":{"imgui":"Auto-generate"}}`
  → `ui_click {"target":{"imgui":"Done"}}`; `list_objects` shows `brim_ears` (count). Ears keep the
  radius they were made with. [ref]
- Raft + draft shield: `{"raft_layers":3,"skirt_loops":1,"draft_shield":"enabled"}` (0 skirt loops =
  no shield). [ref]
- Elephant-foot compensation + brim: `{"elefant_foot_compensation":0.2,"elefant_foot_compensation_layers":2,
  "brim_type":"outer_only","brim_width":3,"brim_object_gap":0,"brim_use_efc_outline":true}` (a 0.4 mm
  total flare = 0.2 per side). [ref]
- Interface-only support filament: `run_action {"id":"orca_command:filament_add:orca"}` →
  `select_preset {"type":"filament","name":"…","filament_slot":2}` → plate type for PETG
  (`references/settings.md`) → auto-answer `{"match":"support material for the support interface","button":"yes"}`
  and `{"match":"recommend the following settings","button":"yes"}` → `set_settings {"scope":"print",
  "values":{"enable_support":true,"support_interface_filament":2,"support_filament":0,
  "support_interface_not_for_body":1}}` → slice; the reply's `filaments[i].support_g` shows the split. [ref]
- Painted blocker under a face: `select` → `ui_click {"target":{"gizmo":"FdmSupports"}}` →
  `ui_click {"target":{"imgui_name":"Fill"}}` → `set_view {"camera":"bottom"}` →
  `ui_mouse {"button":"right","at":{"object":0,"point":[143,128,30],"frame":"world"}}` →
  `ui_click {"target":{"imgui":"Done"}}`. Left button = enforcer. From below the bed hides nothing. [ref]
- Enforcers on real overhangs:
  1. `set_settings {"scope":"print","values":{"enable_support":true,"support_type":"tree(manual)"}}` →
     `ui_click {"target":{"gizmo":"FdmSupports"}}` → `ui_click {"target":{"imgui_name":"Fill"}}`.
  2. `ui_input {"target":{"imgui":"Highlight overhangs"},"value":30}` →
     `ui_input {"target":{"imgui":"On highlighted overhangs only"},"value":true}`.
  3. `set_view {"camera":"front"}` → `set_view {"camera":"fit_objects"}`.
  4. `ui_mouse {"at":{"object":0,"point":[82.5,92.51,58.55],"frame":"world"}}` (the Stanford Bunny's
     chin at the X1C plate centre): with Fill and "On highlighted overhangs only", one click fills the
     connected overhang.
  5. `ui_click {"target":{"imgui":"Done"}}`. [ref]

## Traps

- Manual types start "Highlight overhangs" at 0, which filters nothing until set. Brush max 8 mm.
- Painting reads back as facet counts only (`painted.supports.enforcers/blockers` in `list_objects`);
  prove placement from the G-code's support moves.
- Enforcers with supports off give a warning in `notifications`. Slicing with supports off can open a
  modal "floating cantilever" warning: pre-register `{"match":"cantilever","button":"ok"}`.

## Not possible

Typed painting or region selection (paint, or use `add_part` volumes); brim ear positions (the count
only; positions are in a saved 3MF's `Metadata/brim_ear_points.txt`).
