# Development & limitations

Environment setup, test execution, UI smoke-testing harness, and known system limitations.

## Developing

To work on Overseer, use a checkout linked into Tern instead of the installed copy (Tern then reloads the plugin whenever a file in the checkout changes):

```sh
tern plugin remove overseer   # if it was installed with `tern plugin install`
git clone https://github.com/mmnavarr/overseer.git && cd overseer
tern plugin link "$PWD"
```

### Running tests

Run the helper's tests (they use temporary repos, config and approvals):

```sh
python3 -B tests/test_worktree_operation.py -v
python3 -B tests/test_subworktree.py -v
bash -n worktree-operation.sh omp/skills/overseer-subworktree/overseer-subworktree
```

Check the worktree rule fires on the commands it should, and only those:

```sh
omp ttsr test --rule omp/rules/overseer-worktrees.md --source tool --tool bash 'wt switch --create fix/x --no-cd'
omp ttsr test --rule omp/rules/overseer-worktrees.md --source tool --tool bash 'wt switch fix/existing'
```

The extension's pure helpers (`familyFromConfig`, `mergeNotes`, `promptNote`, …) are exported, so `bun` can import them for a quick check; end to end, start an omp session in a Tern pane with `omp -e omp/overseer-rails/index.ts` and look for its hidden `overseer-rails` messages in the session file.

### Isolated UI smoke testing

Smoke-test UI changes in an **isolated Tern window** so your real sessions are untouched:

```sh
D=$(mktemp -d); mkdir -p "$D/config"
TERN_CONFIG_DIR="$D/config" tern plugin link "$PWD"
TERN_CONFIG_DIR="$D/config" TERN_DAEMON_SOCKET="$D/daemon.sock" \
  tern --control "$D/desktop.sock"
```

Drive it with `tern ctl --control "$D/desktop.sock" …`:

- `ready`, `palette Projects`, `key Enter`, `click X Y [right]`, `shot NAME` (screenshots land in `target/shots/tern/live/`)
- `tree "[data-role=overseer\.worktreeRow]"` returns node rects and text
- `a11y` returns menu items and their bounds
- `carly lua "…"` runs Lua with a window `cx`, and `state` shows the result under `.carly.last_tool`

Seed projects by writing `$D/config/plugin-data/overseer/kv.json` before launch.

### Engineering conventions

- Editing any file in a linked plugin reloads it in every running Tern. Develop in a separate checkout linked into an isolated window so your everyday Overseer isn't reloaded mid-edit.
- After changing an icon, run `python3 icons/generate-css.py`.
- Bump `version` in `plugin.toml` and `omp/package.json` for user-visible changes and add an entry to [`CHANGELOG.md`](../CHANGELOG.md). A change to how the panel or its state is structured is a major version.
- Keep external commands in `worktrunk.luau` as argv arrays. Shell is only used where a redirect is needed, with arguments as `$0…$n`.

## Known limitations

- **macOS and Linux**: The folder picker uses AppleScript on macOS. On Linux it uses the desktop's own dialog — `kdialog`, KDE's, on KDE (the widget Dolphin uses), `zenity` elsewhere, each falling back to the other — and without either it asks for a path over the focused pane. The shortcut is ⌘⌥⇧P; on Linux ⌘ is the Super key. Tern's desktop test harness (`tern ctl`, headless scenarios) is macOS-oriented; the plugin's own tests and the helper script run on both.
- **Visualize PR limits**: Visualize PR fails on PRs whose diff exceeds GitHub's 20,000-line API limit (`gh pr diff` returns HTTP 406). The tab shows the error.
- **Panel move animation**: The panel moves into each tab you switch to, so every switch animates briefly and resizes the terminals in both tabs (full-screen programs redraw). In a tab split top-and-bottom it sits beside the top half.
- **Reload during Visualize PR**: A plugin reload during a Visualize PR run, including the one Tern does when you edit a file in a linked plugin, ends the run and leaves its tab on the waiting message. Run it again.
- **Multiple Tern windows**: Multiple Tern windows work but haven't been exhaustively tested together.
- **Coordination dependencies**: Coordination needs omp and pi-intercom, on one machine. Other agent CLIs get the panel and status, not the rails.
- **Overseer session wiring**: Only a worktree's overseer session is wired to its parent: another omp session in the same sub-worktree that gets stuck doesn't wake the parent, though its notes still reach the family.
- **needs you? false positive**: **needs you?** is read from the setup tab's output, so a setup step that prints nothing for a minute trips it too.
- **Worktree rule firing**: The worktree rule fires at most once per omp session; after that the agent relies on the skill it read.
