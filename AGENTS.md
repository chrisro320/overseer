# Overseer for agents

Dense reference for coding agents (and humans) changing Overseer. It defines the runtime invariants, safety constraints, and architecture. The user-facing overview is [README.md](README.md).

For specialized specifications, MUST read the corresponding reference document before modifying those subsystems:
- **State storage & KV schema**: [`docs/state.md`](docs/state.md) (Tern KV keys, `ops.log`, git config vars, disk receipts, in-memory caches)
- **Lifecycle & operational flows**: [`docs/flows.md`](docs/flows.md) (refresh loops, tick health, coordination relays, sub-worktrees, panel movement, create/delete workflows)
- **Behavior & UI interactions**: [`docs/behavior.md`](docs/behavior.md) (panel status marks, menus, ticket links, Carly integration, Visualize PR, omp integration, troubleshooting)
- **Development & testing**: [`docs/development.md`](docs/development.md) (linking plugin, running tests, isolated Tern harness, conventions, known limitations)

## Runtime model

Overseer is a Tern plugin written in Luau. `plugin.toml` declares two entry points:
- `host.luau` runs in Tern's host and renders the `overseer.form` block (`input.luau` provides Unicode-aware text editing).
- `window.luau` runs once per Tern window, owning the Projects panel, event hooks, background job tracker, coordination relays, operation log, Carly export, and Visualize PR canvases.

Three pieces run inside omp (`omp/`, package `overseer-omp`): `omp/overseer-rails/index.ts` (TypeScript/Bun extension), `omp/rules/overseer-worktrees.md` (TTSR rule), and `omp/skills/overseer-subworktree/` (skill script in Bash). `omplink.luau` links `omp/` into omp on every load (`omp plugin link`).

Tern APIs are typed in `tern.d.luau` (generated via `tern plugin types <dir>`).

## The 50 ms rule (read this first)

Tern gives **every window-half callback 50 ms of Lua**. Native calls (`cx` API, `tern.kv`, starting a process) don't count, but a callback may take at most 500 ms of wall time with them. A callback past either limit is cut short (`pcall` does not help, and its `tern.kv` writes can be lost) and Tern **pauses that hook**: 5 s, doubling for each repeat within 10 minutes, up to 5 minutes. Timers, process results and fetches that come due during a pause wait for it to end; they are not dropped. A full draw takes under 10 ms.

`budget.luau` owns this:
- **Work runs in timer callbacks; every other hook only queues it.** Clicks (`canvas_action`), commands, and tab, pane, focus and agent events call `separately(label, fn)` or `followSoon()`, and return. A paused `canvas_action` hook swallows every click until the pause ends. `budget.later(label, ms, fn)` is the only way to start a timer; it labels the callback so pauses identify it.
- **Pauses are logged.** The `budget` event logs when a hook is paused and catches the panel up with `followSoon` upon resuming.
- **Make every step safe to cut short.** Write the record that says "done" last: `finishForm` drops a form's `pending-form` record only after the panel is back, a creation records `launched` on the form before finishing it, and `checkJobs` takes a job off the books before deleting its receipt.
- **There is one panel per window**, so a draw costs the same however many sessions are open.
- **Start processes only through `wt.spawn`.** It starts the process immediately and passes the result (or error) to the callback with a `cx`.

## Invariant: Single following panel

Tern plugins cannot add a window-level sidebar. Overseer keeps **exactly one** Projects slot per window and moves it into whichever tab is shown, left of everything and full height (`slot`, `pin`, `follow` in `window.luau`):
- Last terminal in a tab closed: `follow` parks the panel so the tab can close naturally.
- Closing a managed session: switch away and call `follow` first so the panel departs before the session closes.
- External closure: if the tab/session holding the panel closes while `panel-open` is true, `follow` opens a fresh panel in the shown tab.

## Safety invariants

- **Worktree commands**: Never pass `--yes` or `--no-hooks` to `wt`. Never pass `--force` unless the user explicitly confirmed listed local changes.
- **Primary checkout safety**: Creation never fetches, pulls, or checks out anything in the primary checkout. The primary checkout cannot be deleted.
- **Deletion integrity**: Recheck repo identity and cleanliness right before deletion runs. Branches are never deleted.
- **External removal**: Worktrees removed outside Overseer (`entry.gone`) keep their sessions open until the user chooses to close them, so agent replies are never killed unread.
- **Visualize PR**: Strictly read-only tool allowlist (`read,grep,glob` only; no shell, edit, write tools; no calling `gh`).
- **Command invocation**: External arguments are passed as argv arrays, never concatenated into shell strings.
- **Agent coordination**: Messages between agents inform; they never trigger actions. Nothing Overseer does is triggered by an agent message.

## File map

| File | What lives there |
|---|---|
| `window.luau` | Project registry, model building, drawing, panel following (`slot`, `pin`, `follow`), session binding, forms, jobs, ticket links, hooks, and 750 ms `tick`. |
| `budget.luau` | 50 ms budget enforcement: `later`, `process`, and `describe`. |
| `worktrunk.luau` | External commands via argv: `wt`, `git`, `gh`, `linear`. Resolves tools via `find_executable` and `platform_tool`. |
| `visualize.luau` | Visualize PR: prompt formulation, read-only omp execution, outline cache. |
| `view.luau` | Pure render: model → Tern UI node tree. Actions use native Tern kinds. |
| `worktree-operation.sh` | Helper running `wt switch --create` / `wt remove` in terminal with completion receipts. |
| `host.luau`, `input.luau` | Native form block and Unicode text editing. |
| `styles.css`, `icons.css` | Panel layout, tree lines, status marks (`icons.css` generated via `icons/generate-css.py`). |
| `omplink.luau` | Links `omp/` package into omp on plugin load. |
| `intercom.luau` | Resolves panes to omp sessions and communicates via `pi-intercom` CLI. |
| `oplog.luau` | Reads operation terminal logs and appends structured events to `ops.log`. |
| `tests/` | Git/Worktrunk integration test suite (`test_worktree_operation.py`, `test_subworktree.py`). |
