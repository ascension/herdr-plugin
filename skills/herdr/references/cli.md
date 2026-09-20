# Herdr CLI reference

The `herdr` binary speaks to the running Herdr server over `~/.config/herdr/herdr.sock`.
List/read commands print JSON to stdout. Verified against herdr 0.9.1 — run
`herdr <command> --help` when in doubt; the installed help is authoritative.

## Agents

```sh
herdr agent list                                   # all agents: pane_id, kind, status, cwd, name
herdr agent get <target>                           # one agent's record
herdr agent read <target> [--source visible|recent|recent-unwrapped|detection] \
                 [--lines N] [--format text|ansi]
herdr agent prompt <target> <text>                 # atomic text + Enter; refuses blocked agents
                 [--wait] [--until <status>] [--timeout <ms>]
herdr agent start <name> --kind <kind> --pane <id> [--timeout <ms>] [-- <agent_arg>...]
herdr agent wait <target> [--until <status>]... [--timeout <ms>]
herdr agent send-keys <target> <key...>            # enter, escape, up, down, c-c, ...
herdr agent rename <target> <name>|--clear
herdr agent focus <target>
herdr agent explain <target> [--json]              # why herdr classifies the pane the way it does
herdr agent attach <target> [--takeover]           # interactive takeover — for a human, not automation
```

`agent start` requires an existing pane at an interactive shell prompt — split one with
`pane split` first. `--kind` is a fixed enum (`--help` lists it: pi, claude, codex, devin,
cursor, gemini, grok, muse, amp, opencode, …) and `NAME` must be unique across the fleet.
`--timeout` (default 30000, max 300000) waits for the pane's shell and the agent's interactive
readiness, so success means the agent is promptable.

`--source detection` reads the snapshot herdr's status classifier sees — the fastest way to
check why a pane reports its current status.

## Panes

```sh
herdr pane list [--workspace <id>]
herdr pane current
herdr pane get <pane_id>
herdr pane layout [--pane ID|--current]
herdr pane process-info [--pane ID|--current]
herdr pane read <pane_id> [--source ...] [--lines N] [--format text|ansi]
herdr pane send-text <pane_id> <text>              # literal text, no Enter
herdr pane send-keys <pane_id> <key> [key ...]
herdr pane wait-output <pane_id> --match <text>|--regex <pat> [--source ...] [--timeout MS]
herdr pane run <pane_id> <cmd>                     # run a shell command in a shell pane
herdr pane split [<pane_id>|--pane ID|--current] --direction right|down \
               [--ratio F] [--cwd PATH] [--env K=V] [--focus|--no-focus]
herdr pane move <pane_id> --tab <tab_id> --split right|down | --new-tab | --new-workspace
herdr pane rename <pane_id> <label>|--clear
herdr pane zoom [<pane_id>] [--toggle|--on|--off]
herdr pane close <pane_id>                         # destructive
herdr pane report-agent <pane_id> --source ID --agent LABEL --state ...   # self-reporting (in-pane agents)
herdr pane release-agent <pane_id> --source ID --agent LABEL
```

`pane report-agent` lets a non-TUI process self-report agent status so herdr tracks it like a
first-class agent — it is for processes running inside the pane, not for external operators.

## Waits

`agent wait` and `agent prompt --wait` take repeated `--until <status>` flags over
`idle | working | blocked | done | unknown`. `pane wait-output` matches a substring or regex on
pane output. A timeout returns a report, not an error — re-check the pane instead of assuming
failure, and read the pane before retrying a prompt (a stalled send may already have landed).

## Layout and state

```sh
herdr api snapshot                                 # full runtime state dump (JSON)
herdr api schema [--json]                          # socket protocol schemas
herdr status                                       # server version, protocol, socket path
herdr workspace <subcommand>                       # create / rename / focus / close / list
herdr tab <subcommand>                             # create / rename / focus / close (destructive)
herdr worktree <subcommand>                        # git worktree helpers
herdr notification <subcommand>
herdr integration <subcommand>
herdr session <subcommand>
herdr --skill                                      # the skill for agents running INSIDE panes
```

## Gotchas

- Agents on the alternate screen (most TUIs) do not grow host scrollback — `--source recent`
  can be thin; use `--source detection` or `visible` for state, `recent` for transcripts.
- CLI errors print JSON to stderr (exit 1 for server errors, exit 2 for syntax).
- `agent prompt` refuses agents already `blocked` on input — read the pane and answer its
  pending question first.
- `agent start` does not create panes and does not take `--cwd` — cwd belongs to `pane split`.
- Agent names must be unique; `agent rename` is how you re-tag a pane.
- Old spelling, pre-0.9: `agent send` (literal text only), `agent start <kind> --cwd DIR --
  <argv>`, `agent wait --status`, `herdr wait output`. All gone on 0.9.x.

## MCP equivalent

When the plugin's `herdr` MCP server is connected, these map to `list_agents`, `start_agent`
(splits the pane itself; takes `kind`, `name`, `args`, `cwd`, `pane_id`),
`send_to_agent` (`submit: true` → `agent.prompt`, optional `wait_seconds`), `send_keys`,
`read_pane`, `wait_for_agent_status` (status list, includes `unknown`), `wait_for_output`,
`get_layout`, and `edit_layout`, plus tab/workspace/worktree/notification helpers and the
`herdr_rpc` escape hatch — 13 tools total.
