# Modifiers, parts and per-object settings

## Recipes

- Per object: `set_settings {"scope":"object","object":1,"values":{"wall_loops":5,
  "sparse_infill_density":"40%","sparse_infill_pattern":"gyroid"}}`; read back
  `get_settings {"scope":"object","object":1}` (overrides only, each with `effective` and `source`);
  remove one with `{"key":null}`. Only keys the object list's Add Settings offers are taken
  (`print_sequence` and other process-wide keys are refused). [ref]
- Add a part, negative part, modifier, support blocker or enforcer:
  `add_part {"object":0,"type":"negative","shape":"cylinder","size":[5,20],"position":[128,128,10]}`.
  - `type`: `part`, `negative`, `modifier`, `support_blocker`, `support_enforcer`, `precise_seam`.
  - `shape` + `size`: cube `[x,y,z]`; cylinder, cone, disc `[diameter,height]`; sphere `[d]`;
    torus `[inner,tube]`. Or `"path":"/abs/zone.stl"` for a model file at its own coordinates.
  - `position` is the part's centre on the bed: a through-hole in a 20 mm cube standing on the bed
    is z = 10 with height 20; a 3.2 mm magnet pocket in the bottom is z = 1.6.
  - Add parts before instances: an object with several instances is refused. The reply's `volume`
    is the new part's index (parts are ordered part, negative, modifier, …). [ref]
- Settings inside a modifier: `set_settings {"scope":"volume","object":0,"volume":2,"values":
  {"sparse_infill_density":"100%","wall_loops":4}}` (region keys only). [ref]
- Height ranges ("below 15 mm …, from 15 mm …"):
  `set_layer_ranges {"object":0,"layer_config_ranges":[{"min_z":0,"max_z":15,"values":{"layer_height":0.28}},
  {"min_z":15,"max_z":48,"values":{"layer_height":0.12}}]}`. It replaces all ranges (`[]` removes
  them); ranges may touch, not overlap; `values` take `layer_height`, `extruder` and per-region keys
  (`sparse_infill_density`, `wall_loops`). Read back in `list_objects` `layer_config_ranges`; slice
  and check `layers`. [ref]
- Change a part's type: `select {"volumes":[[0,1]]}` →
  `run_action {"id":"orca_command:obj_type_modifier:orca"}` (`_part`, `_negative`,
  `_support_blocker`, `_support_enforcer`, `_precise_seam`); part indices change, re-read
  `list_objects`. [ref]
- Imported modifier zones: `add_part {"object":0,"type":"modifier","path":"/abs/zones.stl"}` (lands at
  the file's coordinates relative to the object's first part), or load both files and
  `select {"objects":[0,1]}` → `run_action {"id":"orca_command:obj_assemble:orca"}` → change the
  zone part's type as above. [ref]
- Split into parts: `select {"objects":[0]}` → `run_action {"id":"orca_command:obj_split_parts:orca"}`;
  into objects: `obj_split_objects`. [ref]
- Rename an object or part: `select {"objects":[0]}` (or `{"volumes":[[0,1]]}`) →
  `run_action {"id":"orca_command:obj_rename:orca","param":"tall"}`. [ref]
- Move or resize a part: `transform {"object":0,"volume":1,"translate":[100,100,15],"mode":"absolute"}`
  (the part's centre on the bed), `"size":[null,null,3]`, `"scale":2`; rotation relative only. [ref]
- Copy settings to another object: read the source's overrides (`get_settings` object scope) and
  write them with `set_settings` to the other.

## Traps

- `list_objects` part `bbox`, `position` and `scaled_volume` are on the bed (instance 0); an object's
  `bbox` counts model parts only, not modifiers or negative parts.
- A single-part object has no part row: an `extruder` set on its `"volume":0` is the object's (the
  reply notes it); use the object scope.
- `ui_mouse` points on modifiers and negative parts are targets like any part.

## Not possible

Per-range settings outside `layer_config_ranges`' key set (support settings are refused there).
