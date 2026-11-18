# Replays of tool calls

A replay is a JSON list of tool calls that `replay_run` runs in order, checking each reply. Use one to
send a known sequence in one round trip, to answer the dialog a step opens without racing it, to check
results with `expect`, or to keep a session as a file to run again (`replay_export`). Logic over the
project's data (loops, conditions, math) is a Python script instead (`references/python.md`), which can
run a replay too.

## Format

```jsonc
{ "orca_replay_version": "0.2.0", "name": "two walls, sliced",
  "vars": {"model": "${replay_dir}/part.3mf", "walls": 2},          // ${vars.model} below
  "steps": [
    {"tool": "dialog_auto_answer", "args": {"rules": [{"match": "unsaved changes", "button": "no"}]}},
    {"tool": "project_open", "args": {"path": "${vars.model}", "discard_changes": true}},
    {"tool": "settings_set", "args": {"type": "process", "values": {"wall_loops": "${vars.walls}"}}},
    {"tool": "plate_slice", "expect": {"/plates/0/outcome": {"eq": "ok"}}},
    {"tool": "window_open", "args": {"window": "preferences"}, "await": false, "save_as": "prefs"},
    {"tool": "dialog_answer", "args": {"dialog": {"title": "Preferences"}, "button": "close"}},
    {"tool": "app_wait", "args": {"for": "job", "job": "${jobs.prefs}"}},
    {"tool": "window_screenshot", "args": {"target": "canvas", "save_path": "${out_dir}/after.png", "overwrite": true}}
  ]}
```

- **Run it:** `replay_run {"path":"/abs/x.json"}` for a file, or `replay_run {"replay":{…}}` inline (the
  object, or its JSON text with comments as a file holds it). `"vars":{…}` overrides the replay's;
  `"out_dir":"/abs/dir"` sets where `result.json` and screenshots go (default `<name>.out` beside a
  file, `replay-<time>` in the session's output folder for an inline one: `<datadir>/automation/out`,
  or the `--outputdir` folder of a `--run` launch).
- **Placeholders:** `${replay_dir}`, `${out_dir}`, `${env:NAME}`, `${vars.NAME}`, `${jobs.NAME}`, in any
  string; a string that is just one placeholder becomes the value itself (`"${vars.walls}"` is the
  number 2). A `script_run` step's `code` is never substituted.
- **Steps:** `tool`, `args`, `expect`, `optional`, `await`, `save_as`, `timeout_s`. `tool` names any
  command, also one the tool list doesn't show (a menu item's or a Design command, by its name). A step
  that returns a job is awaited (up to `timeout_s`, default 120) before its `expect`. A step that
  opens a dialog: `"await": false, "save_as": "x"`, then the steps that answer it, then `app_wait` job
  `${jobs.x}`.
- **expect:** a JSON pointer into the step's reply → matchers `eq ne gt ge lt le contains regex exists
  empty`, several per pointer. `expect` values are not substituted: check a var through the step's
  `args` instead.
- **Failure:** the first failed step or expectation stops the run unless the step has `"optional":
  true` (use it for a call you expect to be refused, followed by a check that nothing changed). A step
  naming an unknown tool, a field of the wrong type, broken JSON or an `orca_replay_version` this build does
  not read (it reads `0.2.0`) makes the replay invalid before anything runs.
- **Scope:** `dialog_auto_answer` rules a replay registers, and a recording it starts, end with it.
  A replay's `app_wait {"for":"idle"}` doesn't wait for the replay itself.

The reply, also written to `<out_dir>/result.json`:
`{passed, name, failed_step, steps:[{index, tool, args, ok, duration_ms, result | error, details, expect_failure, reason}], out_dir, result_file}`.
A failed step's `reason.kind` says why: `tool` (the tool's own error: `error` has its text and
`details` a refusal's `code`, which no `expect` can check), `expect` (with `pointer`, `op`,
`expected`, `actual`), `expect_invalid`, `python` (a `script_run` step that raised, was cancelled or
timed out), `timeout`, `cancelled`, `placeholder`, `missing_job`. `replay_run` itself returns a job when
the run outlasts the reply budget: `app_wait` for it.

## Recipes

- Many objects, one call: [ref]

  ```json
  {"orca_replay_version": "0.2.0", "name": "cubes", "steps": [
    {"tool": "project_new", "args": {"discard_changes": true}},
    {"tool": "object_add_primitive", "args": {"shape": "cube", "size": [20, 20, 10]}},
    {"tool": "object_add_primitive", "args": {"shape": "cylinder", "size": [10, 15]}},
    {"tool": "object_arrange", "args": {}},
    {"tool": "object_list", "expect": {"/total": {"eq": 2}}}
  ]}
  ```

- From Python, where the steps are computed:
  `orca.host.tools.run_replay({"orca_replay_version": "0.2.0", "name": "x", "steps": steps})` returns the same reply.
- Keep what you did as a file: `replay_export {"path":"/abs/session.json"}` (`"clear":true` at the
  start drops exploration; `"overwrite":true` replaces a file). It holds the calls that changed
  something, plus `app_wait`, `window_screenshot` and the recording commands, in order, with refs replaced by locators and screenshot paths
  moved under `${out_dir}`. Failed calls are left out, including a job that failed later; a call whose
  job still runs is left out and named in `left_out_running` (wait, then export again). Reads
  (`app_state`, `object_list`) are not recorded: add `expect`s by hand.

## Traps

- An auto-answer rule's regex is searched in the title, the message and the dialog's own text, so a
  bare word answers more dialogs than meant: `"unsaved"` also matches Preferences ("Unsaved projects")
  and closes it the moment it opens. Match a phrase (`"unsaved changes"`) or the title (`"^Preferences$"`). [ref]
- A replay's own `replay_run` step gives its nested replay as an object: a text one has the
  outer placeholders spliced in as text, and a value with a quote or a backslash (a Windows path) breaks
  its JSON.
- A replay runs against the project as it is then: start it with `project_new` or `project_open`
  when it depends on the scene.
- Native file pickers can't be answered by a step: use the tool, or the command's `path` argument that
  skips the picker, or a `dialog_auto_answer` rule registered before the step that opens it, which
  hands it a `path` or cancels it.
- A step whose `args` are exactly a tile's (none, for a tile that has none) is the person's form of its
  command (SKILL.md): it may open the command's dialog or a Design card, start a sketch tool, decline, or act at once on the selection
  (`{"tool":"body_delete"}` deletes the selected body with no prompt; `references/cad.md`, "Two
  forms"). Give such a step its arguments, or `"silent": true` where the schema has `silent` and
  nothing else is required (`body_delete {"silent":true}` is refused: it needs `body`); `replay_export`
  keeps them, so a call that ran silently replays silently.

## The person's Action Replay window

File > Action Replay… (`window_open {"window":"action_replay"}`) is where a person opens a replay file,
reads its steps and runs them; Python scripts have their own Run Script panel
(`script_toggle_panel`). Both are hidden from the GUI tools; agents use `replay_run` and `script_run`, and
hand the person a file to open there when they want to run it themselves.
