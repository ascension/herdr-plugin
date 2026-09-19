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
An `agent_id` and a `pane_id` are the same handle (for example `w14:p3`).

## Control surfaces

Two equivalent surfaces. Prefer the `herdr` MCP tools when the plugin's MCP server is connected;
the `herdr` CLI does the same things and prints JSON.

| Operation | MCP tool | CLI |
|---|---|---|
| List agents and status | `list_agents` | `herdr agent list` |
| Spawn an agent | `start_agent` | `herdr agent start <kind> -- <argv...>` |
| Send a prompt | `send_to_agent` with `submit: true` | `herdr agent send <target> <text>` then `herdr pane send-keys <pane> enter` |
| Read output | `read_pane` | `herdr agent read <target> --lines N` |
| Wait for a status | `wait_for_agent_status` | `herdr agent wait <target> --status <s> --timeout MS` |
| Wait for text | `wait_for_output` | `herdr wait output <pane> --match <text>` |
| Send key presses | `send_keys` | `herdr pane send-keys <pane> <key...>` |
| Workspaces / tabs / panes | `get_layout`, `edit_layout` | `herdr pane list`, `herdr pane layout`, `herdr pane move` |

CLI targets accept pane ids, agent names, and detected agent labels. Field names differ between
surfaces: CLI `agent list` returns `pane_id`, `agent_status`, `name`, `cwd`, `agent_session`;
MCP `list_agents` returns `agent_id`, `status`, `cwd`, `focused`. MCP `wait_for_agent_status`
accepts a list of statuses (`["blocked", "done", "idle"]`) but cannot wait for `unknown`.

The full CLI surface is in [references/cli.md](references/cli.md).

## Version drift

This skill was verified against herdr 0.7.3. The CLI changed in 0.7.5: `agent send` became
`agent prompt`, `agent start` requires `--kind <kind> --pane <id>` on an existing shell pane,
`agent wait --status` became `--until`, and `herdr wait` became `pane wait-output`. If a command
errors, run `herdr <command> --help` — the installed help output is authoritative over this file.

## Spawn recipes

`herdr agent start <kind> --cwd <dir> -- <argv...>` opens the agent in a new pane split from the
focused pane. `kind` is the agent label Herdr reports in `agent list` — common kinds are `claude`,
`devin`, `codex`, `pi`, `cursor`. Rename right after spawn so the fleet stays legible:

```sh
herdr agent start claude --cwd ~/Projects/app -- claude
herdr agent rename <target> fix-login-bug
```

| Agent | argv | Notes |
|---|---|---|
| claude | `claude` | Add `--dangerously-skip-permissions` for unattended runs. |
| codex | `codex` | Add `--dangerously-bypass-approvals-and-sandbox` for unattended runs. |
| devin | `devin` | Interactive TUI. Hand it the task with send. |
| pi | `pi` | Interactive. `pi -p "<prompt>"` runs one-shot headless instead. |

An agent started without bypass flags pauses on permission prompts, which surfaces as `blocked`.
Answer the prompt with send or keys. Use bypass flags only when the operator asked for unattended
agents — they let the agent run any command in its cwd, so spawn in the intended directory only.

## Manage loop

1. Survey. `herdr agent list` — note each agent's `pane_id`, `agent` kind, `agent_status`, `cwd`,
   and `name`.
2. Spawn if needed, then `herdr agent wait <target> --status idle --timeout 60000` so the TUI is
   ready before the first prompt.
3. Task. `herdr agent send <target> "<prompt>"`. CLI `agent send` writes literal text; follow with
   `herdr pane send-keys <pane> enter` if the agent does not react. With MCP, call
   `send_to_agent` with `submit: true`.
4. Monitor. `herdr agent wait <target> --status <s>` takes one status per call — poll `agent list`
   or loop the wait for the status you expect. MCP `wait_for_agent_status` accepts a list and is
   the better primitive here.
5. On `blocked`: `herdr agent read <target> --lines 60`, decide what it needs, send the answer —
   text plus enter for prompts, `send-keys` (`up`/`down`/`enter`/`escape`) for menus.
6. On `idle` or `done`: `herdr agent read <target> --source recent` to collect the result.

When `agent list` reports `agent_session`, record it. It is the resumable session handle for that
agent kind and appears only when the integration reports one — pass it back to the agent's own
resume flag (`claude --resume <id>`, `pi --session <path>`) to continue the same session later.

## Rules

- Always `agent read` before `agent send`. Confirm the pane holds the agent you intend — sends
  act on live sessions.
- Drive only agents you spawned or were explicitly handed. Leave `working` agents alone unless
  asked to intervene.
- `blocked` means it needs input. `idle` means it is ready for work. `working` means wait.
- Check the `agent` field before sending — never send agent input into a shell pane.
- `herdr pane close`, `herdr tab close`, and `edit_layout` close/move are destructive. Do not use
  them to clean up unless told.
