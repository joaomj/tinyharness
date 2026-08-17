# Why an OpenCode TUI still takes about 6 seconds to start

Working note. Updated 2026-08-17. Evidence comes from the OpenCode
resource benchmark (work-vms repository, 20260817-172100-6394) run against
the persistent prudentia server on the Contabo VPS, OpenCode 1.18.18.

## The measured behavior

A full OpenCode TUI attached to a warm persistent server takes about 6.2
seconds to first draw (median 6382 ms, p95 7187 ms). The server-side project
is already warm: the first attach after a server restart was 12.9 seconds,
and every later attach was 6.1 to 6.4 seconds.

The ~6 second floor repeats on every fresh attach. It is a per-process
client cost, not a server cost.

## The breakdown of the 6 seconds

| Phase | Measured | Source |
|---|---|---|
| tmux pane and login shell | about 0.7 s | stopwatch between pane start and opencode exec |
| server session create (POST /session) | 12 to 65 ms | server HTTP log |
| LSP processes at first draw | none | process listing during first 2 seconds |
| project size | 872 and 994 tracked files | git ls-files count |
| the opencode client itself | about 5 s, the dominant term | remainder after the phases above |

The opencode client is a full Bun process. It loads the CLI module graph
(the `--version` floor alone is 1.428 seconds warm), boots a second server
worker for the TUI, probes the terminal palette, and initializes the sync
context. The client startup log shows, on every launch:

- `loading path=.../config.json, opencode.json, opencode.jsonc`
- a `watcher backend ... backend=inotify` line on `~/.config/opencode`
- `global event connected` only after those loads

The configuration tree under `~/.config/opencode` is 75 MB across 4661
files. Of those, 63 MB is plugin `node_modules` and 4.4 MB is the
`Prudentia_Roadmap` directory. Plugin code runs inside the client process,
so the client must load that tree on every launch.

## Why the server cannot pay this cost

The server owns sessions, projects, and events. Configuration, plugins, and
terminal state belong to each client process. A warm server cannot make a
new client boot faster, and the server has no flag to host configuration for
clients in OpenCode 1.18.18. This is an architecture property, not a
configuration mistake.

## The mitigation that works: keep one TUI process alive per project

The 6 seconds are paid once per project if the TUI process stays alive.
Reconnecting to the same process costs about 40 ms (median 40 to 44 ms in
the benchmark).

Implementation: each project TUI runs in a named tmux session
(`tui-<project>`) on the VPS. A short attach script creates the session on
first use and attaches on later uses. An idle reaper (systemd timer per
account, default limit 30 minutes) ends TUI sessions that receive no
keystrokes, so memory and CPU are returned instead of being held forever.

The OpenCode server is never affected. A reaped session keeps its session
history on the server; the user reconnects with one command.

## Design consequence for future terminal clients

Any terminal-first agent UI that boots per invocation pays a fixed process
start cost. For small agents the cost is negligible. For heavy clients with
plugins and configuration trees, persistent client processes in a session
manager (tmux) with an idle reaper give the best of both: first use is slow,
every reconnect is instant, and idle memory is reclaimed.
