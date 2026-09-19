# herdr plugin

A Grok Bot / Cowork-format plugin that lets a bot manage terminal coding agents —
devin, claude, codex, pi, cursor, and anything else Herdr detects — through
[Herdr](https://herdr.dev), the terminal workspace manager for AI agents.

## What the bot gets

- The `herdr` skill: the operating manual for the agent fleet. Survey live agents, spawn new
  ones in a target directory, send tasks, wait on status, unblock stuck agents, and collect
  results. Works over the `herdr` CLI alone.
- The `herdr` MCP server (bundled at `servers/herdr-mcp.cjs`): 13 structured tools over Herdr's
  socket API — `list_agents`, `start_agent`, `send_to_agent`, `read_pane`,
  `wait_for_agent_status`, `wait_for_output`, `send_keys`, `get_layout`, `edit_layout`, and
  tab/workspace helpers. Declared with `placement: "client"` so it runs next to the Herdr
  socket on this machine.

## Prerequisites

- Herdr installed and its server running (`herdr agent list` works). The skill was verified
  against herdr 0.7.3 — the CLI surface changed in 0.7.5, and the skill tells the bot to fall
  back to `herdr <command> --help` when a command errors.
- Node.js ≥ 20 for the MCP server.
- The agent CLIs you intend to drive on PATH (`devin`, `claude`, `codex`, `pi`, ...).

## Install

Package and install as a `.plugin` file:

```sh
cd herdr-plugin
zip -r /tmp/herdr.plugin . -x "*.DS_Store" ".git/*"
```

Then hand `herdr.plugin` to the bot (attach it in chat or place it where plugin files are
accepted) and approve the install.

The repo root also carries `.cursor-plugin/marketplace.json`, so the directory works as a
marketplace checkout if you install via a marketplace source instead.

To install through the Grok plugin ecosystem: push this repo to GitHub, then add it as a plugin
source in Grok Bot or run `grok plugin install <name> --trust` inside Grok Build. Installs pin
to a commit SHA. To share it publicly, open a PR adding it to `xai-org/plugin-marketplace` —
the catalog is open.

## Layout

```
.claude-plugin/plugin.json   plugin manifest
.cursor-plugin/marketplace.json  marketplace manifest (same plugin, repo-based install)
.mcp.json                    herdr MCP server registration
servers/herdr-mcp.cjs        bundled herdr-mcp (source: github.com/islam3zzat/herdr-mcp)
skills/herdr/SKILL.md        operating manual — surfaces, spawn recipes, manage loop, rules
skills/herdr/references/cli.md  full herdr CLI reference
```

## Rebuilding the bundled server

From a checkout of herdr-mcp:

```sh
npx esbuild src/index.ts --bundle --platform=node --format=cjs \
  --target=node20 --outfile=/path/to/herdr-plugin/servers/herdr-mcp.cjs
```
