# Settings and presets

## Recipes

- Find keys: `get_settings {"scope":"print","query":"scarf"}` (key or label); meta has `values` and
  their `labels` for enums. Read another preset without selecting it: `"preset":"0.16mm Optimal @BBL X1C"`.
- What a setting does: `explain_settings {"keys":["seam_slope_type"]}`: the whole tooltip, `scope`,
  `where` (page > group in that tab) and `wiki`; get_settings' meta has no tooltip. Which setting
  controls something: `explain_settings {"query":"ooze travel"}`, a few distinctive words matched in
  keys, labels and tooltips, most words matched first (`"scope":"filament"` narrows it).
- Whole view for tuning: `explain_settings {"scope":"process"}` (no keys/query/where) maps every setting
  as `{"Page > Group": {key: label}}`; `{"scope":"process","where":"Strength > Infill"}` returns that
  group in full. Read groups: a whole page (`"where":"Quality"`) can exceed what the client keeps in the
  conversation. Settings with no row on a page are under `"(no page)"`.
- Edit: `set_settings {"scope":"print"|"filament"|"printer","values":{...}}`; a filament slot's preset
  `"filament_slot":2`. The reply's `applied` and `changed` (the app's own adjustments included). [ref]
- Per-variant values (meta lists `variants` and `active`, e.g. X1C speeds and temperatures "Direct
  Drive Standard, Direct Drive High Flow"): a single value writes the entry in use
  (`nozzle_temperature 215` → `"215,220"`), `{"index":1,"value":300}` another entry, an array all. [ref]
- What changed: `get_settings {"scope":"filament","compare":":saved"}` (unsaved edits), `":parent"`
  (vs the system preset), or a preset name. Revert one key:
  `set_settings {"scope":"process","values":{"wall_loops":null}}` (`"reset_to":"parent"` for the
  parent's value); all: `set_settings {"scope":"process","reset_all":true}`. [ref]
- Select: `select_preset {"type":"process","name":"0.24mm Draft @BBL X1C"}`; filament
  `"filament_slot":1`. With unsaved edits add `"changes":"transfer"` (keep them on the new preset) or
  `"discard"`; without it the prompt opens. [ref]
- Save: `save_preset {"type":"filament","name":"Inland PLA+"}` saves the edited preset with its edits
  and selects it (`"overwrite":true` to replace one; `"inside_project":true` keeps it in the project
  only; printers too, Bambu ones included). To keep only some edits:
  `set_settings {"scope":"filament","reset_all":true,"values":{...}}` → `save_preset`. [ref]
- Rename = `save_preset` under the new name → `delete_preset {"type":"filament","name":"<old>"}`. [ref]
- Delete: `delete_preset {"type":"filament","name":"Jayo PLA test 1"}` (no prompt; a printer must be
  selected first; refused for system presets and presets others inherit from). [ref]
- A preset for one object only: object overrides (`references/modifiers.md`), or save the edited
  preset with `"inside_project":true`.
- Share a filament with other printers: `set_settings {"scope":"filament","values":
  {"compatible_printers":["Voron 2.4 350 0.4 nozzle"]}}` (`[]` = all) → `save_preset` over it. [hand]
- Nozzle on a Bambu printer: the sidebar's nozzle combo, which also reaches variants not installed:
  `ui_input {"target":{"class":"ComboBox","index":0},"value":"0.6"}` → `wait {"for":"idle"}` →
  `get_state.presets` (the filament may fall back to "Default Filament": `select_preset` it again) →
  `select_preset` the process (`list_presets {"type":"process","query":"0.6 nozzle"}`). [ref] Other
  printers: `select_preset {"type":"printer","name":"<model> 0.6 nozzle"}` when installed.
  `nozzle_type` (`hardened_steel`) is a printer setting; `save_preset` the printer to keep it.
- Plate type: `set_settings {"scope":"project","values":{"curr_bed_type":"Textured PEI Plate"}}`
  (`meta.offered` lists the printer's); one plate only: plate scope (`references/output.md`). [ref]
- Print host: `set_settings {"scope":"printer","values":{"host_type":"octoprint","print_host":"192.168.1.50","printhost_apikey":"…"}}`
  → `save_preset {"type":"printer","name":"…"}`. "Octo/Klipper" = `octoprint`; Moonraker `moonraker`.
- Import presets: `run_action {"id":"orca_command:import_configs:orca","param":"/abs/My PLA.json"}`
  (`.json`, `.zip`, `.orca_filament`; `{"path":…,"on_conflict":"overwrite"}` replaces). The message
  names what was imported. [ref]
  - A `.orca_printer` / `.orca_bundle` imports its presets as `_local/<id>/<name>`, and its process
    and filament then don't show as compatible with its printer. Unzip it and import its
    `printer/*.json`, then `process/*.json`, then `filament/*.json`, one call each, and
    `select_preset` the printer, process and filament by name. [ref] The bundle's base printer must
    be installed.
- Export presets: `run_action {"id":"orca_command:export_config:orca","param":"/abs/dir"}` opens the
  dialog (script, `"await":false`): `ui_click {"target":{"label":"Filament presets(.zip)","within":"Export Preset Bundle"}}`
  → `ui_input {"target":{"name":"<preset>","within":"Export Preset Bundle"},"value":true}` →
  `dialog_respond {"button":"ok"}` → `dialog_respond {"dialog":{"title":"Info"},"button":"ok"}`.
- Preferences: read `get_settings {"scope":"preferences","keys":["export_statistics"]}`; change in the
  dialog (script): `run_action {"id":"orca_command:open_preferences:orca","param":"developer:export_statistics"}`
  (`"await":false`) → `ui_input {"target":{"name":"export_statistics","within":"Preferences"},"value":true}`
  → `dialog_respond {"dialog":{"title":"Preferences"},"button":"close"}`. Checkboxes and fields save;
  most drop-down lists don't.
- Create a custom printer (fallback): run `references/scripts/create-printer.json` with
  `run_script {"script":<its JSON>}` after changing the vendor, model, nozzle, bed size, origin,
  height and template it types; checkboxes and paired fields are addressed by index.

## Label → key

Scarf joint placement `seam_slope_type` (`external` = Contour, `all` = Contour and hole), Conditional
scarf `seam_slope_conditional`, Scarf length `seam_slope_min_length`, `seam_slope_steps`,
`scarf_joint_speed`; `wall_sequence` `"inner-outer-inner wall"`; Minimum wall width
`min_bead_width`; Small perimeters `small_perimeter_speed` + `small_perimeter_threshold`; fuzzy skin
Contour `external`, Painted only `none`, noise `"perlin"`; overhang 50–75 % `overhang_3_4_speed`,
75 %+ `overhang_4_4_speed`; overhang fan (filament) `enable_overhang_bridge_fan`,
`overhang_fan_speed`, `overhang_fan_threshold "25%"`; min layer time (filament)
`slow_down_for_layer_cooling` + `slow_down_layer_time`; `elefant_foot_compensation` (sic).

## Traps

- `select_preset` printer silently changes process and filaments: re-read `get_state.presets`.
- `new_project {"discard_changes":true}` keeps modified presets (its reply lists them under
  `modified`); add `"changes":"discard"` to drop them.
- Uninstalled system presets are refused ("does not offer"); `list_presets {"include_hidden":true}`
  shows them with `visible: false`.
- PETG on the X1C's default Cool Plate fails to slice: set Textured PEI first. PETG + a PLA support
  filament fails ("nozzle temperatures are incompatible") unless the preference
  `enable_high_low_temp_mixed_printing` is on.

## Not possible

Installing system printers or filaments, or the setup wizard's choices (its page takes
`webview_eval` after `run_action {"id":"orca_command:help_setup_wizard:orca"}`, but no recipe is
verified); selecting hidden presets by name (on Bambu printers the nozzle combo above reaches other
nozzle variants); writing preferences without the dialog.
