# Developing OrcaSlicer with its MCP server

For an OrcaSlicer checkout: launching the app with MCP, reproducing issues, scripts that replay,
the test scripts, and platform notes. `docs/HLSD/mcp-server.md` has
the design (`docs/HLSD/design-tab.md` the Design tab's), `tools/mcp/README.md` the harness.

## Launch

Build Release (`cmake --build build/arm64 --config Release --target all --` on macOS,
`cmake --build build --config Release --target all --` on Linux, `build_win.bat` on Windows), then:

```bash
tools/mcp/orca_mcp.py launch --print-config          # macOS, Windows (python tools/mcp/orca_mcp.py …)
tools/mcp/orca_mcp.py launch --xvfb --print-config   # Linux, on a private X display
```

`launch` starts the build's binary (or `--bin PATH`; on macOS the bundle under `build/<arch>`, on
Windows `build-clang\src\Release\orca-slicer.exe` too) with `--mcp` and a fresh temporary datadir,
seeded with a finished setup wizard and one printer, Bambu Lab X1 Carbon 0.4 by default
(`--printer "<model>@<nozzle>" --vendor <profile vendor> --filament "<preset>"` for another). It
waits until the app is ready and prints the session: `url`, `port`, `token`, `pid`, `datadir`; with
`--print-config` also the `claude mcp add` line.

- `--cad` turns the Design (CAD) tab on, which `cad_call` needs. The app reads it only at startup,
  so pass it on every launch, also with a reused datadir.
- `--datadir DIR` reuses a datadir: pass the first launch's on every relaunch, so presets stay the
  same. Never point it at your real OrcaSlicer profile.
- `--port N` when 13619 is taken (your own OrcaSlicer with MCP on). Several instances at once: give
  each its own `ORCA_MCP_SESSION` file and `--port`.
- Arguments after `--` go to the app. `tools/mcp/orca_mcp.py stop` ends the app and its Xvfb.
- Startup problems go to `<datadir>/mcp-launch.log`; at run time `get_log {"level":"warning"}`.
- Where `tools/mcp/bench/run_bench.py` exists (the agent benchmark), `run_bench.py app --port N
  --bin PATH [--seed '<case seed JSON>']` starts an app set up as benchmark runs are (CAD on, extra
  printers and filaments from the seed) and records it in `$ORCA_MCP_SESSION`.

Talk to the app in two ways:

- **The harness**, from any shell: `tools/mcp/orca_mcp.py call <tool> '<json args>' --wait` (`--wait`
  follows a returned job). Shortcuts: `tools`, `snapshot`, `screenshot /abs/out.png --target canvas`,
  `run-script FILE --out DIR`. From Python: `sys.path.insert(0, "tools/mcp")`, `OrcaMcp(port, token)`,
  `call()`, `await_job()`, `images()`.
- **Agent tools**, registered with
  `claude mcp add orcaslicer -- /abs/build/src/Release/orca-slicer --mcp-stdio --datadir DIR`
  (Codex: `codex mcp add orcaslicer -- …` with the same command). The
  stdio bridge reads the port and token of the instance on that datadir at every connect, so
  relaunches need no new registration. Register before starting the session that uses them.

## Reproduce an issue

1. Read the issue (`gh issue view <N> --repo OrcaSlicer/OrcaSlicer --comments`): presets, changed
   settings, steps. Download attachments into a work folder; every path you pass is absolute.
2. `get_state`: `ready`, `presets`, `dialogs`, `display_backend`. After a forced stop the app asks
   to restore the project: `dialog_auto_answer {"rules":[{"match":"restore","button":"no"},
   {"match":"unsaved changes","button":"no"}]}` (a rule also answers a matching open dialog).
3. `open_files {"paths":["/abs/work/issue.3mf"]}` (a project, with its presets and plates;
   `"as":"geometry"` for its objects only; `"discard_changes":true` over a modified project).
4. Configure with `list_presets` / `select_preset` and `get_settings {"query":…}` /
   `set_settings` (the SKILL.md and `references/settings.md`).
5. Do the steps with typed tools, then `run_action`, then the GUI tools (the area references).
6. `slice {}`; read `status` and `plates[].outcome`, the statistics, `get_preview`.
7. Point the view: `set_view {"tab":"preview","camera":"iso","layer":40,"view_type":"feature_type"}`;
   `"layer_z":3.2`, `"legend":{"travel":true}`; `get_state.preview` reads it back.
8. `screenshot {"target":"canvas","save_path":"/abs/work/before.png","overwrite":true}`; `"window"`,
   `"dialog"`, or a widget's locator. Read the full-size PNG to look closer.
9. `export_script {"path":"/abs/work/repro.json"}`. Failed calls are left out, a job that failed
   later included; a call whose job still runs is left out and named in `left_out_running` (wait,
   export again). `export_script {"clear":true}` at the start drops exploration. Then edit it: add
   `expect`s that hold once the bug is fixed, rename the screenshot to `after.png`, use
   `${script_dir}/…` for files kept beside it.
10. Fix, rebuild, `stop`, launch with the same `--datadir`, then
    `tools/mcp/orca_mcp.py run-script /abs/work/repro.json --out /abs/work` (exit 0 = passes).
    Without the harness: `orca-slicer --datadir DIR --mcp-script repro.json --mcp-script-out /abs/work`
    (under Xvfb set `DISPLAY` and `GDK_BACKEND=x11`); it writes `result.json` and the screenshots and
    exits 0 (passed), 1 (a step failed) or 2 (invalid script).
11. Compare `before.png` and `after.png` and each step's result.

## Script format

```jsonc
{ "orca_script": 1, "name": "repro #12345",
  "vars": {"model": "${script_dir}/part.3mf"},          // ${vars.model} below
  "steps": [
    {"tool": "dialog_auto_answer", "args": {"rules": [{"match": "unsaved", "button": "no"}]}},
    {"tool": "open_files", "args": {"paths": ["${vars.model}"], "discard_changes": true}},
    {"tool": "set_settings", "args": {"scope": "print", "values": {"wall_loops": 3}}},
    {"tool": "slice", "expect": {"/plates/0/outcome": {"eq": "ok"}}},
    {"tool": "set_view", "args": {"tab": "preview", "camera": "iso"}},
    {"tool": "screenshot", "args": {"target": "canvas", "save_path": "${out_dir}/after.png", "overwrite": true}}
  ]}
```

Placeholders: `${script_dir}`, `${out_dir}`, `${env:NAME}`, `${vars.NAME}`, `${jobs.NAME}`. A step
that returns a job is awaited (up to `timeout_s`, default 120) before its `expect`. For a dialog the
step opens: `"await": false, "save_as": "x"`, the answering steps, then
`{"tool": "wait", "args": {"for": "job", "job": "${jobs.x}"}}`. `expect` maps a JSON pointer to
matchers `eq ne gt ge lt le contains regex exists empty`. `"optional": true` lets a step fail; the
test scripts use it for refusals, followed by a check that nothing changed. A script's
`dialog_auto_answer` rules end with it. `run_script {"path":…,"vars":{…}}` overrides `vars`.

## Rules the GUI tools follow

- Refs go stale when the UI changes (and after a language change): snapshot again, narrowed with
  `"query"`. Scripts store locators, never refs.
- While an operation runs (a slice, an export, or a background job a call started: arrange, orient,
  fill), other changes are refused; `wait` for its job or `cancel` it. A job a person started (typing
  in the Text gizmo) refuses changes too; reads still answer.
- A Speed Dial command that did nothing is an error with the reason.
- Native dialogs (message boxes, file pickers) can't be clicked: register `dialog_auto_answer` before
  the step that opens them (a picker can only be cancelled; its text is its prompt). wx dialogs take
  `dialog_respond` or an auto-answer, which presses the button and runs its handler.
- `webview_eval` runs in a modal's page only for the setup wizard and the preset bundle dialogs
  (`orca_command:help_setup_wizard:orca`, `orca_command:preset_bundle:orca`, `"await": false` in a
  script); commit such a page with its own button (`closed: true` in the reply).

## Test scripts

Replay scripts in `tools/mcp/tests/` pass on the current build; their steps are verified call
shapes. Run one with `tools/mcp/orca_mcp.py run-script tools/mcp/tests/<name>.json --out /tmp/<name>`:

- `script_basic.json` (smallest end-to-end), `../smoke_test.json`;
- `scene.json` (transform, add_part, arrange, fill, object commands), `layers.json` (height ranges,
  variable layer height, layer G-codes), `multicolor.json` (part filaments, project scope),
  `plates.json` (plate scope, filament slots), `presets.json` (save, compare, delete, import,
  preferences), `calibration.json`, `slice_stats.json`, `preview.json`;
- `gizmos.json` (all 17 gizmos; at another size via `run_script` `"vars":{"window_size":[1100,760]}`);
- `context_menus.json`, `rows.json`, `plate_icons.json`, `dialogs_canvas_input.json`,
  `webview_modal.json`;
- with `--cad`: `cad_sketch_to_solid.json`, `cad_sketch_edit.json`, `cad_topology_targets.json`,
  `cad_features.json`, `cad_doc.json`.

Harness tests (`tools/mcp/test_*.py`, each against the launched instance): `test_stdio_bridge.py`,
`test_recovery_under_load.py`, `test_cancel_pending_slice.py`, `test_ref_locator_export.py`,
`test_worker_admission.py`, `test_export_running_job.py` (about 2 min), `test_send_gcode.py` (stub
host), `test_set_view_preview.py`, `test_webview_modal.py`, and with `--cad` `test_cad_replay.py`,
`test_cad_slow_build.py`. Unit tests: `tests/slic3rutils/test_mcp_*.cpp` (`ctest --test-dir
./tests/slic3rutils -C Release`), tags `[Mcp]`, `[Mcp][CAD]`.

A new feature MCP should reach: a Speed Dial command in `NativeCommands.cpp` (key, description,
`Path` param that skips its dialog) is enough for most; a typed tool in the matching
`McpTools*.cpp` when it needs structured arguments or results (`docs/HLSD/mcp-server.md`, "Making a
feature reachable").

## Linux display notes

- `--xvfb` runs on X11, so `window` and `dialog` screenshots are exact while the window fits on the
  screen (default 1600×1000; `--width`/`--height`, or `set_view {"window_size":[1400,900]}`).
- Under Wayland without `--xvfb`, `get_state.display_backend` is `wayland`: `window` and `dialog`
  screenshots are `approximate` (drawn from the widget tree, web views may be blank). Use `"canvas"`,
  exact everywhere, or launch with `GDK_BACKEND=x11`.
- On X11 OrcaSlicer's own popups are drawn into the capture; another application's window over it
  comes out black (`approximate`). Xvfb has nothing on top.
- Launching by hand on a fresh datadir: set `SSL_CERT_FILE` (the harness does), or a CA-bundle
  prompt blocks startup.

## Windows notes

- `window` and `dialog` screenshots come from `PrintWindow`: exact even when covered, web views and
  3D views included; a minimized window can't be captured.
- The app takes the foreground briefly at start. The GUI tools synthesize input inside the app and
  never move the real pointer. The harness gives the instance its own WebView2 profile.

## macOS notes

- The app opens on your desktop and takes the focus while it starts. The GUI tools never move the
  real pointer.
- `canvas` screenshots are exact; `window`, `dialog` and widget ones are `approximate` (content view
  without the title bar; web views may be blank).
- A native dialog holds the UI thread until answered: the call that opened it returns
  `blocked_by_dialog` with the dialog `native: true`, and every UI tool fails, naming it, until it
  closes. Register `dialog_auto_answer` first; a folder picker matches by its prompt ("Choose
  Download Directory").
- `Ctrl` and `Cmd` both mean Command in `ui_key` chords and in `ui_click`/`ui_mouse` `modifiers`;
  `RawCtrl` is the Control key.
- Context menus are captured, never shown, so a click that opens one cannot hold the UI thread;
  `get_state.closed_menus` lists menus closed unseen.
