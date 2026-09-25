# WIP: sway-context

Named "contexts" for sway + swaysome. A context is a swaysome slot `N` (0-9)
with a name. On every *work* output (every active output that is not listed
as ambient) the workspace `<group>N` is renamed to `<group>N:<name>`, so the
name shows up in waybar and `swaysome focus N` keeps working unchanged.

## Commands

    sway-context switch [name]   menu of contexts; pick one, or type a new name to create it
    sway-context new [name]      create a context in a free slot and switch to it
    sway-context next / prev     cycle through contexts in slot order
    sway-context name [name]     name the current slot as a new context (switches all work outputs to it,
                                 claiming whatever is on those workspaces), or rename an existing one.
                                 Aliases: rename, adopt
    sway-context close [name] [--kill]
                                 forget a context; with --kill also close its windows
    sway-context kill [name]     close all windows of a context and forget it (= close --kill)
    sway-context teardown [name] run the project's teardown hook (e.g. remove the worktree), then kill
    sway-context list            show slots, names, window counts
    sway-context current         print the current context name (for bars)
    sway-context sync            re-apply stored names after a sway restart
    sway-context run [action]    run an action in the current context (see Projects); menu without an argument
    sway-context env [--fish]    print the current context's environment, for eval in a shell
    sway-context dir             print the current context's project directory
    sway-context show [name]     show how a context resolves: project, vars, env, dir, actions (dry run)

Without an argument, `switch`, `new`, `name` and `run` prompt via rofi. Any dmenu-like
program works: set `"menu": ["noctalia", "dmenu"]` for the noctalia launcher, for example.

## Install

    ln -s ~/src/sway-context-manager/bin/sway-context ~/bin/sway-context
    ln -s ~/src/sway-context-manager/sway/context.conf ~/.config/sway/config.d/context.conf
    mkdir -p ~/.config/sway-context && cp config.example.json ~/.config/sway-context/config.json
    swaymsg reload

Keys (see `sway/context.conf`): `$mod+x` switch or create, `$mod+Shift+x` close current (windows stay),
`$mod+Control+x` name or rename the current slot, `$mod+Tab` / `$mod+Shift+Tab` next / previous context,
`$mod+p` menu of actions for the current context, `$mod+Return` terminal in the current context.

## Outputs

Outputs are addressed by position: all active outputs sorted left to right, index 0
being the leftmost and -1 the rightmost. `ambient_outputs` lists the indexes (or sway
output names) that do not take part in contexts; every other output is a work output.
If every active output is ambient, for example an undocked laptop, all of them count as work
outputs, so contexts keep working on the single screen.

## Projects and actions

A context can belong to a *project*, matched by regexp on the context name. The project
supplies placeholder values, an environment, a working directory and a set of *actions*:
named commands you run in the context with `sway-context run`, and that a new context
runs automatically via `on_create`.

    "actions": {
      "terminal": { "launch": ["kitty"], "description": "terminal in the project directory" },
      "editor":   { "launch": ["zed", "{dir}"], "output": 1 },
      "tracker":  { "launch": ["google-chrome", "--new-window", "{issue_url}"], "output": -1 },
      "mr":       { "launch": ["sh", "-c", "URL=$(glab mr view \"$1\" -F json | jq -r .web_url); [ -n \"$URL\" ] || URL=$(glab repo view -F json | jq -r .web_url); exec google-chrome --new-window \"$URL\"", "sh", "{branch}"], "output": -1 },
      "worktree": { "launch": ["sh", "-c", "cd {repo} && wt switch -y -c {branch} -b origin/master --config-set 'worktree-path = \"{worktree_path}\"'"],
                    "window": false }
    },
    "projects": [
      {
        "name": "proj-issue", "extends": "proj",
        "match": "^proj-(\d+)$",
        "vars": { "issue_url": "https://tracker.example.com/issue/{1}" }
      },
      {
        "name": "proj",
        "match": "^proj-",
        "vars": { "repo": "~/src/myproject", "branch": "me/{name}", "worktree_path": "{repo}.{name}",
                  "issue_url": "https://tracker.example.com/board" },
        "env": { "CONTEXT_NUMBER": "{slot}", "TEST_ENV_NUMBER": "{slot + 1}", "PORT": "{3000 + slot}" },
        "dir": "{worktree_path}",
        "setup": "worktree",
        "on_create": ["editor", "tracker"]
      }
    ]

### Actions

- `launch` is the argv. Every argument may use placeholders: `{name}` (context name),
  `{slot}` (slot number, a small integer unique among live contexts), `{rest}` (the name
  after the project match), `{1}`..`{9}` (capture groups), `{dir}` (project directory) and
  any key of the project's `vars`. Integer arithmetic over them works too: `{3000 + slot}`.
  Unknown `{…}` are left alone; a leading `~` is expanded.
- Everything launched runs in `dir` (if the project has one) with the project's `env` plus
  `SWAY_CONTEXT`, `SWAY_CONTEXT_SLOT` and `SWAY_CONTEXT_DIR`. So a terminal opened via an
  action already sits in the right directory with the right variables, and an editor started
  this way passes them on to its integrated terminals.
- `output` is the position index the window opens on; without it, the focused output.
  The window lands on the context's workspace of that output.
- `layout` runs `layout <mode>` on that workspace first.
- `window: false` marks commands that open no window (setup steps, clipboard helpers).
  They run synchronously and only log; everything else is launched one at a time, and the
  next new window sway reports is moved to the target workspace (`launch_timeout` seconds
  per app), so placement does not depend on focus or startup time.
- `env` adds variables for this action only. `description` is shown in the `run` menu;
  `hidden: true` leaves the action out of the menu (it can still be run by name and used as
  a `setup`/`teardown` hook).
- Actions are global. A project may add or override them in its own `actions` block.
- Actions may call `sway-context` itself, which puts commands into the `run` menu:
  `"kill": { "launch": ["~/bin/sway-context", "kill", "{name}"], "window": false }`.

### Projects

- `match` is a regexp; the first matching project wins, so list specific projects before
  general ones. `name` is for display and for `extends`.
- `extends` refines another project: `vars`, `env` and `actions` are merged, everything else
  is overridden. This keeps per-issue and per-project variants to a few lines.
- `vars` are placeholder values and may themselves use placeholders (and earlier vars).
- `env` is exported to everything launched in the context. `{slot}` is handy for anything
  that must differ between contexts running at the same time: `TEST_ENV_NUMBER` for
  parallel_tests style databases (`{slot + 1}`, since the first number is the empty one),
  `PORT` for dev servers. Projects only need env-driven defaults, so colleagues without
  sway-context see no difference.
- `dir` is either a path template or an argv whose stdout is the path. If the directory does
  not exist, `setup` (an action name or an argv) is run once and the lookup retried. This
  is how worktrees get created on demand without hardwiring any particular tool. Naming the
  worktree path yourself (as above, via worktrunk's `--config-set worktree-path`) keeps the
  directory tied to the context even when you check out someone else's branch in it.
- `teardown` is the inverse (an action name or an argv), run by `sway-context teardown` when
  the directory exists. If it fails, nothing else happens: e.g. `wt remove` refuses a worktree
  with uncommitted changes, so the context and its windows stay. `close` and automatic
  pruning never run it.
- `on_create` lists actions to run when a context is *created* (`new`, a new name typed into
  the switcher, or `name` on an unnamed slot). Not on `switch` or rename.

### Shell integration

`sway-context env` and `sway-context dir` let an existing terminal join the context:

    ctx() { eval "$(sway-context env)"; cd "$(sway-context dir)"; }          # bash/zsh
    function ctx; sway-context env --fish | source; cd (sway-context dir); end   # fish

### Environment and logging

Programs started from a keybinding run in sway's environment, which lacks variables your
shell sets up (typically `SSH_AUTH_SOCK`). `launch_env_files` lists shell files to source
first, e.g. `["$XDG_RUNTIME_DIR/ssh-agent.env"]`. Output of launched programs and launcher
warnings go to `~/.local/state/sway-context/launch.log`. `sway-context show <name>` prints
how a name resolves without running anything.

Failures are also shown as desktop notifications, since from a keybinding nobody sees stderr:
an action exiting non-zero (with the last lines of its output), a program that opens no
window within `launch_timeout`, and errors like an unknown action. `notify` is the command
used, run with a summary and a body argument; the default is
`["notify-send", "-u", "critical", "-a", "sway-context"]`, `[]` turns notifications off.

## Files

- config: `~/.config/sway-context/config.json` (`SWAY_CONTEXT_CONFIG` overrides)
- state:  `~/.local/state/sway-context/contexts.json` (`SWAY_CONTEXT_STATE` overrides).
  Only needed to remember names of slots whose workspaces are currently empty;
  live workspace names always win.

## How it works

- Output group = first digit of the output's current workspace number (swaysome layout).
- Slot is free when no context has it and no workspace `<group>N` exists on any work output.
- Switching focuses `workspace number <group>N` on each work output, then renames the
  workspaces to `<group>N:<name>`. Focus returns to the work output you came from.
- Contexts with no windows left are dropped automatically when you switch or create a context.
  The context you are currently on is only dropped once you have left it.
- Moving windows between contexts is just `swaysome move N` as before.
