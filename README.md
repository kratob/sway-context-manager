# WIP: sway-context

Named "contexts" for sway + swaysome. A context is a swaysome slot `N` (0-9)
with a name. On every *work* output (every active output that is not listed
as ambient) the workspace `<group>N` is renamed to `<group>N:<name>`, so the
name shows up in waybar and `swaysome focus N` keeps working unchanged.

## Commands

    sway-context switch [name]   menu of contexts; pick one, or type a new name to create it
    sway-context new [name]      create a context in a free slot and switch to it
    sway-context name [name]     name the current slot as a new context (switches all work outputs to it,
                                 claiming whatever is on those workspaces), or rename an existing one.
                                 Aliases: rename, adopt
    sway-context close [name] [--kill]
                                 forget a context; with --kill also close its windows
    sway-context list            show slots, names, window counts
    sway-context current         print the current context name (for bars)
    sway-context sync            re-apply stored names after a sway restart
    sway-context template <name> show what the matching template would launch (dry run)

Without an argument, `switch`, `new` and `name` prompt via rofi.

## Install

    ln -s ~/src/sway-context-manager/bin/sway-context ~/bin/sway-context
    ln -s ~/src/sway-context-manager/sway/context.conf ~/.config/sway/config.d/context.conf
    mkdir -p ~/.config/sway-context && cp config.example.json ~/.config/sway-context/config.json
    swaymsg reload

Keys (see `sway/context.conf`): `$mod+x` switch or create, `$mod+Shift+x` close current (windows stay),
`$mod+Control+x` name or rename the current slot.

## Outputs

Outputs are addressed by position: all active outputs sorted left to right, index 0
being the leftmost and -1 the rightmost. `ambient_outputs` lists the indexes (or sway
output names) that do not take part in contexts; every other output is a work output.

## Templates

A template launches applications when a context is *created* (`new`, a new name typed
into the switcher, or `name` on an unnamed slot). Not on `switch` or rename.

    "templates": [
      {
        "match": "^proj-",
        "outputs": {
          "1":  { "launch": [["zed", "~/src/myproject"]], "layout": "stacking" },
          "-1": { "launch": [["google-chrome", "--new-window", "https://…/{name}"]] }
        }
      }
    ]

- `match` is a regexp against the context name; the first matching template wins.
- `outputs` keys are output position indexes (see above). Ambient or missing outputs are skipped.
- `launch` is a list of argv arrays. `{name}` is the context name, `{rest}` the part after
  the match (`proj-1234` → `1234`), `{1}`..`{9}` are regexp capture groups; a leading `~` is expanded.
- `layout` runs `layout <mode>` on the (empty) workspace before launching.
- Apps are launched one at a time; the next new window sway reports is moved to the
  target workspace, so placement does not depend on focus or startup time
  (`launch_timeout` seconds per app).
- Programs started from a keybinding run in sway's environment, which lacks variables your
  shell sets up (typically `SSH_AUTH_SOCK`). `launch_env_files` lists shell files to source
  before launching, e.g. `["$XDG_RUNTIME_DIR/ssh-agent.env"]`.
- Output of launched programs and launcher warnings go to `~/.local/state/sway-context/launch.log`.
  `sway-context template <name>` shows what would run, including the environment picked up.

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
