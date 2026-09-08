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

Without an argument, `switch`, `new` and `name` prompt via rofi.

## Install

    ln -s ~/src/sway-context-manager/bin/sway-context ~/bin/sway-context
    ln -s ~/src/sway-context-manager/sway/context.conf ~/.config/sway/config.d/context.conf
    mkdir -p ~/.config/sway-context && cp config.example.json ~/.config/sway-context/config.json
    swaymsg reload

Keys (see `sway/context.conf`): `$mod+x` switch or create, `$mod+Shift+x` close current (windows stay),
`$mod+Control+x` name or rename the current slot.

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

## Later

- Per-context app templates (terminal in project dir, editor, ...).
