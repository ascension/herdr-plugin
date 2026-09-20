---
name: herdr
description: >
  Manage coding agents running in Herdr workspaces — devin, claude, codex, pi, cursor, and others.
  Use when asked to start, task, monitor, unblock, or collect results from terminal coding agents,
  survey a fleet of agents, or hand work to a specific agent. Trigger phrases include "herdr",
  "spin up an agent", "check on my agents", "the codex agent is stuck", "run this in claude",
  "collect what the devin agent finished".
---

# Herdr agent control

Herdr is a terminal workspace manager for AI coding agents. Every agent runs in a pane inside a
workspace → tab → pane tree and reports a live status: `working`, `blocked`, `idle`, `done`, or
`unknown` (present but unclassifiable — not proof of completion).
An `agent_id` and a `pane_id` are the same handle (for example `w14:p3`); agents also carry a
unique display `name` set at start or via `agent rename`.

## Control surfaces

Two equivalent surfaces. Prefer the `herdr` MCP tools when the plugin's MCP server is connected;
the `herdr` CLI does the same things and prints JSON.

| Operation | MCP tool | CLI |
|---|---|---|
| List agents and status | `list_agents` | `herdr agent list` |
| Spawn an agent | `start_agent` (splits a pane itself) | `herdr pane split` then `herdr agent start <name> --kind <kind> --pane <id>` |
| Send a prompt | `send_to_agent` with `submit: true` | `herdr agent prompt <target> <text>` |
| Read output | `read_pane` | `herdr agent read <target> --lines N` |
| Wait for a status | `wait_for_agent_status` | `herdr agent wait <target> --until <s>` |
| Wait for text | `wait_for_output` | `herdr pane wait-output <pane> --match <text>` |
| Send key presses | `send_keys` | `herdr agent send-keys <target> <key...>` |
| Workspaces / tabs / panes | `get_layout`, `edit_layout` | `herdr pane list`, `herdr pane layout`, `herdr pane move` |

CLI targets accept pane ids, agent names, and detected agent kinds. Field names differ between
surfaces: CLI `agent list` returns `pane_id`, `agent_status`, `name`, `cwd`; MCP `list_agents`
returns `agent_id`, `status`, `name`, `cwd`, `focused`. `send_to_agent` submit is atomic
(text + Enter, refuses `blocked` agents) and accepts `wait_seconds`; the CLI equivalent is
`agent prompt` with `--wait --until <status>`.

The full CLI surface is in [references/cli.md](references/cli.md).

## Version drift

This skill was verified against herdr 0.9.1 (socket protocol 22). On 0.7.x the CLI looked
different: `agent send` (literal text, needed a `pane send-keys enter` follow-up) instead of
`agent prompt`, `agent start <kind> --cwd <dir> -- <argv...>` instead of `--kind --pane`,
`agent wait --status` instead of `--until`, and `herdr wait output` instead of
`pane wait-output`. If a command errors, run `herdr <command> --help` — the installed help
output is authoritative over this file.

## Spawn recipes

`agent start` launches a registered agent kind into an existing shell pane — it does not create
the pane. Split one first, then start:

```sh
herdr pane split --current --direction right --cwd ~/Projects/app --no-focus   # prints the new pane_id
herdr agent start fix-login --kind claude --pane <pane_id> -- --dangerously-skip-permissions
```

`--kind` is a fixed enum (`pi`, `claude`, `codex`, `devin`, `cursor`, `gemini`, `grok`, `muse`,
`amp`, `opencode`, … — `herdr agent start --help` lists the live set) and `NAME` must be unique.
Args after `--` go to the agent executable. `--timeout` (default 30s) waits for interactive
readiness, so a successful `start` means the agent TUI is actually up.

MCP `start_agent` does the split itself when `pane_id` is omitted: pass `kind`, `cwd`, and
`direction`; give it `name` and `args` as needed.

| Agent | Extra args | Notes |
|---|---|---|
| claude | `--dangerously-skip-permissions` | Needed for unattended runs. |
| codex | `--dangerously-bypass-approvals-and-sandbox` | Needed for unattended runs. |
| devin | — | Interactive TUI. Task it with `agent prompt`. |
| pi | — | Interactive. For one-shot headless work use `herdr pane run <pane> "pi -p '<prompt>'"` instead. |

An agent started without bypass flags pauses on permission prompts, which surfaces as `blocked`.
Use bypass flags only when the operator asked for unattended agents — they let the agent run any
command in its cwd, so spawn in the intended directory only.

## Manage loop

1. Survey. `herdr agent list` — note each agent's `pane_id`, `agent` kind, `agent_status`,
   `cwd`, and `name`.
2. Spawn if needed (split + start above). `agent start` already waits for interactive
   readiness, so the returned agent is promptable immediately.
3. Task. `herdr agent prompt <target> "<prompt>"` delivers text + Enter atomically and refuses
   a `blocked` agent. Add `--wait --until idle --timeout 120000` to block until it settles;
   MCP `send_to_agent` takes `submit: true` plus optional `wait_seconds`.
4. Monitor. `herdr agent wait <target> --until <s>` accepts repeated `--until` flags; MCP
   `wait_for_agent_status` accepts a list (including `unknown`) and is the better primitive.
5. On `blocked`: `herdr agent read <target> --lines 60`, decide what it needs, send the answer —
   `agent prompt` for text, `agent send-keys` (`up`/`down`/`enter`/`escape`) for menus.
6. On `idle` or `done`: `herdr agent read <target> --source recent` to collect the result.

## Rules

- Always `agent read` before `agent prompt`. Confirm the pane holds the agent you intend —
  prompts act on live sessions.
- Drive only agents you spawned or were explicitly handed. Leave `working` agents alone unless
  asked to intervene.
- `blocked` means it needs input. `idle` means it is ready for work. `working` means wait.
  `unknown` is unclassified, not finished.
- Check the `agent` field before prompting — never send agent input into a shell pane.
- `herdr pane close`, `herdr tab close`, and `edit_layout` close/move are destructive. Do not
  use them to clean up unless told.
- This skill drives Herdr from outside. Agents running inside a Herdr pane get the built-in
  `herdr --skill` instead — it assumes `HERDR_ENV=1` and covers self-reporting, not fleet
  control.
