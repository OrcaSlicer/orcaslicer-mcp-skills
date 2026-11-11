# Plugins

OrcaSlicer plugins are Python packages that add script commands, main-window pages, slicing and G-code
post-processing steps, texture packs and printer connections. Through MCP an agent runs what an installed
plugin offers, and can write, install, reload and debug a plugin for the person. A one-off computation is
a Python script (`references/python.md`), not a plugin: write a plugin when the person wants a command,
tab or post-processing step that stays in the app.

## Using installed plugins

- **Find what a plugin offers:** `search_actions {"query":"<plugin or command name>"}` lists its script
  commands (`kind: "plugin"`, id `plugin_script_action:<capability>:<plugin key>`) and pages
  (`plugin_page_action:<capability>:<plugin key>`). `search_tools {"query":"<words>"}` also lists the
  operations plugins offer agents, `plugin.<plugin key>.<name>` with their schema; `search_tools {}` lists
  every name. Neither kind is in the tool list.
- **Run a script command:** `run_action {"id":"<capability name>"}` (or the full id) →
  `{"message":"Script plugin started."}`. It runs on its own; its result arrives as a notification:
  find its message in `get_app_state.notifications` (in no fixed order; an error one has `level: "error"`
  and the traceback). [ref]
- **Call an operation:** `call_tool {"tool":"plugin.<key>.<name>","args":{…}}` → the handler's JSON
  reply. It runs with the plugin's permissions, as a job when it takes long. [ref]
- **Open a page:** `run_action {"id":"plugin_page_action:<capability>:<key>"}` shows the plugin's tab.
- **Drive its windows, dock panels and pages** like any web page: `list_widgets {"query":"…"}` lists their
  elements; `click {"target":{"web":"Go","within":"<window title>"}}`,
  `set_widget_value {"target":{"css":"#name","within":"<window title>"},"value":"…"}`; `run_javascript` runs
  JavaScript in one. Dock panels show only on the Prepare and Preview tabs.

## The Plugins dialog

`run_action {"id":"open_plugins"}` opens it, modeless, as the web dialog "Plugins". Its rows are the
installed plugins with an Activate checkbox, version, source (Local, Subscribed, Mine) and status:

| Status | Means |
|---|---|
| Activated | loaded |
| Inactive | installed, not loaded: tick it |
| Error | failed to load: see Debugging below |
| RuntimeError | loaded, but one capability failed and is off |

- Activate or deactivate one: `click {"target":{"css":"label.plugin-checkbox:has(input[data-plugin-key=\"<key>\"])","within":"Plugins"}}`;
  the reply's `checked` and `page.added` (`"Activated …"`, `"Failed to activate …"`) say what happened. [ref]
  `list_widgets` finds a row by its name, not its key, and may leave out rows scrolled away; the locator
  needs neither.
- `"Expand <name> capabilities"` shows a plugin's capabilities with their own checkboxes; the details
  below have tabs Plugin Info, Description, Config (each capability's JSON configuration), Changelog,
  Diagnostics (the load error) and Permissions.
- `run_action {"id":"refresh_plugins"}` rescans the plugin folders (notification "Plugins refreshed."),
  and an open Plugins dialog shows the result. A Local row's version is its file's as of the last
  refresh or activation, and a Subscribed row's is the release installed from the cloud; the code that
  runs is what the last activation loaded.

## Permission prompts

A plugin asks before it reads or writes files outside OrcaSlicer's folders, connects to the network,
starts a program or a thread, calls the MCP tools, or offers an operation. The prompt is a modal dialog
titled "Plugin permissions" or "Plugin permission request", whose `message` names what it asks
(`get_app_state.dialogs`); `answer_dialog {"button":"yes"|"no"}` answers it on every platform. [ref]

- These answers protect the person's computer and are saved with the plugin (shown on its Permissions
  tab): ask the user before pressing Yes, unless it is your own plugin in a throwaway test data folder.
- No denies that access: the plugin's call raises `PermissionError`, or for a load-time request the
  plugin fails to load (an operation that was refused is just not offered).
- The Permissions tab lists a plugin's saved permissions, where the person adds, removes or grants
  Full access. Reinstalling or updating a plugin clears them.

## Writing a plugin

A single-file plugin is `<datadir>/orca_plugins/<key>/<key>.py`, one entry file per folder. The file's
name without `.py` is the plugin key in action ids and `plugin.<key>.<name>`; keep the folder named the
same, in lower case with underscores. The data folder is the parent of the `mcp` folder in
`get_app_state.mcp.discovery_file`. That is the person's real data folder unless OrcaSlicer was launched
with `--datadir`: say what you are adding before writing there.

```python
# /// script
# dependencies = []                      # pip requirements, installed when the plugin loads
#
# [tool.orcaslicer.plugin]
# name = "Tallest Object"
# description = "Reports the tallest object."
# author = "Your Name"
# version = "1.0.0"
# ///
import orca


def tallest():
    m = orca.host.model()
    objects = [m.object(i) for i in range(m.object_count())]
    if not objects:
        return None
    top = max(objects, key=lambda o: o.bounding_box().size[2])
    return {"name": top.name, "height": top.bounding_box().size[2]}


class ReportTallest(orca.script.ScriptPluginCapabilityBase):
    def get_name(self):                   # the command's name; unique in the plugin, no ";"
        return "Report tallest"

    def execute(self):                    # runs on a worker thread when the person runs it
        t = tallest()
        if t is None:
            return orca.ExecutionResult.skipped("No objects.")
        return orca.ExecutionResult.success(f"{t['name']}: {t['height']:.2f} mm")


@orca.plugin                              # exactly one package class per plugin
class TallestObject(orca.base):
    def register_capabilities(self):
        orca.register_capability(ReportTallest)
        # Offered to agents and scripts as plugin.<key>.tallest once the person allows it.
        orca.register_operation("tallest", "The tallest object's name and height in mm.",
                                {"type": "object", "properties": {}}, lambda args: tallest(),
                                changes=False)
```

Capability bases (each needs `get_name()`; look the rest up with `search_python_api`):

| Base | Entry | Runs |
|---|---|---|
| `orca.script.ScriptPluginCapabilityBase` | `execute(self)` → `ExecutionResult` | when the person runs it from the Plugins dialog or the Speed Dial (`run_action`), on a worker thread; it may be `async def` |
| `orca.pages.PagesPluginCapabilityBase` | `get_ui(self)` → HTML; `on_message(self, data)`, `post_message(data)`, `get_icon(self)` | a main-window tab while the capability is on |
| `orca.slicing.SlicingPipelineCapabilityBase` | `execute(self, ctx)` → `ExecutionResult` | at slicing steps (`ctx.step`, `orca.slicing.Step`), on the slicing thread; at `psGCodePostProcess` it edits the file at `ctx.gcode_path` in place (the preview doesn't show the edit) |
| `orca.resource_texture.ResourceTexturePluginCapabilityBase` | `get_texture_dir(self)` | a texture pack; wheel (`.whl`) plugins only |
| `orca.printer_agent.PrinterAgentBase` | the agent methods | a printer connection (experimental) |

- **Results:** `orca.ExecutionResult.success(message)`, `.skipped(message)`,
  `.failure(orca.PluginResult.RecoverableError, message)`; the message is what the person sees.
- **Hooks on any capability:** `on_load()`, `on_unload()`, and `on_lifecycle_event(self, event, ctx)`
  for project, slicing, object, plate, preset and printer events (`search_python_api {"query":"LifecycleEvent"}`
  lists them; slicing events come on a background thread).
- **Configuration:** `get_default_config()` returns a dict; `json.loads(self.get_config())` reads the
  saved one, `self.save_config(json.dumps(cfg))` writes it; the person edits it on the Config tab.
- **Permissions** are declared in `register_capabilities()`:
  `orca.request_permissions(fs_read=[…], fs_readwrite=["/abs/dir/*"], network=["https://host"], process=[…], threading=True, tools=True)`
  shows one prompt at load (No fails the load), except `tools=True`, which asks on its own and
  leaves the plugin loading without it when refused. Declare what the plugin needs there: an
  undeclared access in a script's `execute()` or a thread raises `PermissionError` at once and opens a
  prompt whose Yes lets the next attempt through. [ref]
- **The host API** is `orca.host`, as for scripts (`references/python.md`): model, meshes, presets,
  `orca.host.ui` windows, dock panels, progress dialogs and notifications, `orca.host.plugin.storage()`
  (the plugin's folder). Never import another GUI toolkit (tkinter, Qt, wx).
- **Tool calls from a plugin** (`orca.host.tools.call`) need `tools=True` and come from `execute()` or a
  thread it starts, never from a window callback or the UI thread; `export_replay` is refused to it.
- A wheel plugin (`.whl` as the folder's entry file) takes its name and version from the wheel's
  `METADATA` instead of the PEP 723 block, for plugins with several modules or files.

The plugin developer guide and API pages are in the OrcaSlicer wiki
(https://github.com/OrcaSlicer/OrcaSlicer_WIKI/tree/main/developer_reference/plugin_development). Where
the guide and `search_python_api` differ, `search_python_api` describes the running build.

## Install, run, change, debug

1. Install it from a file anywhere:
   `run_action {"id":"install_local_plugin","param":"<absolute path of the .py or .whl>"}` copies it to
   `<datadir>/orca_plugins/<file name>/` (`orca_plugins/tallest_object.py/tallest_object.py`); later
   edits go to that copy. The reply's message names the plugin, or the call fails saying why, such as a
   `.py` with no `name` in its `[tool.orcaslicer.plugin]` table. Replacing an installed plugin asks
   "Overwrite Plugin" first: the reply is `blocked_by_dialog`; `answer_dialog {"button":"Overwrite"}`,
   then `wait {"for":"job","job":"<job>"}` for the result. [ref] Or write
   `<datadir>/orca_plugins/<key>/<key>.py` yourself and `run_action {"id":"refresh_plugins"}`. Either way
   `run_action {"id":"open_plugins"}` then lists it Inactive. [ref] With no file tools of your own, write
   the file from `run_script` (`os.makedirs`, `open(path, "w")`). [ref]
2. Tick its checkbox (above). Answer the prompts its `register_operation` and `request_permissions`
   calls open (`get_app_state.dialogs`; the tick's reply lists the first). [ref]
3. Run it: `run_action {"id":"Report tallest"}` → read `get_app_state.notifications`;
   `call_tool {"tool":"plugin.tallest_object.tallest","args":{}}`; for a slicing step, slice and export.
   Notifications have no id or time, so a second run with the same message looks like the first:
   check a change through an operation, or put something that changes in the message.
4. **After an edit**, untick and tick it again in the Plugins dialog: a loaded plugin keeps the code it
   loaded with. Saved grants aren't asked again, an operation's included when its description
   changed. [ref] Reinstalling instead leaves the plugin Inactive and asks every permission again.
   A wheel's load reuses the copy extracted when it first loaded, so install a new build of it with
   `install_local_plugin` instead. Don't edit a Subscribed plugin's files: a cloud update replaces them.
5. **Debugging:**
   - Status Error after ticking: `get_log {"level":"error"}` has
     `[load_plugin_impl] FAIL plugin=<key> error=<the exception>`. [ref]
   - The traceback, and everything the plugin prints to `sys.stderr`, is in
     `<datadir>/log/python_*.log` (the newest file). [ref]
   - A command that failed or raised says so in its notification, with the traceback. [ref]
   - `PermissionError`: a prompt was answered No, or the path is one OrcaSlicer never lets plugins
     touch (names with `secret`, `cert` or `conf` in them).
   - `orca_plugins/<key>/.install_state.json` holds the plugin's enabled state and saved permissions;
     the dialog writes it. A plugin folder without one is discovered but not loaded. Python code
     can't read or write it, nor `OrcaSlicer.conf`.

## Not possible

Browsing and subscribing to cloud plugins needs the person signed in to Orca Cloud, in the plugin
browser. `install_local_plugin` without a path, and the Plugins dialog's own Install local plugin, open a
file picker: give the command the path instead.
