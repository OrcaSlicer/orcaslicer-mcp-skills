# Python scripts

`run_python` runs Python inside OrcaSlicer against `orca.host`, the plugin API, and calls the MCP tools
through `orca.host.tools.call(name, args)`. Use it to compute over objects, meshes and settings, or to
make the same calls over many objects in one round trip. A part to model is a Design tab job
(`references/cad.md`), not a mesh a script writes.

## Before the first run

- `get_state.python.scripts_allowed` must be true. Only the person can turn it on, in Preferences >
  AI features > Python scripts, since scripts run with OrcaSlicer's full access to the computer. When it
  is off, ask them or do the work with tools.
- Look the API up: `python_api {"query":"mesh vertices"}` gives classes and members with their
  signature and documentation, best match first. Read the signature: `name` and `size` are properties
  (`o.name`, `bbox.size[2]`), `object(i)`, `bounding_box()` and `vertices()` are methods. Its reply's
  `numpy` says whether numpy is installed; mesh and matrix arrays need it.

## Recipes

- Add three cubes of different heights, arrange them, and read their heights back: [ref]

  ```python
  import orca
  t = orca.host.tools
  t.call("new_project", {"discard_changes": True})
  for i in range(args["n"]):
      t.call("run_action", {"id": "add_primitive_cube", "param": f"10 10 {5 * (i + 1)}"})
      t.call("select", {"none": True})
  t.call("arrange", {})
  m = orca.host.model()
  result = [[m.object(i).name, m.object(i).bounding_box().size[2]] for i in range(m.object_count())]
  ```

  sent as `run_python {"code":"<the code>","args":{"n":3}}` →
  `{"status":"finished","result":[["Cube",5.0],["Cube",10.0],["Cube",15.0]],"output":""}`.
  - `args` holds what you pass; set `result` to a JSON value to return it; `print()` comes back as
    `output`.
  - `status` is `finished`, `raised` (with `traceback` and `message`), `cancelled` or `timed_out`
    (`timeout_s`, default 300).
  - A tool call from Python has your guards, undo and reply; a failed one raises
    `orca.host.tools.ToolError` (`str()` is its message). One that returns a job:
    `t.call("wait", {"for": "job", "job": r["job"]})`.
- Mesh data with numpy when `python_api` says `"numpy": false`: write the script to a file that starts
  with a PEP 723 block, and send its text with the file's path as `filename`, which makes it a file run
  that installs the dependencies first: [ref]

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

  `run_python {"code":"<the file's text>","filename":"/abs/heights.py"}`. Vertices are in the part's
  own mesh coordinates; world ones are `instance.matrix() @ volume.matrix()` applied to them.

## Traps

- A script that raised keeps what it changed before the error: read the state (`list_objects`) before
  running it again.
- While a modal dialog is open, tool calls raise "A modal dialog is open": answer it first
  (`get_state.dialogs`, `dialog_respond`), outside the script.
- The code runs on a worker thread: calls documented as UI thread only raise `RuntimeError`.
- Objects from `orca.host.model()` are references into the live project: fetch them again after a call
  that changes it (arrange, split and delete renumber objects).
- `export_script` records a run as one step with its code, which a replay runs again against the
  project as it is then.

## Not possible

Turning Python scripts on from a tool.
