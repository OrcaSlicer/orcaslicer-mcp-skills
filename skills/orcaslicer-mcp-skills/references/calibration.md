# Calibration

## Make a test

`calibrate` builds the Calibration menu's test without its dialog: a new project with the test
model, the presets changed as the test needs, printed with filament slot 1. Select the printer,
process and filament first, then:

`calibrate {"test":"pressure_advance","start":0,"end":0.06,"step":0.002,"discard_changes":true,"changes":"discard"}`
→ `slice {}` → `export {"format":"gcode","path":"/abs/pa.gcode","overwrite":true}`. [ref]

- `discard_changes` replaces an unsaved project; `changes` is required while presets are modified:
  `"discard"` drops the edits first (a clean test), `"keep"` runs with them.
- The reply has `params` (the values used, defaults filled in), `objects`, `names`,
  `presets_changed`, `warnings`, and `result_keys` (scope and keys where the measured value goes).

| test | parameters (omitted = the dialog's default) |
|---|---|
| `temperature` | `start`, `end` °C (rounded down to 5); `test_model` `standard`/`overhang` on belt printers |
| `max_volumetric` | `start`, `end`, `step` mm³/s |
| `pressure_advance` | `method` `tower` (default) / `line` / `pattern`; `start`, `end`, `step`; line: `print_numbers`; pattern: `accelerations` [..], `speeds` [..] (one test per pair) |
| `flow_ratio` | `method` `yolo` (default) / `yolo_perfectionist` / `pass1` / `pass2`; `top_surface_pattern` `archimedeanchords` / `monotonic` |
| `retraction` | `start`, `end`, `step` mm |
| `vfa` | `start`, `end`, `step` mm/s; `auto_adjust`, `auto_scale` |
| `cornering` | `start`, `end` (junction deviation mm, or jerk mm/s); `test_model` `scv_v2` |
| `input_shaping_freq` | `start`, `end` Hz (X), `start_y`, `end_y`; `damping`; `shaper_type` as the firmware names it (`"MZV"` on Klipper, case-sensitive); `test_model` `ringing_tower`/`fast_tower` |
| `input_shaping_damp` | `freq_x`, `freq_y` Hz; `start`, `end` damping; `shaper_type`; `test_model` |

Examples: `{"test":"flow_ratio","method":"pass1","top_surface_pattern":"monotonic",…}`;
`{"test":"pressure_advance","method":"pattern","accelerations":[1000,2000],"speeds":[50,100],…}`
(several objects on one plate: `slice {}`, then
`export {"format":"sliced_3mf","plate":"all","path":"/abs/pa.gcode.3mf","overwrite":true}`);
`{"test":"input_shaping_freq","start":15,"end":110,"shaper_type":"MZV",…}`. [ref]

## Apply and save a measured value

`new_project {"discard_changes":true,"changes":"discard"}` (drops the test's own settings) →
`set_settings` the value in the `result_keys` scope → `save_preset` the preset to keep it. [ref]

- Pressure advance: `set_settings {"scope":"filament","values":{"enable_pressure_advance":true,"pressure_advance":0.034}}`
  → `save_preset {"type":"filament","name":"…"}`.
- Max volumetric speed: `filament_max_volumetric_speed`; retraction: `filament_retraction_length`
  (filament override) or `retraction_length` (printer); flow: `filament_flow_ratio`.
- VFA ripples: `set_settings {"scope":"printer","values":{"resonance_avoidance":true,
  "min_resonance_avoidance_speed":110,"max_resonance_avoidance_speed":130}}` → `save_preset` printer.
- Cornering: `machine_max_junction_deviation` (Marlin 2 with JD) or `machine_max_jerk_x`/`_y`.

Reading results:
- PA tower: start + step × height in mm (17 mm, 0.002/mm → 0.034).
- Max flow: start + step × height (19 mm, 5 + 0.5 × 19 = 14.5; −10 % margin → 13.05).
- Flow pass 1 "+5" → ratio × 1.05; pass 2 adds its offset to the pass-1 ratio.
- Retraction tower: export the G-code and read
  `; Calib_Retraction_tower: Z_HEIGHT: 12, length:3.2` at the height (don't compute it). [ref]
- Tolerance test: `run_action {"id":"orca_command:add_handy_orca_tolerance_test:orca"}` with
  `{"xy_hole_compensation":0,"xy_contour_compensation":0}`.

## Traps

- Each test starts a new project and answers its own prompts: a project with unsaved changes needs
  `discard_changes`, and modified presets need `changes`, or it is refused.
- `machine_max_*` values hold a normal and a silent entry per variant (`meta.variants`); a single
  value writes only the normal entry in use (`12` → `"12,9,9,9"`). Give the silent one by `index`
  too when the printer runs in silent mode.
- Input shaping on a Marlin-flavoured Bambu printer generates but does not slice.

## Not possible

The calibrated value per layer as data (read the exported G-code).
