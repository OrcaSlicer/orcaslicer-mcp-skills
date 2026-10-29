# Speed Dial actions

Generated from `list_actions`; prefer a typed tool where one exists (`calibrate`, `arrange`, `transform`).

`run_action {"id": "orca_command:<key>:orca"}` runs one; `"param"` passes its value as a string. A `path`
param skips the file dialog: an absolute path for inputs, the JSON string `{"path": "/abs/out", "overwrite": true}`
for outputs. `list_actions {"query": "..."}` searches them with each param's help. Object commands act on what
`select` picked, and one that did nothing (nothing selected) is an error. Actions without a param that open a dialog
return `blocked_by_dialog`. The Help and Plugins groups are left out, except the setup wizard.

## Add Handy models

| key | param | does |
|---|---|---|
| `add_handy_3dbenchy` |  | Add a 3DBenchy object to the current plate. |
| `add_handy_autodesk_fdm_test` |  | Add a Autodesk FDM Test object to the current plate. |
| `add_handy_cali_cat` |  | Add a Cali Cat object to the current plate. |
| `add_handy_orca_badge` |  | Add a Orca Badge object to the current plate. |
| `add_handy_orca_cube` |  | Add a Orca Cube object to the current plate. |
| `add_handy_orca_string_hell` |  | Add a Orca String Hell object to the current plate. |
| `add_handy_orca_tolerance_test` |  | Add a Orca Tolerance Test object to the current plate. |
| `add_handy_orcasliced_combo` |  | Add a OrcaSliced Combo object to the current plate. |
| `add_handy_stanford_bunny` |  | Add a Stanford Bunny object to the current plate. |
| `add_handy_voron_cube` |  | Add a Voron Cube object to the current plate. |

## Add Primitive

| key | param | does |
|---|---|---|
| `add_primitive_cone` | text: base diameter and height in mm, separated by a space | Add a cone object to the current plate; fails while an object is selected. With a size, skips the size dialog. The shape may then be turned toward the part-cooling fan, so the size is along its own axes before that turn. |
| `add_primitive_cube` | text: X Y Z in mm, separated by spaces | Add a cube object to the current plate; fails while an object is selected. With a size, skips the size dialog. |
| `add_primitive_cylinder` | text: diameter and height in mm, separated by a space | Add a cylinder object to the current plate; fails while an object is selected. With a size, skips the size dialog. The shape may then be turned toward the part-cooling fan, so the size is along its own axes before that turn. |
| `add_primitive_disc` | text: diameter and height in mm, separated by a space | Add a disc object to the current plate; fails while an object is selected. With a size, skips the size dialog. The shape may then be turned toward the part-cooling fan, so the size is along its own axes before that turn. |
| `add_primitive_svg` | path | Add an SVG as a new object, or as a part of the selected object. With a path, uses that file without a dialog; the part is created in the background (wait for idle), and the SVG gizmo opens on it. |
| `add_primitive_sphere` | text: diameter in mm | Add a sphere object to the current plate; fails while an object is selected. With a size, skips the size dialog. |
| `add_primitive_text` |  | Add a Text object to the current plate. |
| `add_primitive_torus` | text: inner diameter and tube diameter in mm, separated by a space | Add a torus object to the current plate; fails while an object is selected. With a size, skips the size dialog. The shape may then be turned toward the part-cooling fan, so the size is along its own axes before that turn. |

## Object

| key | param | does |
|---|---|---|
| `obj_assemble` |  | Assemble two or more selected objects into one object of parts named Assembly. They are removed and it is added last, so object indices change; the reply has its index (object) and name. |
| `obj_arrange` |  | Arrange all objects on the unlocked plates with the saved Arrange options, as the toolbar does; the call returns when the arrangement is done. The arrange tool also takes options, one plate or fill bed. |
| `obj_orient` |  | Auto-orient the selection, or every object when nothing is selected; the call returns when it is done. |
| `obj_center` |  | Center the selection on its plate. |
| `obj_type_modifier` |  | Make the selected parts modifiers, as Change Type does; not the object's last solid part. Parts are listed by type, so part indices change. |
| `obj_type_negative` |  | Make the selected parts negative parts, as Change Type does; not the object's last solid part. Parts are listed by type, so part indices change. |
| `obj_type_part` |  | Make the selected parts (select {volumes} first) solid parts, as Change Type does. Parts are listed by type, so part indices change. |
| `obj_type_precise_seam` |  | Make the selected parts Center precise seams (seams keep their subtype), as Change Type does; not the object's last solid part, nor a text or SVG part. Parts are listed by type, so part indices change. |
| `obj_type_support_blocker` |  | Make the selected parts support blockers, as Change Type does; not the object's last solid part, nor a text or SVG part. Parts are listed by type, so part indices change. |
| `obj_type_support_enforcer` |  | Make the selected parts support enforcers, as Change Type does; not the object's last solid part, nor a text or SVG part. Parts are listed by type, so part indices change. |
| `obj_instances_down` |  | Remove an instance of the selected object. |
| `obj_delete_all` |  | Delete all objects. |
| `obj_delete` |  | Delete the selected objects or parts. |
| `deselect_all` |  | Clear the selection. |
| `obj_drop` |  | Drop the selection onto the bed. |
| `obj_instances_up` |  | Add an instance of the selected object. |
| `obj_mirror_x` |  | Mirror the selection along X. |
| `obj_mirror_y` |  | Mirror the selection along Y. |
| `obj_mirror_z` |  | Mirror the selection along Z. |
| `obj_order_by_filament` | text: all, to order every plate | Order the current plate's objects by filament, which sets the object list and print order, as the plate menu's Order By does. Choosing the order they already have reverses it. Object indices change; the reply lists the new order (order: [{object, name, plate}]) and descending. |
| `obj_order_by_height` | text: all, to order every plate | Order the current plate's objects by height, which sets the object list and print order, as the plate menu's Order By does. Choosing the order they already have reverses it. Object indices change; the reply lists the new order (order: [{object, name, plate}]) and descending. |
| `obj_order_by_name` | text: all, to order every plate | Order the current plate's objects by name, which sets the object list and print order, as the plate menu's Order By does. Choosing the order they already have reverses it. Object indices change; the reply lists the new order (order: [{object, name, plate}]) and descending. |
| `obj_order_by_size` | text: all, to order every plate | Order the current plate's objects by size (bounding box volume), which sets the object list and print order, as the plate menu's Order By does. Choosing the order they already have reverses it. Object indices change; the reply lists the new order (order: [{object, name, plate}]) and descending. |
| `obj_rename` | text: the new name | Rename the selected object or part (select one). With a name, skips the Rename dialog; a name with characters a file name cannot hold is refused. |
| `obj_fit_volume` |  | Scale the selection to fit the print volume. |
| `select_all` |  | Select all objects in the 3D view. |
| `obj_set_instances` | index: total number of instances, 1-1000 | Give the selected object this many instances, in one undo step. With a number, skips the dialog. New instances overlap; arrange afterwards. The reply has object and instances. |
| `obj_instances_to_objects` |  | Make each instance of the selected object a separate object (select {objects: [n]}); with some of its instances selected, only those. New objects are added last; the reply lists the objects the instances are in now (objects). |
| `obj_split_objects` |  | Split the selected object into separate objects. |
| `obj_split_parts` |  | Split the selected object into parts. |

## Filament

| key | param | does |
|---|---|---|
| `filament_add` |  | Add a filament slot, as the + button above the filament list does; the message names the new slot. It takes the next colour and the last slot's preset (change them with select_preset filament_slot and set_settings scope project filament_colour). It goes after the physical slots: mixed filaments move up by one, and objects on them follow. |
| `filament_delete` | index: 1-based filament slot; without it, the last physical slot, as the - button | Delete a filament slot, as its Delete menu item does; objects on it move to another slot. A mixed filament (project scope filament_is_mixed) is refused. |

## Plate

| key | param | does |
|---|---|---|
| `plate_add` |  | Add a plate. |
| `plate_delete` |  | Delete the current plate. |
| `plate_duplicate` |  | Duplicate the current plate. |
| `plate_goto` | index: 0-based plate index | Make a plate the current one. |
| `orca_plate_goto:0:orca` |  | Make plate 1 (index 0) the current plate. |
| `plate_rename` |  | Rename the current plate (opens a dialog). |
| `plate_toggle_lock` |  | Lock or unlock the current plate. |

## Slice & Export

| key | param | does |
|---|---|---|
| `export_3mf` | path | Export the model as a .3mf without slicing data. With a path, writes there without a dialog. |
| `export_all_sliced_file` | path | Export every plate's sliced file into one .gcode.3mf. With a path, writes there without a dialog. |
| `export_gcode` | path | Export the current plate's G-code. With a path, exports there without a dialog once the plate is sliced; the export runs in the background. |
| `export_stl` | path | Export all objects as one STL. With a path, writes there without a dialog. |
| `export_sliced_file` | path | Export the current plate's sliced file (.gcode.3mf). With a path, writes there without a dialog. |
| `send_gcode` |  | Open the Send G-code to printer host dialog for the current plate, as the Print button does for a printer with a print host (OctoPrint, Moonraker, PrusaLink, Duet...). Set the file name with ui_input and press Upload or Upload and Print with dialog_respond; the plate is sliced if needed, then uploaded in the background: get_state.uploads shows the row's state, and a failed upload also opens an error dialog. A replay uploads again. Fails for a printer without a print host, or one that prints through the Bambu network or printer agents. |
| `slice_all` |  | Slice every plate and show the Preview tab. Returns at once; use the slice tool to wait for results. |
| `slice_and_preview` |  | Slice the current plate and show the Preview tab. Returns at once; use the slice tool to wait for the result. |

## Export

| key | param | does |
|---|---|---|
| `export_drc_single` |  | Export all objects as one Draco (.drc) file. |
| `export_drc_multi` |  | Export each object as its own Draco (.drc) file. |
| `export_stl_multi` | path | Export each object as its own STL. With a directory, writes there without a dialog. |
| `export_config` | path | Export presets as a bundle or ZIP archives. Opens a dialog: click the label of the kind to export, check the presets (their checkboxes are named after them), press OK, then answer the result message. With a directory, OK writes there instead of asking for a folder. |
| `export_toolpaths_obj` |  | Export the sliced toolpaths as an OBJ. |

## Import

| key | param | does |
|---|---|---|
| `import_file` | path | Import models (3MF/STL/STEP/SVG/OBJ/AMF). With a path, imports that file without a dialog. |
| `import_configs` | path | Import presets. With a path, imports a preset .json, a .zip of them or an .orca_printer, .orca_filament or .orca_bundle without a dialog, and the message names the presets imported, replaced and kept; a .gcode loads the settings it was sliced with. |
| `import_zip_archive` |  | Import models from a ZIP archive (opens a file dialog). |

## Commands

| key | param | does |
|---|---|---|
| `go_to_layer` | percent: 0-100 | Show the Preview tab at a layer chosen as a percentage of the layer range. |
| `go_to_tab` | tab | Switch to a main window tab. |
| `load_project` | path | Open a project. With a path, opens that file without a dialog (asks first if the current project has unsaved changes). |
| `new_project` |  | Start a new, empty project (asks first if the current one has unsaved changes). |
| `open_preferences` | text: general, control, graphics, online or developer, optionally :<key>, e.g. control:auto_arrange | Open the Preferences dialog on a tab, scrolled to one preference when given (shown on its own tab). Its controls are named after their preference keys: ui_input {"target": {"name": key, "within": "Preferences"}} changes one through the dialog; get_settings scope preferences reads them. |
| `redo` |  | Redo the last undone change. |
| `save_project` |  | Save the project to its current file (asks for a name if it has none). |
| `save_project_as` | path | Save the project as a .3mf file. With a path, saves there without a dialog. |
| `undo` |  | Undo the last change to the project. |

## Calibration

| key | param | does |
|---|---|---|
| `calib_cornering` |  | Open the Cornering Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_flow_ratio` |  | Open the Flow Ratio Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_input_shaping_damp` |  | Open the Input Shaping Damping Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_input_shaping_freq` |  | Open the Input Shaping Frequency Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_max_volumetric` |  | Open the Max Volumetric Speed Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_pressure_advance` |  | Open the Pressure Advance Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_retraction` |  | Open the Retraction Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_temperature` |  | Open the Temperature Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |
| `calib_vfa` |  | Open the VFA Calibration dialog; the calibrate tool runs the same test with typed parameters and no dialog. |

## Presets

| key | param | does |
|---|---|---|
| `preset_bundle` |  | Open the preset bundle manager. |
| `sync_presets` |  | Synchronize user presets with the cloud account. |

## Printer

| key | param | does |
|---|---|---|
| `sync_ams` |  | Synchronize the filament list from the connected printer's AMS. |

## Mode

| key | param | does |
|---|---|---|
| `mode_advanced` |  | Switch the settings view to Advanced mode. |
| `mode_expert` |  | Switch the settings view to Expert mode. |
| `mode_simple` |  | Switch the settings view to Simple mode. |
| `toggle_developer_mode` |  | Turn Developer mode on or off. |

## View

| key | param | does |
|---|---|---|
| `view_fit_bed` |  | Fit the build plate in the 3D view. |
| `reset_window_layout` |  | Restore the default window layout. |
| `view_toggle_perspective` |  | Switch the 3D camera between perspective and orthographic. |
| `view_bottom` |  | Point the 3D camera: View: Bottom. |
| `view_default` |  | Reset the 3D camera to the default view and fit the plate. |
| `view_front` |  | Point the 3D camera: View: Front. |
| `view_iso` |  | Point the 3D camera: View: Isometric. |
| `view_left` |  | Point the 3D camera: View: Left. |
| `view_rear` |  | Point the 3D camera: View: Rear. |
| `view_right` |  | Point the 3D camera: View: Right. |
| `view_top` |  | Point the 3D camera: View: Top. |

## Help

| key | param | does |
|---|---|---|
| `help_setup_wizard` |  | Open the setup wizard. |
