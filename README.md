# herdr-worktree-rm

Delete a Git worktree checkout from [Herdr](https://herdr.dev) without freezing
the UI.

Herdr's built-in **Delete worktree checkout…** keeps its confirmation dialog
open, and the whole client blocked, until `git worktree remove` finishes. On a
large checkout with `node_modules` or `target/` that is tens of seconds to
minutes (see
[herdrdev/herdr#1428](https://github.com/herdrdev/herdr/discussions/1428),
[#4578](https://github.com/herdrdev/herdr/issues/4578)).

This plugin does the same removal, but:

1. asks for confirmation in a small popup,
2. runs `herdr worktree remove` in a detached background process,
3. animates a `⠹ removing…` spinner on the workspace's sidebar row,
4. shows a notification when it finishes or fails.

Other workspaces and agents stay usable the whole time.

## Install

```bash
herdr plugin install dorshinar/herdr-worktree-rm
```

Requires `bash` and `jq`. Linux and macOS.

## Configure

Bind the action to a key in `~/.config/herdr/config.toml`:

```toml
[[keys.command]]
key = "shift+alt+d"
type = "plugin_action"
command = "dorshinar.worktree-rm.remove"
description = "delete worktree checkout (background)"
```

To see the spinner, add the `$removing` token to your Space rows. Setting
`rows` replaces the default layout, so include the defaults too:

```toml
[ui.sidebar.spaces]
rows = [
  ["state_icon", "workspace", { token = "$removing", fg = "#fc0", dim = false }],
  ["branch", "git_status"],
]
```

Then `herdr server reload-config`.

## Use

Focus the worktree workspace and press the key. The popup mirrors Herdr's own
delete dialog:

```text
 delete worktree checkout?
 This removes the checkout folder:
 ~/Code/wiz/wiz-worktrees/wiz/scratch-1022
 The branch is not deleted. The Herdr workspace will close.
 Dirty or untracked files will be permanently deleted.

   ↵ delete anyway      esc cancel
```

| Input                    | Effect                                                      |
| ------------------------ | ----------------------------------------------------------- |
| Enter, `y`, click button | remove; on a dirty checkout the button reads "delete anyway" and forces |
| `f`                      | remove with `--force` regardless of the dirty check         |
| Esc, `q`, click cancel   | cancel                                                      |

The dirty check is a `git status` capped at 1.5 s. When it times out the dialog
says so and plain Enter refuses a dirty checkout, so press `f` if you know you
want it gone.

The popup closes immediately; the removal continues in the background and a
notification reports the result.

You can also run it from a shell without a keybinding:

```bash
herdr plugin action invoke dorshinar.worktree-rm.remove
```

## How it works

- `action` mode opens the plugin's popup pane, passing the workspace id through
  `--env`, so the action targets the workspace it was invoked on.
- `confirm` mode runs inside the popup, draws the dialog, reads a key or a
  mouse click, and starts `run` mode in its own session (`setsid`, or `perl`
  on macOS) so it survives the popup's terminal closing.
- `run` mode calls `herdr worktree remove` and, while it runs, updates a
  display-only workspace metadata token every 150 ms with a 2 s TTL. The token
  expires on its own if the worker dies.

## License

MIT
