# Replays of tool calls

A replay is a JSON list of tool calls that `run_replay` runs in order, checking each reply. Use one to
send a known sequence in one round trip, to answer the dialog a step opens without racing it, to check
results with `expect`, or to keep a session as a file to run again (`export_replay`). Logic over the
project's data (loops, conditions, math) is a Python script instead (`references/python.md`), which can
run a replay too.

## Format

```jsonc
{ "orca_replay": 1, "name": "two walls, sliced",
  "vars": {"model": "${replay_dir}/part.3mf", "walls": 2},          // ${vars.model} below
  "steps": [
    {"tool": "auto_answer_dialogs", "args": {"rules": [{"match": "unsaved changes", "button": "no"}]}},
    {"tool": "open_files", "args": {"paths": ["${vars.model}"], "discard_changes": true}},
    {"tool": "set_settings", "args": {"scope": "process", "values": {"wall_loops": "${vars.walls}"}}},
    {"tool": "slice", "expect": {"/plates/0/outcome": {"eq": "ok"}}},
    {"tool": "run_action", "args": {"id": "open_preferences"}, "await": false, "save_as": "prefs"},
    {"tool": "answer_dialog", "args": {"dialog": {"title": "Preferences"}, "button": "close"}},
    {"tool": "wait", "args": {"for": "job", "job": "${jobs.prefs}"}},
    {"tool": "screenshot", "args": {"target": "canvas", "save_path": "${out_dir}/after.png", "overwrite": true}}
  ]}
```

- **Run it:** `run_replay {"path":"/abs/x.json"}` for a file, or `run_replay {"replay":{…}}` inline (the
  object, or its JSON text with comments as a file holds it). `"vars":{…}` overrides the replay's;
  `"out_dir":"/abs/dir"` sets where `result.json` and screenshots go (default `<name>.out` beside a
  file, `<datadir>/mcp/out/replay-<time>` for an inline one).
- **Placeholders:** `${replay_dir}`, `${out_dir}`, `${env:NAME}`, `${vars.NAME}`, `${jobs.NAME}`, in any
  string; a string that is just one placeholder becomes the value itself (`"${vars.walls}"` is the
  number 2). A `run_script` step's `code` is never substituted.
- **Steps:** `tool`, `args`, `expect`, `optional`, `await`, `save_as`, `timeout_s`. A step that returns
  a job is awaited (up to `timeout_s`, default 120) before its `expect`. A step that opens a dialog:
  `"await": false, "save_as": "x"`, then the steps that answer it, then `wait` job `${jobs.x}`.
- **expect:** a JSON pointer into the step's reply → matchers `eq ne gt ge lt le contains regex exists
  empty`, several per pointer. `expect` values are not substituted: check a var through the step's
  `args` instead.
- **Failure:** the first failed step or expectation stops the run unless the step has `"optional":
  true` (use it for a call you expect to be refused, followed by a check that nothing changed). A step
  naming an unknown tool, a field of the wrong type or broken JSON makes the replay invalid before
  anything runs.
- **Scope:** `auto_answer_dialogs` rules a replay registers, and a recording it starts, end with it.
  A replay's `wait {"for":"idle"}` doesn't wait for the replay itself.

The reply, also written to `<out_dir>/result.json`:
`{passed, name, failed_step, steps:[{index, tool, args, ok, duration_ms, result | error, expect_failure, reason}], out_dir, result_file}`.
A failed step's `reason.kind` says why: `tool` (the tool's own error), `expect` (with `pointer`, `op`,
`expected`, `actual`), `expect_invalid`, `python` (a `run_script` step that raised, was cancelled or
timed out), `timeout`, `cancelled`, `placeholder`, `missing_job`. `run_replay` itself returns a job when
the run outlasts the reply budget: `wait` for it.

## Recipes

- Many objects, one call: [ref]

  ```json
  {"orca_replay": 1, "name": "cubes", "steps": [
    {"tool": "new_project", "args": {"discard_changes": true}},
    {"tool": "run_action", "args": {"id": "add_primitive_cube", "param": "20 20 10"}},
    {"tool": "select_objects", "args": {"none": true}},
    {"tool": "run_action", "args": {"id": "add_primitive_cylinder", "param": "10 15"}},
    {"tool": "arrange", "args": {}},
    {"tool": "list_objects", "expect": {"/total": {"eq": 2}}}
  ]}
  ```

- From Python, where the steps are computed:
  `orca.host.tools.run_replay({"orca_replay": 1, "name": "x", "steps": steps})` returns the same reply.
- Keep what you did as a file: `export_replay {"path":"/abs/session.json"}` (`"clear":true` at the
  start drops exploration; `"overwrite":true` replaces a file). It holds the calls that changed
  something, plus `wait` and `screenshot`, in order, with refs replaced by locators and screenshot paths
  moved under `${out_dir}`. Failed calls are left out, including a job that failed later; a call whose
  job still runs is left out and named in `left_out_running` (wait, then export again). Reads
  (`get_app_state`, `list_objects`) are not recorded: add `expect`s by hand.

## Traps

- An auto-answer rule's regex is searched in the title, the message and the dialog's own text, so a
  bare word answers more dialogs than meant: `"unsaved"` also matches Preferences ("Unsaved projects")
  and closes it the moment it opens. Match a phrase (`"unsaved changes"`) or the title (`"^Preferences$"`). [ref]
- A replay's own `run_replay` step gives its nested replay as an object: a text one has the
  outer placeholders spliced in as text, and a value with a quote or a backslash (a Windows path) breaks
  its JSON.
- A replay runs against the project as it is then: start it with `new_project` or `open_files`
  when it depends on the scene.
- Native file pickers can't be answered by a step: use the tool or the Speed Dial `path` param that
  skips the picker, or an `auto_answer_dialogs` rule registered before the step that opens it, which
  hands it a `path` or cancels it.

## The person's Action Replay window

File > Action Replay… (Speed Dial `open_action_replay`) is where a person opens a replay file, reads
its steps and runs them; Python scripts have their own Run Script panel (Speed Dial
`toggle_run_script`). Both are hidden from the GUI tools; agents use `run_replay` and `run_script`, and
hand the person a file to open there when they want to run it themselves.
