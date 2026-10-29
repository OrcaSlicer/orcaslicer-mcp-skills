# OrcaSlicer MCP skills

Agent skills for [OrcaSlicer](https://github.com/OrcaSlicer/OrcaSlicer), the open-source 3D printing
slicer. With OrcaSlicer's MCP server connected, an AI agent such as Claude Code, Codex or opencode
can load, place and arrange models, change print, filament and printer settings and presets, add
supports, modifiers and multi-colour painting, use the cut, text and other gizmos, run calibration
tests, model parts in the Design (CAD) tab, slice, and read or export the results. The skill teaches the agent the
tool calls that work, the settings whose meaning their label hides, and the traps that otherwise cost
retries, so tasks finish in fewer calls and with fewer screenshots.

It also covers developing OrcaSlicer itself: launching a build with MCP, reproducing an issue, and
recording replayable test scripts.

## Before you start

Open OrcaSlicer, turn on **Preferences > General > AI features > MCP server**, and use **Add to an
agent** to copy the setup for your agent. The skill does nothing until the agent is connected to a
running OrcaSlicer.

## Install

### Claude Code

```bash
claude plugin marketplace add OrcaSlicer/orcaslicer-mcp-skills
```

```bash
claude plugin install orcaslicer-mcp-skills@orcaslicer
```

Update later with `claude plugin update orcaslicer-mcp-skills@orcaslicer`.

### Codex

```bash
codex plugin marketplace add OrcaSlicer/orcaslicer-mcp-skills
```

```bash
codex plugin add orcaslicer-mcp-skills@orcaslicer
```

Update later with `codex plugin marketplace upgrade orcaslicer`, then run the `codex plugin add`
command again.

### opencode

opencode has no plugin marketplace, so it takes the skill as a folder. Clone this repository and copy
the skill into `~/.config/opencode/skills`:

```bash
git clone https://github.com/OrcaSlicer/orcaslicer-mcp-skills.git
```

```bash
mkdir -p ~/.config/opencode/skills && cp -R orcaslicer-mcp-skills/skills/orcaslicer-mcp-skills ~/.config/opencode/skills/
```

On Windows, in PowerShell:

```powershell
New-Item -ItemType Directory -Force "$HOME\.config\opencode\skills" | Out-Null; Copy-Item -Recurse orcaslicer-mcp-skills\skills\orcaslicer-mcp-skills "$HOME\.config\opencode\skills\"
```

opencode also reads `~/.claude/skills` and `~/.agents/skills`, so a manual install there (below)
already reaches it; keep the skill in one of these folders only. A Claude Code or Codex plugin
install doesn't reach opencode. Restart opencode after installing; to update, `git pull` in the
clone and copy the folder again.

### Manual install

Copy the skill folder into your agent's skills folder: `~/.claude/skills` for Claude Code,
`~/.agents/skills` for Codex and other agents that read it. On macOS and Linux:

```bash
git clone https://github.com/OrcaSlicer/orcaslicer-mcp-skills.git
```

```bash
mkdir -p ~/.claude/skills && cp -R orcaslicer-mcp-skills/skills/orcaslicer-mcp-skills ~/.claude/skills/
```

```bash
mkdir -p ~/.agents/skills && cp -R orcaslicer-mcp-skills/skills/orcaslicer-mcp-skills ~/.agents/skills/
```

On Windows, in PowerShell:

```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null; Copy-Item -Recurse orcaslicer-mcp-skills\skills\orcaslicer-mcp-skills "$HOME\.claude\skills\"
```

To update, `git pull` in the clone and copy the folder again. Start a new agent session after
installing or updating.

## Use

Ask for what you want done in OrcaSlicer, for example "load benchy.stl, scale it to 150 %, slice it
with 0.16 mm layers and tell me the print time and filament weight". The agent loads the skill when
OrcaSlicer's MCP tools are available or the request is about OrcaSlicer.

## What it runs

The plugin is instructions only: one skill with its reference files. It installs no MCP server, runs
no code of its own and sends nothing anywhere. The agent works through the OrcaSlicer MCP connection
you set up, which accepts connections from your own computer only. See the
[privacy policy](PRIVACY.md).

## Releasing

Raise `version` in `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` with every release;
Claude Code and Codex installs update only when it changes. The Codex manifest also carries the
OpenAI plugin directory listing and marks the plugin desktop-only, since it needs OrcaSlicer running
on the same computer. Check the plugin with `claude plugin validate --strict .` before pushing.

## License

[AGPL-3.0](LICENSE), the same as OrcaSlicer.
