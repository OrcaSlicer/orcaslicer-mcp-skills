# Python scripts

`run_script` runs Python inside OrcaSlicer as a main program (`__name__ == "__main__"`) against `orca`,
the plugin API: it reads the project's objects, meshes and presets, opens windows and progress dialogs
for the person, and calls these same MCP tools through `orca.host.tools.call(name, args)`, each with its
guards, undo and reply. Use it to compute over objects, meshes and settings, to make the same calls over
many objects in one round trip, or for a macro that asks the person something in a window. A part to model
is a Design tab job (`references/cad.md`); a command the user wants to keep in the app is a plugin
(`references/plugins.md`); a fixed list of tool calls to run again is a replay (`references/replays.md`).

## Before the first run

- A script always runs, restricted unless `get_app_state.python.scripts_unrestricted` is true. A
  restricted script's own Python code reads and writes files only in OrcaSlicer's data folder, reads its
  resources folder, the standard library and, in a file run, the file's folder, and can't start a thread
  or a program or use the network: such an access raises `PermissionError`, whose message names the
  setting. The tools it calls through `orca.host.tools` are not restricted, so export and file work goes
  through them. Only the person can lift the restriction, in Preferences > AI features > Python scripts,
  which lets scripts bypass the permission checks: when a script needs more, ask them.
  (`python.available` false: this build has no Python.)
- Look the API up: `search_python_api {"query":"mesh vertices"}` gives classes and members with their
  signature and documentation, best match first. Read the signature: `name` and `size` are properties
  (`o.name`, `bbox.size[2]`), `object(i)`, `bounding_box()` and `vertices()` are methods. Its reply's
  `numpy` says whether numpy is installed; mesh and matrix arrays need it.

## Two kinds of run

| | code sent as text | a file run |
|---|---|---|
| call | `run_script {"code":"…","args":{…}}` | `run_script {"code":"<the file's text>","filename":"/abs/x.py"}`, `filename` naming that existing file |
| `args` / `result` | `args` holds what you pass; `result`, if set, must be a JSON value or the run is `raised` | the same, except a `result` that isn't JSON is no result (the output says so) |
| `__file__`, `sys.argv` | none, `['']` | the path, `[path]` |
| modules beside it | not importable | `import helper` loads `helper.py` from its folder, afresh each run |
| PEP 723 `dependencies` | ignored | installed before the code runs (numpy, requests…) |
| `orca.host.plugin.storage()` | raises `RuntimeError` | the file's folder |

The reply is `{status, result, output, traceback?, message?}`: `status` is `finished`, `raised` (with
`traceback` and `message`), `cancelled` or `timed_out` (`timeout_s`, default 300, at most 3600). `print()`
comes back as `output`. A run that outlasts the reply budget replies `{"status":"running","job":"jN"}`:
`wait {"for":"job","job":"jN"}` (repeat it); `cancel_job {"job":"jN"}` stops it.

## The API at a glance

`search_python_api` has every member with its documentation; these are the entry points.

| Need | Call |
|---|---|
| Objects | `m = orca.host.model()`; `m.object_count()`, `m.object(i)` → `name`, `bounding_box()` (`.size`, `.min`, `.max`, mm tuples), `volume(i)` (a part: the Python API keeps the model's word), `instance(i)`, `config_value(key)` (its override, `None` when unset) |
| Meshes | `volume.mesh()` → `vertex_count()`, `vertex(i)`, `triangle(i)` without numpy; `vertices()`, `triangles()`, `face_normals()` as numpy arrays, in the part's own coordinates |
| Placement | `instance.offset()`, `rotation()`, `scaling_factor()`, `mirror()` tuples; `instance.matrix() @ volume.matrix()` (numpy) maps mesh to world |
| Preset values | `b = orca.host.preset_bundle()`; `b.full_config_value(key)` (the merged settings slicing uses), `b.current_process_preset()`, `b.current_printer_preset()`, `b.current_filament_presets()`, each `config_value(key)`; values are strings |
| Change settings | `orca.host.tools.call("set_settings", {...})`, as an agent does; or `orca.host.preset.process.update(key, "value")` (`.printer`, and `.filament.update(slot, key, value)` for a 1-based slot), the edit path the GUI's fields take |
| Edit another preset | `t = orca.host.preset.process.begin_update("<name>")` (`.printer`; `.filament.begin_update(slot, "<name>")`), then `t.set(key, "value")` and `t.commit()`. The name is selected first, as `select_preset` selects it: by the name its selector shows or its internal name, a hidden preset shown first, a filament assigned to its slot as a pick in the sidebar (`references/multicolor.md`). `ValueError` for a name the selector doesn't offer and for a mixed-colour slot, `IndexError` for a slot the printer lacks, `RuntimeError` when the tab ends on another preset. Over unsaved edits the Transfer-or-discard prompt opens and `run_script` replies `blocked_by_dialog`: answer it, then wait for the job |
| Tools | `orca.host.tools.call(name, args)` → the reply as a dict, for any command `search_tools` finds (the Speed Dial's and the Design tab's by their key); `orca.host.tools.run_replay(replay, vars)` runs a replay (`references/replays.md`); a failed call raises `orca.host.tools.ToolError` (`str(e)` its message, `e.details` its data: a refusal's `code`, as in `e.details.get("code") == "no_objects"`). `orca.host.tools.list()` lists every command as `{name, title, group, description, read_only, listed}`; `orca.host.tools.describe(name)` gives one with its `input_schema` and `examples` (an unknown name raises `ToolError`, `details["did_you_mean"]`) |
| Ask the person | `orca.host.ui.message(text, title, buttons="yes_no", icon="question")` → `"yes"`…; `create_window(html, title, width, height, on_message=…)` (modeless by default); `create_progress_dialog(title, message, maximum)` → `update(value, message)`; `push_notification(orca.host.ui.NotificationLevel.RegularNotificationLevel, text)` |
| The project | `orca.host.plater()`: `is_project_dirty()`, `is_presets_dirty()`. Its `project_path()` and `export_3mf_copy(path)` are UI thread only, so a script reads the path from `orca.host.tools.call("get_app_state", {})["project"]["path"]` and writes a copy with the `export` tool (`"format":"3mf"`) |

## Recipes

- Add three cubes of different heights, arrange them, and read their heights back: [ref]

  ```python
  import orca
  t = orca.host.tools
  t.call("new_project", {"discard_changes": True})
  for i in range(args["n"]):
      t.call("run_action", {"id": "add_primitive_cube", "args": {"size": [10, 10, 5 * (i + 1)]}})
      t.call("select_objects", {"none": True})       # a primitive is refused while something is selected
  t.call("arrange", {})
  m = orca.host.model()
  result = [[m.object(i).name, m.object(i).bounding_box().size[2]] for i in range(m.object_count())]
  ```

  sent as `run_script {"code":"<the code>","args":{"n":3}}` →
  `{"status":"finished","result":[["Cube",5.0],["Cube",10.0],["Cube",15.0]],"output":""}`.
  A call that returns a job: `t.call("wait", {"for": "job", "job": r["job"]})`.
- Mesh data with numpy when `search_python_api` says `"numpy": false`: write the script to a file that starts
  with a PEP 723 block and send it as a file run, which installs the dependencies first: [ref]

  ```python
  # /// script
  # dependencies = ["numpy"]
  # ///
  import numpy as np
  import orca
  m = orca.host.model()
  result = [[m.object(i).name, float(np.ptp(np.asarray(m.object(i).volume(0).mesh().vertices())[:, 2]))]
            for i in range(m.object_count())]
  ```

  `run_script {"code":"<the file's text>","filename":"/abs/heights.py"}`.
- A macro that asks the person in a window, then acts: [ref]

  ```python
  import queue
  import orca

  def main():
      answers = queue.Queue()
      orca.host.ui.create_window(
          "<input id='h' value='12'> <button onclick=\"window.orca.postMessage({h: "
          "document.getElementById('h').value})\">Go</button>",
          title="Cube height", width=300, height=160, on_message=answers.put)
      h = float(answers.get()["h"])          # waits until the person presses Go
      orca.host.tools.call("select_objects", {"none": True})
      orca.host.tools.call("run_action", {"id": "add_primitive_cube", "args": {"size": [10, 10, h]}})
      global result
      result = {"height": h}

  if __name__ == "__main__":
      main()
  ```

  The window's callback runs on the UI thread, where tool calls are refused, so it hands the message to
  the script's own code through a queue. The `run_script` call replies `running` while the window waits;
  the person (or you, for a test: `set_widget_value {"target":{"css":"#h","within":"Cube height"},"value":"18"}`,
  `click {"target":{"web":"Go","within":"Cube height"}}`) answers it, then `wait` for the job. The
  run's windows close when it ends.

## Traps

- A script that raised keeps what it changed before the error: read the state (`list_objects`) before
  running it again.
- While a modal dialog is open, tool calls that change the project raise "A modal dialog is open":
  answer it first (`get_app_state.dialogs`, `answer_dialog`), outside the script. That includes a modal
  window the script opened itself, so interactive macros use modeless windows.
- The code runs on a worker thread: calls documented as UI thread only (`orca.host.app_info()`,
  `orca.host.plater().project_path()`) raise `RuntimeError`. Window callbacks run on the UI thread and
  can't call the tools.
- The run ends once its code and the non-daemon threads and pool tasks it started are done; their tool
  calls work while it lasts. `wait {"for":"idle"}` does not wait for a `run_script` run, only for the
  calls it makes: wait for its job instead.
- A `result` set inside a function needs `global result`.
- Objects from `orca.host.model()` are references into the live project: fetch them again after a call
  that changes it (arrange, split and delete renumber objects).
- Overlapping runs share `sys.modules`: give helper modules distinctive names.
- `export_replay` records a run as one step with its code, which a replay runs again against the
  project as it is then; its own tool calls are not recorded.

## Not possible

Turning Python scripts on from a tool.
