# State

Detailed reference for Overseer's state storage, KV schema, disk artifacts, and memory structures.

## Tern plugin KV

Stored at `~/Library/Application Support/Tern/plugin-data/overseer/kv.json` (or Linux equivalent under `$XDG_DATA_HOME` / `~/.local/share/Tern/`), accessed via `tern.kv`:

| Key | Shape | Notes |
|---|---|---|
| `projects` | `[{id, name, path, base, linear?}]` | `id` is the repository's shared Git directory, so linked checkouts dedupe. `linear` overrides the CLI workspace. |
| `worktree-names` | `{[projectId]: {[path]: name}}` | Display names. Removed when Overseer deletes the worktree, or when a refresh finds it gone. |
| `sessions` | `{[path]: {id, host, project}}` | Worktree → Tern session binding. A refresh drops bindings of gone worktrees whose session is closed. |
| `icebox` | `{[projectId]: {[path]: true}}` | Parked worktrees. `show-icebox` is the header toggle. |
| `jobs` | `{[id]: {kind = "create" \| "remove", project, receipt, pane, …}}` | Running creations and deletions; creations also carry `branch`, `displayName?` and `agentPrompt?`. Persisted so a plugin reload doesn't lose them. |
| `panel-open`, `panel-share` | `boolean`, `number` | Whether you want the Projects panel shown (the shortcut sets it), and its width as a share of the tab. |
| `pending-form:<pane>`, `form:<token>` | `{token, mode, project, path?, response?, launched?}`, the form's answer | An open form's request, keyed by its pane, and the answer the form block posts. `forms` moves the answer into the request (`response`) so a reload resumes it. |
| `agents-finished` | `{[paneId]: "done" \| "failed"}` | Agent turns that ended while their tab wasn't shown. Cleared when the tab is shown, the agent starts again, or the pane closes. |
| `rails` | `{queue, unread, sentAt, checkin, told}` | Coordination relays; see [Flows](flows.md#coordination). |

## Operation log (`ops.log`)

Beside the kv store, `ops.log` in the plugin data directory is the operation log:
- Format: one JSON line per creation, deletion, brief, relay and gone worktree (start, finish, duration, outcome, first warning), newest last.
- Capped at 256 KB.
- The **Show Overseer's operation log** command opens it in `less`.
- The same entries go to Tern's log (`~/Library/Logs/Tern/tern.log` or Linux equivalent), at `info` for routine starts and finishes, and `warn` for failures, warnings and "needs you".

## Shared Git configuration

Shared with agents through Git config (Worktrunk's per-branch variables):
- `overseer-parent`: sub-worktree links.
- `overseer-lead`: the pane id of a worktree's overseer session.

## Disk files (`tern.plugin.data`)

Files located under `tern.plugin.data`:
- `creation-<id>.status` receipts written by the operation helper script.
- `removal-<id>.status` receipts written by the operation helper script.
- `visual-pr/<project>-<pr>-<headsha>.md` outline cache.

## In-memory state

Maintained in memory only during runtime:
- Per-project `cache`: worktrees, PRs, gone paths with open sessions, loading/error flags.
- `rendered`: signature cache per panel pane.
- The panel's active pane and last tab.
- Linear default workspace lookup.
- Open outline canvases for Visualize PR.
