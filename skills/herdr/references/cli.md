# Herdr CLI reference

The `herdr` binary speaks to the running Herdr server over `~/.config/herdr/herdr.sock`.
List/read commands print JSON to stdout. Verified against herdr 0.7.3 — run
`herdr <command> --help` when in doubt; the installed help is authoritative (the CLI surface
changed in 0.7.5: `agent send` → `agent prompt`, `agent start` → `--kind --pane`,
`agent wait --status` → `--until`, `herdr wait` → `pane wait-output`).

## Agents

```sh
herdr agent list                                   # all agents: pane_id, agent kind, status, cwd, name, session
herdr agent get <target>                           # one agent's record
herdr agent read <target> [--source visible|recent|recent-unwrapped|detection] \
                 [--lines N] [--format text|ansi] [--ansi]
herdr agent send <target> <text>                   # writes literal text (no implicit Enter)
herdr agent start <kind> [--cwd PATH] [--workspace ID] [--tab ID] \
                  [--split right|down] [--env KEY=VALUE] [--focus|--no-focus] -- <argv...>
herdr agent wait <target> --status idle|working|blocked|unknown [--timeout MS]
herdr agent rename <target> <name>|--clear
herdr agent focus <target>
herdr agent explain <target> [--json]              # why herdr classifies the pane the way it does
herdr agent explain --file PATH --agent LABEL [--json]
herdr agent attach <target> [--takeover]           # interactive takeover — for a human, not automation
```

`--source detection` reads the snapshot herdr's status classifier sees — the fastest way to
check why a pane reports its current status.

## Panes

```sh
herdr pane list [--workspace <id>]
herdr pane current [--pane ID|--current]
herdr pane get <pane_id>
herdr pane layout [--pane ID|--current]
herdr pane process-info [--pane ID|--current]
herdr pane read <pane_id> [--source ...] [--lines N] [--format text|ansi]
herdr pane send-text <pane_id> <text>
herdr pane send-keys <pane_id> <key> [key ...]     # enter, escape, up, down, c-c, ...
herdr pane split [<pane_id>] --direction right|down [--ratio F] [--cwd PATH] [--env K=V] [--focus|--no-focus]
herdr pane move <pane_id> --tab <tab_id> --split right|down | --new-tab | --new-workspace [options]
herdr pane rename <pane_id> <label>|--clear
herdr pane zoom [<pane_id>] [--toggle|--on|--off]
herdr pane close <pane_id>                         # destructive
herdr pane report-agent <pane_id> --source ID --agent LABEL --state idle|working|blocked|unknown \
                        [--message TEXT] [--seq N] [--agent-session-id ID] [--agent-session-path PATH]
herdr pane report-agent-session <pane_id> --source ID --agent LABEL [--seq N] [...]
herdr pane release-agent <pane_id> --source ID --agent LABEL [--seq N]
```

`pane report-agent` lets a non-TUI process self-report agent status so herdr tracks it like a
first-class agent — useful when wrapping a headless runner.

## Waits

```sh
herdr wait agent-status <pane_id> --status idle|working|blocked|done|unknown [--timeout MS]
herdr wait output <pane_id> --match <text> [--regex] [--source ...] [--timeout MS]
```

A timeout returns a report, not an error — re-check the pane instead of assuming failure.

## Layout and state

```sh
herdr api snapshot                                 # full runtime state dump (JSON)
herdr api schema [--json]                          # socket protocol schemas
herdr workspace <subcommand>                       # workspace helpers
herdr tab <subcommand>                             # create / rename / focus / close (close is destructive)
herdr worktree <subcommand>                        # git worktree helpers
herdr notification <subcommand>
herdr integration <subcommand>
herdr session <subcommand>
```

## Gotchas

- Agents on the alternate screen (most TUIs) do not grow host scrollback — `--source recent`
  can be thin; use `--source detection` or `visible` for state, `recent` for transcripts.
- CLI errors print JSON to stderr (exit 1 for server errors, exit 2 for syntax).
- `herdr agent send` writes literal text only — submit with `herdr pane send-keys <pane> enter`.

## MCP equivalent

When the plugin's `herdr` MCP server is connected, these map to `list_agents`, `start_agent`,
`send_to_agent`, `send_keys`, `read_pane`, `wait_for_agent_status`, `wait_for_output`,
`get_layout`, and `edit_layout`, plus tab/workspace/worktree/pane helpers — 13 tools total.
MCP `wait_for_agent_status` accepts a status list but cannot wait for `unknown`.
