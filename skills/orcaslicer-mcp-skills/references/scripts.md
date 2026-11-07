# Scripts of tool calls

A script is a JSON list of tool calls that `run_script` runs in order, checking each reply. Use one to
send a known sequence in one round trip, to answer the dialog a step opens without racing it, to check
results with `expect`, or to keep a session as a replayable file (`export_script`). Logic over the
project's data (loops, conditions, math) is Python instead (`references/python.md`), which can run a
script too.

## Format

```jsonc
{ "orca_script": 1, "name": "two walls, sliced",
  "vars": {"model": "${script_dir}/part.3mf", "walls": 2},          // ${vars.model} below
  "steps": [
    {"tool": "dialog_auto_answer", "args": {"rules": [{"match": "unsaved changes", "button": "no"}]}},
    {"tool": "open_files", "args": {"paths": ["${vars.model}"], "discard_changes": true}},
    {"tool": "set_settings", "args": {"scope": "print", "values": {"wall_loops": "${vars.walls}"}}},
    {"tool": "slice", "expect": {"/plates/0/outcome": {"eq": "ok"}}},
    {"tool": "run_action", "args": {"id": "open_preferences"}, "await": false, "save_as": "prefs"},
    {"tool": "dialog_respond", "args": {"dialog": {"title": "Preferences"}, "button": "close"}},
    {"tool": "wait", "args": {"for": "job", "job": "${jobs.prefs}"}},
    {"tool": "screenshot", "args": {"target": "canvas", "save_path": "${out_dir}/after.png", "overwrite": true}}
  ]}
```

- **Run it:** `run_script {"path":"/abs/x.json"}` for a file, or `run_script {"script":{…}}` inline (the
  object, or its JSON text with comments as a file holds it). `"vars":{…}` overrides the script's;
  `"out_dir":"/abs/dir"` sets where `result.json` and screenshots go (default `<name>.out` beside a
  file, `<datadir>/mcp/out/script-<time>` for an inline one).
- **Placeholders:** `${script_dir}`, `${out_dir}`, `${env:NAME}`, `${vars.NAME}`, `${jobs.NAME}`, in any
  string; a string that is just one placeholder becomes the value itself (`"${vars.walls}"` is the
  number 2). A `run_python` step's `code` is never substituted.
- **Steps:** `tool`, `args`, `expect`, `optional`, `await`, `save_as`, `timeout_s`. A step that returns
  a job is awaited (up to `timeout_s`, default 120) before its `expect`. A step that opens a dialog:
  `"await": false, "save_as": "x"`, then the steps that answer it, then `wait` job `${jobs.x}`.
- **expect:** a JSON pointer into the step's reply → matchers `eq ne gt ge lt le contains regex exists
  empty`, several per pointer. `expect` values are not substituted: check a var through the step's
  `args` instead.
- **Failure:** the first failed step or expectation stops the run unless the step has `"optional":
  true` (use it for a call you expect to be refused, followed by a check that nothing changed). A step
  naming an unknown tool, a field of the wrong type or broken JSON makes the script invalid before
  anything runs.
- **Scope:** `dialog_auto_answer` rules a script registers, and a recording it starts, end with it.
  A script's `wait {"for":"idle"}` doesn't wait for the script itself.

The reply, also written to `<out_dir>/result.json`:
`{passed, name, failed_step, steps:[{index, tool, args, ok, duration_ms, result | error, expect_failure, reason}], out_dir, result_file}`.
A failed step's `reason` says why: `tool` (the tool's own error), `expect {pointer, op, expected,
actual}`, `timeout`, `cancelled`, `placeholder`, `missing_job`. `run_script` itself returns a job when
the run outlasts the reply budget: `wait` for it.

## Recipes

- Many objects, one call: [ref]

  ```json
  {"orca_script": 1, "name": "cubes", "steps": [
    {"tool": "new_project", "args": {"discard_changes": true}},
    {"tool": "run_action", "args": {"id": "add_primitive_cube", "param": "20 20 10"}},
    {"tool": "select", "args": {"none": true}},
    {"tool": "run_action", "args": {"id": "add_primitive_cylinder", "param": "10 15"}},
    {"tool": "arrange", "args": {}},
    {"tool": "list_objects", "expect": {"/total": {"eq": 2}}}
  ]}
  ```

- From Python, where the steps are computed:
  `orca.host.tools.run_script({"orca_script": 1, "name": "x", "steps": steps})` returns the same reply.
- Keep what you did as a file: `export_script {"path":"/abs/session.json"}` (`"clear":true` at the
  start drops exploration; `"overwrite":true` replaces a file). It holds the calls that changed
  something, plus `wait` and `screenshot`, in order, with refs replaced by locators and screenshot paths
  moved under `${out_dir}`. Failed calls are left out, including a job that failed later; a call whose
  job still runs is left out and named in `left_out_running` (wait, then export again). Reads
  (`get_state`, `list_objects`) are not recorded: add `expect`s by hand.

## Traps

- An auto-answer rule's regex is searched in the title, the message and the dialog's own text, so a
  bare word answers more dialogs than meant: `"unsaved"` also matches Preferences ("Unsaved projects")
  and closes it the moment it opens. Match a phrase (`"unsaved changes"`) or the title (`"^Preferences$"`). [ref]
- A JSON script's own `run_script` step gives its nested script as an object: a text one has the
  outer placeholders spliced in as text, and a value with a quote or a backslash (a Windows path) breaks
  its JSON.
- A replay runs against the project as it is then: start the script with `new_project` or `open_files`
  when it depends on the scene.
- Native file pickers can't be answered by a step: use the tool or the Speed Dial `path` param that
  skips the picker, or a `dialog_auto_answer` rule to cancel it.

## The person's Run Script dialog

File > Run Script… (Speed Dial `open_script_runner`) is where a person opens, edits and runs a JSON
script or a `.py` file. It is hidden from the GUI tools; agents use `run_script` and `run_python`, and
hand the person a file to open there when they want to run it themselves.
