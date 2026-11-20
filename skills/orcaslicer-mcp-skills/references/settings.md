# Settings and presets

## Recipes

- What a setting does: `settings_get {"keys":["seam_slope_type"]}` with no type finds it among the
  process, filament and printer settings: the item's `type` (the one `settings_set` takes), `value`,
  and `meta` with the label, `description` (the whole tooltip), `where` (page > group in that tab), and
  for enums `values` with their `labels`. [ref]
- Which setting controls something: `settings_get {"query":"ooze travel"}`, a few distinctive words
  matched in keys, labels and descriptions, best first (20 unless `limit`; `"type":"filament"` narrows
  it). [ref] Read another preset without selecting it: `"type":"process","preset":"0.16mm Optimal @BBL X1C"`.
- Whole view for tuning: `settings_list {"type":"process","where":"?"}` names no group, so its error
  lists every page > group of the process in `groups` (`"type":"filament"` or `"printer"` for theirs);
  `{"type":"process","where":"Strength > Infill"}` returns that group in full. Read groups: a whole page
  (`"where":"Quality"`) can exceed what the client keeps in the conversation. [ref]
- Edit: `settings_set {"type":"process"|"filament"|"printer","values":{...}}`; a filament slot's preset
  `"filament_slot":2` (a mixed filament's slot has none and is refused, `mixed_filament_slot`). The
  reply's `applied` and `changed` (the app's own adjustments included). [ref]
- Per-variant values (meta lists `variants` and `active`, e.g. X1C speeds and temperatures "Direct
  Drive Standard, Direct Drive High Flow"): a single value writes the entry in use
  (`nozzle_temperature 215` → `"215,220"`), `{"index":1,"value":300}` another entry, an array all. [ref]
- What changed: `settings_get {"type":"filament","compare":":saved"}` (unsaved edits), `":parent"`
  (vs the system preset), or a preset name. Revert one key:
  `settings_set {"type":"process","values":{"wall_loops":null}}` (`"reset_to":"parent"` for the
  parent's value); all: `settings_set {"type":"process","reset_all":true}`. [ref]
- Select: `preset_select {"type":"process","name":"0.24mm Draft @BBL X1C"}`; filament
  `"filament_slot":1`. With unsaved edits add `"preset_changes":"transfer"` (keep them on the new preset) or
  `"discard"`; without it the prompt opens. [ref]
- Save: `preset_save {"type":"filament","name":"Inland PLA+"}` saves the edited preset with its edits
  and selects it (`"overwrite":true` to replace one; `"inside_project":true` keeps it in the project
  only; printers too, Bambu ones included). To keep only some edits:
  `settings_set {"type":"filament","reset_all":true,"values":{...}}` → `preset_save`. [ref]
- Rename = `preset_save` under the new name → `preset_delete {"type":"filament","name":"<old>"}`. [ref]
- Delete: `preset_delete {"type":"filament","name":"Jayo PLA test 1"}` (no prompt; a printer must be
  selected first; refused for system presets and presets others inherit from). [ref]
- A preset for one object only: object overrides (`references/modifiers.md`), or save the edited
  preset with `"inside_project":true`.
- Share a filament with other printers: `settings_set {"type":"filament","values":
  {"compatible_printers":["Voron 2.4 350 0.4 nozzle"]}}` (`[]` = all) → `preset_save` over it. [hand]
- A preset that isn't installed (most system printers and filaments on a fresh install):
  `preset_list {"type":"printer","query":"X1 Carbon","include_uninstalled":true}` lists it with
  `installed: false` → `preset_select` it by name; the reply's `shown` names it. It stays in the
  selectors for this session only: after a restart the app selects an installed preset again. [ref]
- Install for good, as the Printer Selection and System Filaments dialogs do:
  `printer_list_models {"query":"Velo"}` → `printer_install {"models":[{"model":"Goofoo Velo+","vendor":"Goofoo","nozzles":["0.4"]}]}`
  (every nozzle without `nozzles`; `"preset_changes":"transfer"|"discard"` while presets have unsaved
  changes, which the reload drops); the newly added printer is selected, which can show a message
  (a plate's own bed type reset: `blocked_by_dialog`, answer it, then wait for the job).
  `filament_install {"names":["Bambu PETG HF"]}` or `{"vendor":"Generic","filament_type":"PETG"}`.
  `printer_uninstall` and `filament_uninstall` take the same names. [ref]
- Nozzle: `preset_select {"type":"printer","name":"Bambu Lab X1 Carbon 0.6 nozzle"}`, installed or
  not; the process follows (`0.30mm Standard @BBL X1C 0.6 nozzle`) and the filament may change:
  re-read `app_state.presets` and `preset_select` the filament again. [ref] `nozzle_type`
  (`hardened_steel`) is a printer setting; `preset_save` the printer to keep it.
- Plate type: `project_set_settings {"values":{"curr_bed_type":"Textured PEI Plate"}}`
  (`project_get_settings` `meta.offered` lists the printer's); one plate only: `plate_set_settings`
  (`references/output.md`). [ref]
- Print host: `settings_set {"type":"printer","values":{"host_type":"octoprint","print_host":"192.168.1.50","printhost_apikey":"…"}}`
  → `preset_save {"type":"printer","name":"…"}`. "Octo/Klipper" = `octoprint`; Moonraker `moonraker`.
- Import presets: `preset_import {"path":"/abs/My PLA.json"}`
  (`.json`, `.zip`, `.orca_filament`; `"on_conflict":"overwrite"` replaces presets that
  exist). The message names what was imported. [ref]
  - A `.orca_printer` / `.orca_bundle` imports its presets as `_local/<id>/<name>`, and its process
    and filament then don't show as compatible with its printer. Unzip it and import its
    `printer/*.json`, then `process/*.json`, then `filament/*.json`, one call each, and
    `preset_select` the printer, process and filament by name. [ref] The bundle's base printer must
    be installed.
- Export presets: `preset_export {"path":"/abs/dir"}` opens the
  dialog (replay, `"await":false`): `widget_click {"target":{"label":"Filament presets(.zip)","within":"Export Preset Bundle"}}`
  → `widget_set_value {"target":{"name":"<row>","within":"Export Preset Bundle"},"value":true}` (a
  checkbox is named as its row reads: for filament ZIPs the filament's name, `Bambu PLA Basic`) →
  `dialog_answer {"button":"ok"}` (`"await":false` too: it opens the result message) →
  `dialog_answer {"dialog":{"title":"Info"},"button":"ok"}`. [ref]
- Preferences: `preferences_get {"keys":["export_statistics"]}` reads one (no argument: all, tab by
  tab, with `kind`, `choices` and `refused`); `preferences_set {"values":{"export_statistics":true}}`
  changes it as its control does. A preference `preferences_get` marks `refused` (the language, the
  login region, the network plug-in, file associations, the MCP server's switch and port…) changes in
  the dialog (replay): `window_open {"window":"preferences","tab":"<tab>","key":"<key>"}`
  (`"await":false`) → `widget_set_value {"target":{"name":"<key>","within":"Preferences"},"value":…}`
  → `dialog_answer {"dialog":{"title":"Preferences"},"button":"close"}`. Checkboxes and fields save;
  most drop-down lists don't.
- Create a custom printer (fallback): run `references/replays/create-printer.json` with
  `replay_run {"replay":<its JSON>}` after changing the vendor, model, nozzle, bed size, origin,
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

- `preset_select` printer silently changes process and filaments: re-read `app_state.presets`.
- A printer or process `preset_select` that ends on another preset than the one asked for is an error
  naming both (`"B" was selected instead of "A".`, `other_preset_selected`), and B stays selected.
- `project_new {"discard_changes":true}` keeps modified presets (its reply lists them under
  `modified`); add `"preset_changes":"discard"` to drop them.
- `preset_list` leaves out presets that aren't installed unless `"include_uninstalled":true`; a name
  that isn't there at all (`preset_not_offered`, with the closest names in `similar`), or a preset that
  doesn't suit the selected printer (`preset_incompatible`), is refused with that reason.
- A key of another type is refused, naming the type to pass (`"k" is a filament setting`).
- PETG on the X1C's default Cool Plate fails to slice: set Textured PEI first. PETG + a PLA support
  filament fails ("nozzle temperatures are incompatible") unless the preference
  `enable_high_low_temp_mixed_printing` is on, which `preferences_set` refuses: change it in the dialog
  (above), whose checkbox asks to confirm a warning first ("Bed Temperature Difference Warning": Yes).

## Not possible

The setup wizard's choices (its page opens with `window_open {"window":"setup_wizard"}` and its
elements are `widget_list` items that `widget_click` and `widget_set_value` take by `{"web":"Next"}`
locators, but no recipe is verified; `printer_install` and `filament_install` install what it would).
