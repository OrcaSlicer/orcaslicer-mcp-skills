# Developing OrcaSlicer with its MCP server

For an OrcaSlicer checkout: launching the app with MCP, reproducing issues, replays,
the tests, and platform notes. `docs/HLSD/mcp-server.md` has
the design (`docs/HLSD/design-tab.md` the Design tab's), `tools/mcp/README.md` the harness.

## Launch

Build Release (`cmake --build build/arm64 --config Release --target all --` on macOS,
`cmake --build build --config Release --target all --` on Linux, `build_win.bat` on Windows), then:

```bash
tools/mcp/orca_mcp.py launch --print-config              # macOS, Windows (python tools/mcp/orca_mcp.py …)
tools/mcp/orca_mcp.py launch --headless --print-config   # Linux with a desktop: a private headless one, off yours
tools/mcp/orca_mcp.py launch --xvfb --print-config       # Linux without a desktop (CI, a server)
```

On Linux prefer `--headless` (see "Linux display notes"); use `--xvfb` only where no desktop is
installed. `--headless` falls back to Xvfb by itself when it finds no headless compositor.

`launch` starts the build's binary (or `--bin PATH`; on macOS the bundle under `build/<arch>`, on
Windows `build-clang\src\Release\orca-slicer.exe` too) with `--mcp` and a fresh temporary datadir,
seeded with a finished setup wizard and one printer, Bambu Lab X1 Carbon 0.4 by default
(`--printer "<model>@<nozzle>" --vendor <profile vendor> --filament "<preset>"` for another). It
waits until the app is ready and prints the session: `url`, `port`, `token`, `pid`, `datadir`; with
`--print-config` also the `claude mcp add` line.

- `--cad` starts the app with `--cad`, which turns the Design (CAD) tab on for that run; the Design
  commands need it (`references/cad.md`). It never writes the datadir's settings, so pass it on every
  launch, also with a reused datadir.
- `--unrestricted-scripts` starts the app with `--unrestricted-scripts`, which lets Python scripts
  (`run_script`) run unrestricted for that run. It never writes the datadir's settings either. Without
  it the datadir's setting decides: off in a fresh one, so scripts run restricted
  (`references/python.md`). No tool turns it on. A `.py` given to `--run` runs as a Python script.
- `--datadir DIR` reuses a datadir: pass the first launch's on every relaunch, so presets stay the
  same. Never point it at your real OrcaSlicer profile.
- `--port N` when 13619 is taken (your own OrcaSlicer with MCP on). Several instances at once: give
  each its own `ORCA_MCP_SESSION` file and `--port`.
- Arguments after `--` go to the app. `tools/mcp/orca_mcp.py stop` ends the app and its private
  display.
- Launch from a folder that holds no `.py` files: the app's working directory is on Python's module
  path, so a file there can shadow a standard module that `run_script` needs.
- Startup problems go to `<datadir>/mcp-launch.log`; at run time `get_log {"level":"warning"}`.

Talk to the app in two ways:

- **The harness**, from any shell: `tools/mcp/orca_mcp.py call <tool> '<json args>' --wait` (`--wait`
  follows a returned job). Shortcuts: `tools`, `widgets`, `screenshot /abs/out.png --target canvas`,
  `run-replay FILE --out DIR`. From Python: `sys.path.insert(0, "tools/mcp")`, `OrcaMcp(port, token)`,
  `call()`, `await_job()`, `images()`, `call_command()` (any command by name through `call_tool`,
  its job followed to the end).
- **Agent tools**, registered with
  `claude mcp add orcaslicer -- /abs/build/src/Release/orca-slicer --mcp-stdio --datadir DIR`
  (Codex: `codex mcp add orcaslicer -- …` with the same command). The
  stdio bridge reads the port and token of the instance on that datadir at every connect, so
  relaunches need no new registration. Register before starting the session that uses them.

## Reproduce an issue

1. Read the issue (`gh issue view <N> --repo OrcaSlicer/OrcaSlicer --comments`): presets, changed
   settings, steps. Download attachments into a work folder; every path you pass is absolute.
2. `get_app_state`: `ready`, `presets`, `dialogs`, `display_backend`. After a forced stop the app asks
   to restore the project: `auto_answer_dialogs {"rules":[{"match":"restore","button":"no"},
   {"match":"unsaved changes","button":"no"}]}` (a rule also answers a matching open dialog).
3. `open_files {"paths":["/abs/work/issue.3mf"]}` (a project, with its presets and plates;
   `"as":"geometry"` for its objects only; `"discard_changes":true` over a modified project).
4. Configure with `list_presets` / `select_preset` and `get_settings {"query":…}` /
   `set_settings` (the SKILL.md and `references/settings.md`).
5. Do the steps with typed tools, then `run_action`, then the GUI tools (the area references).
6. `slice {}`; read `status` and `plates[].outcome`, the statistics, `get_slice_stats`.
7. Point the view: `set_view {"tab":"preview","camera":"iso","layer":40,"view_type":"feature_type"}`;
   `"layer_z":3.2`, `"legend":{"travel":true}`; `get_app_state.preview` reads it back.
8. `screenshot {"target":"canvas","save_path":"/abs/work/before.png","overwrite":true}`; `"main"`,
   `"dialog"`, a window's ref from `list_windows`, or a widget's locator. Read the full-size PNG to
   look closer.
9. `export_replay {"path":"/abs/work/repro.json"}`. Calls that failed are left out, whether at once
   or as a job later; a call whose job still runs is left out and named in `left_out_running` (wait,
   export again). `export_replay {"clear":true}` at the start drops exploration. Then edit it: add
   `expect`s that hold once the bug is fixed, rename the screenshot to `after.png`, use
   `${replay_dir}/…` for files kept beside it.
10. Fix, rebuild, `stop`, launch with the same `--datadir`, then
    `tools/mcp/orca_mcp.py run-replay /abs/work/repro.json --out /abs/work` (exit 0 = passes).
    Without the harness: `orca-slicer --datadir DIR --run repro.json --outputdir /abs/work`
    (on a launch's private display, export the variables in the session file's `display_env`); it
    writes `result.json` and the screenshots to the `--outputdir` folder and
    exits 0 (passed), 1 (a step failed) or 2 (the replay is invalid or cannot be read).
11. Compare `before.png` and `after.png` and each step's result.

## Replay format

The replay format (steps, placeholders, `expect`, `await`/`save_as`, `optional`, the result) is in
`references/replays.md`. A repro replay starts from a known scene (`open_files` with
`discard_changes`), ends with the `expect`s that hold once the bug is fixed, and keeps its files beside
it as `${replay_dir}/…`. A `.py` given to `--run` runs as a Python file run
(`references/python.md`).

## Plugins in a test instance

To load a plugin in a harness instance without its prompts, side-load it before the launch as
`tools/mcp/test_plugin_tools.py` does: copy it to `<datadir>/orca_plugins/<key>/<key>.py` and write
`<datadir>/orca_plugins/<key>/.install_state.json`:

```json
{"installed_from": "local", "installed_version": "1.0.0", "plugin_name": "<name>", "enabled": true,
 "capabilities": [{"<capability name>": true}],
 "permissions": {"threading": true}}
```

`threading` and `full_access` are booleans; `fs_read`, `fs_readwrite`, `network_http`,
`network_socket`, `process` and `events` are lists. Tool calls and operations need no permission.
A running instance loads one through the Plugins dialog instead (`references/plugins.md`). `tools/mcp/tests/plugins/` holds the fixture
plugins the harness tests use: `mcp_tools_fixture` (tool calls from a plugin, an operation),
`mcp_web_fixture` (every kind of page control, in a dialog, a window, a dock panel and a config page)
and `mcp_tools_broken` (a load that fails).

## Rules the GUI tools follow

- Refs go stale when the UI changes (and after a language change): call `list_widgets` again,
  narrowed with `"query"`. Replays store locators, never refs.
- While an operation runs (a slice, an export, or a background job a call started: arrange, orient,
  fill), other changes are refused; `wait` for its job or `cancel_job` it. A job a person started (typing
  in the Text gizmo) refuses changes too; reads still answer.
- A Speed Dial command that did nothing is an error with the reason.
- A command, the tool that does the same thing and the plugin API's preset calls go through one
  function of the operations layer (`src/slic3r/GUI/Operations/`), so every route accepts and refuses
  the same states, with one wording and a `code` on the line after it (`docs/HLSD/mcp-server.md`,
  "The operations layer").
- wx dialogs take `answer_dialog` or an auto-answer, which presses the button and runs its handler;
  their `message` reads their controls (SKILL.md, "Dialogs"). A native message box takes
  `answer_dialog` on Windows and Linux. A file or folder picker, and any native dialog on macOS, only
  takes an `auto_answer_dialogs` rule registered before the step that opens it (a picker's rule
  matches its prompt, and hands it a file with `"button":"ok","path":"/abs/file"`, except on Linux;
  without a path it cancels the picker).
- Web pages (the setup wizard, sign-in, web dialogs, plugin windows and dock panels) are driven like
  widgets: `list_widgets` lists each shown page's elements under its web view, and `click`,
  `set_widget_value` and `press_keys` take them by ref or by `{"web":"Next","kind":"button","css":"#id","within":…}`.
  `run_javascript` runs JavaScript in a page; while a modal is open, only its pages answer. Commit a
  modal page with its own button (`closed: true` in the reply).

## Tests

The replays in `tools/mcp/tests/` pass on the current build; their steps are verified call
shapes. Run one with `tools/mcp/orca_mcp.py run-replay tools/mcp/tests/<name>.json --out /tmp/<name>`:

- `replay_basic.json` (smallest end-to-end), `../smoke_test.json`;
- `scene.json` (transform, add_part, arrange, fill, object commands), `layers.json` (height ranges,
  variable layer height, layer G-codes), `multicolor.json` (part filaments, project scope),
  `plates.json` (plate scope, filament slots), `presets.json` (save, compare, delete, import,
  preferences), `calibration.json`, `slice_stats.json`, `preview.json`;
- `gizmos.json` (all 17 gizmos; at another size via `run_replay` `"vars":{"window_size":[1100,760]}`);
- `context_menus.json`, `rows.json`, `plate_icons.json`, `dialogs_canvas_input.json`,
  `webview_modal.json`, `web_dialogs.json` (web pages driven like widgets), `agent_forms.json`
  (argument names and forms agents often write), `recording.json` (needs an ffmpeg with libx265 and
  libx264);
- `ops_*.json`, one per operation of the operations layer, each driven through every route that
  reaches it (the command, the tool and, for presets, a pick in the sidebar and the plugin API): they
  record what each call leaves, and `result.json` keeps the refusals' texts to compare between two
  builds. The preset ones need a freshly launched app, and those whose commands write files a fresh
  `--out`;
- with `--cad`: `cad_sketch_to_solid.json`, `cad_sketch_edit.json`, `cad_topology_targets.json`,
  `cad_features.json`, `cad_doc.json`, `cad_canvas.json`.

Harness tests (`tools/mcp/test_*.py`, each against the launched instance): `test_stdio_bridge.py`,
`test_recovery_under_load.py`, `test_cancel_pending_slice.py`, `test_ref_locator_export.py`,
`test_worker_admission.py`, `test_export_running_job.py` (about 2 min), `test_send_gcode.py` (stub
host), `test_operation_routes.py` (a command, its tool and the plugin API accept and refuse the same
states in the same words; on a freshly launched app), `test_set_view_preview.py`,
`test_set_view_camera.py`, `test_set_window.py`,
`test_list_windows.py`, `test_webview_modal.py`, `test_action_replay_hidden.py`, and with `--cad`
`test_cad_replay.py`, `test_cad_slow_build.py`, `test_cad_failed_job.py`, `test_cad_examples.py` (every
Design command's examples) and the Design tab's GUI tests `test_cad_sketching.py`, `test_cad_click_edit.py`
and `test_cad_context_menu.py`.
`test_web_ui.py` drives a fixture plugin's pages: `test_web_ui.py --install DIR`, launch with
`--datadir DIR`, then run it. `test_plugin_tools.py` (a plugin's calls into the tools) launches its own
instances, and `test_orca_mcp_attach.py` (how the harness attaches to a running app) needs none.
Unit tests: `tests/slic3rutils/test_mcp_*.cpp`, `test_command_catalog.cpp` and `test_operations.cpp`
(`ctest --test-dir ./tests/slic3rutils -C Release`), tags `[Mcp]`, `[Mcp][CAD]`, `[CommandCatalog]`,
`[Operations]`.

A new feature MCP should reach: a Speed Dial command in `NativeCommands.cpp` (key, title, group,
function, a description, and an `input_schema` of the arguments that stand for its dialog: with
none it asks, with them it runs without asking) is enough for most; a typed tool in the matching
`McpTools*.cpp` when it needs structured arguments or results. Both are commands of one catalog,
which `search_tools` and `call_tool` reach by key. What the command or tool does (its checks, its
refusals and their wording, its undo snapshot) goes into an operation in
`src/slic3r/GUI/Operations/` that both call. A new or changed command changes the command
reference, `tests/data/command_catalog.json`, which a unit test compares with the catalog
(`docs/HLSD/mcp-server.md`, "Making a feature reachable" and "The operations layer").

## Linux display notes

- `--headless` starts a headless Wayland compositor on a session bus of its own and runs the app on
  its XWayland (`display_backend` `x11`), off your desktop: the app draws with the GPU's OpenGL
  driver, and the compositor is a window manager, so focus, stacking and popup lists behave as on a
  desktop, and the D-Bus services the app starts, such as the desktop portals, stay out of your
  session. It takes KWin (`kwin_wayland --virtual`),
  else mutter; `--headless kwin|mutter` names one. Its output goes to `<datadir>/mcp-display.log`.
- GNOME, `mutter --headless` (test required): not yet run on a GNOME machine. Before relying on it,
  check that the window shows and takes input, that `main` screenshots are exact, and that the app's
  log (`got opengl version …`) names the GPU's driver, not `llvmpipe`; report what you find.
- `--xvfb` is for a machine without a desktop (CI, a server). It is X11 too, but draws OpenGL in
  software (`llvmpipe`) and runs no window manager, so it is slower and further from what people see:
  a popup list (a combo box's drop-down, the Print button's menu) closes as soon as it opens, so
  `list_widgets` finds no rows in it and `tests/rows.json` fails.
- On X11 (`--headless`, `--xvfb`) `main` and `dialog` screenshots are exact while the window fits on
  the screen (default 1600×1000; `--width`/`--height`, or `set_window {"size":[1400,900]}`).
- Under Wayland without either, `get_app_state.display_backend` is `wayland`: `main` and `dialog`
  screenshots are `approximate` (drawn from the widget tree, web views may be blank). Use `"canvas"`,
  exact everywhere, or launch with `GDK_BACKEND=x11`.
- On X11 OrcaSlicer's own popups are drawn into the capture; another application's window over it
  comes out black (`approximate`). A private display has nothing else on it.
- Record videos under X11 too, where a recording shows the app's windows as they are on the screen:
  `orca_mcp.py launch --headless --width 1920 --height 1080` (`--xvfb` without a desktop), then
  `set_window {"position":[0,0],"size":[1920,1080]}` before `start_recording` so the window fills the
  screen. A dialog's title bar, drawn by the compositor, is not in the video; its contents are.
  Under Wayland a recording draws each dialog centred on the window it was opened from, where the
  compositor usually puts it, and leaves out modeless windows and native dialogs (message boxes, file
  pickers); `stop_recording`'s `note` says so.
- Launching by hand on a fresh datadir: set `SSL_CERT_FILE` (the harness does), or a CA-bundle
  prompt blocks startup.

## Windows notes

- `main` and `dialog` screenshots come from Windows Graphics Capture (`PrintWindow` where that is
  unavailable): exact even when covered, web views and 3D views included; a minimized window can't be
  captured.
- The app takes the foreground briefly at start. The GUI tools synthesize input inside the app and
  never move the real pointer. The harness gives the instance its own WebView2 profile.

## macOS notes

- The app opens on your desktop and takes the focus while it starts. The GUI tools never move the
  real pointer.
- `canvas` screenshots are exact, and so are `main`, `dialog` and widget ones on macOS 14.4 and later
  (ScreenCaptureKit); before it, or with the window partly off its display, they are `approximate`
  (the content view drawn without the title bar; web views may be blank).
- A native dialog holds the UI thread until answered: the call that opened it returns
  `blocked_by_dialog` with the dialog `native: true`, and every UI tool fails, naming it, until it
  closes. Register `auto_answer_dialogs` first; a folder picker matches by its prompt ("Choose
  Download Directory").
- `Ctrl` and `Cmd` both mean Command in `press_keys` chords and in `click`/`mouse_gesture` `modifiers`;
  `RawCtrl` is the Control key.
- Context menus are captured, never shown, so a click that opens one cannot hold the UI thread;
  `get_app_state.closed_menus` lists menus closed unseen.
