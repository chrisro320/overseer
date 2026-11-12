# Flows

Detailed operational flows, background jobs, coordination protocols, and lifecycle transitions.

## Refresh

`refresh` runs `wt list --format=json` per project, beside `refreshLinks` (`wt config state get --format=json`, keeping the `overseer-parent` vars as `entry.parents` and `overseer-lead` as `entry.leads`; a failed read keeps the last links), then `refreshPullRequests` (one `gh pr list` per GitHub project, at most every 2 minutes, immediately after ↻ or a worktree set change).

An open panel refreshes every 15 s from `tick`, a hidden one every 60 s so the coordination relays keep working.

## Tick health

`tick` runs every 750 ms under a generation number and records `lastTick`.

`ensureTicking`, called from every event hook and from `processClock` (a `/bin/sleep 5` process whose callback runs in the process hook, outside the timer budget), starts a new generation when the tick is more than 5 s late and Tern hasn't paused the timer hook, logging `tick restarted`.

While the tick was late or the timer hook is paused, the process clock also runs `checkJobs(cx, true)` (finishing jobs inline) and `follow`. The tick draws only every 5 s (`followSoon(quiet)`); events draw at once. `snapshot(cx)` fetches panes, tabs and agents once per callback for `model`.

## Coordination

Three parts, one shared vocabulary (the `overseer-parent`/`overseer-lead` branch variables and pi-intercom):

- **The omp extension** (`omp/overseer-rails/index.ts`) runs in each interactive omp session in a Tern pane (subagents and the advisor are headless and skip it). `before_agent_start` reads the family from git config (`worktrunk.state.<branch>.vars.*`, milliseconds where `wt` takes ~0.4 s), diffs each relative's branch tip against what this session last saw (kept as `overseer-rails.seen` session entries, so it survives resume), adds the latest note from the board, and flags files both branches change against `origin/HEAD`. Notes live in pi-intercom's extension state `overseer-rails/v1` (newest 80, well under its 64 KiB limit; the elected owner commits, others publish to it). `overseer_family` resolves a relative's lead pane to its intercom address with `tern inspect --json` (the pane's `omp_session_file` names the omp session id, which is pi-intercom's default address) and sends through pi-intercom's outbox.
- **Overseer's relays** (`window.luau`, "Coordination rails"): `onAgentChange` (from `agent_state`/`agent_finished`) records check-ins and relays a sub-worktree's lead going waiting or failed, and remembers a finished turn so `noticeFinished` relays it once its PR is known (once per PR and head commit, within 15 minutes). `noticeMoves` compares each listing's head commits with the last: a parent that moved notifies children containing its old tip when that tip isn't on the base branch. `tell` queues per worktree; `flushRails` sends to the worktree's lead (`leadOf`: `overseer-lead` while its agent runs, else the only agent) at most once a minute per pane, three attempts. `intercom.luau` resolves the pane with `tern inspect` and runs pi-intercom's CLI as `overseer`. `syncLeads` keeps `overseer-lead` true for worktrees in a family.
- **Briefs**: `overseer-subworktree --brief` writes `overseer-brief.md` into the new worktree's private Git directory; `startBriefs` renames it to `overseer-brief.started.md` (so a refresh can't start it twice) and runs `tern new session|tab … -- omp @<file>` without switching you away, binding a new session to the worktree.

Kept in kv `rails` (times are `os.time()` seconds): `queue`, `unread`, `sentAt`, `checkin`, `told`. `reconcileGone` drops gone worktrees' records; rate marks expire after a minute.

## Sub-worktrees

`owners(entry)` maps each worktree to the top of its `overseer-parent` chain among the listed worktrees (a link past the listing stops there, a cycle or a primary checkout as child stays on top).

`model` puts owned worktrees in their owner's `children` (Ice Box ones last; an iced owner parks them; `folded` is in memory) with `childAgent` the most urgent agent among them; unfolded, the view wraps the owner's group and a `c` column of its sub-worktrees in a `t<hex>` lineage column, which draws the tree lines (`styles.css`), and path recovery (`w<hex>.row`, matched at the id's end) works unchanged. Rows mark the active worktree `active` rather than Tern's `pick`, whose accent bar would sit among those lines.

**Move under…** offers only top-level worktrees (`cx:pick`), and both it and **Make top-level** go through `setParent`: an optimistic `entry.parents` update, `wt config state vars set|clear`, then `refreshLinks`.

Deleting an owner opens the `delete-family` confirmation; **Delete all** queues one `beginRemoval` per sub-worktree with `keepIfDirty` (a dirty one is kept with a toast instead of asking) and then the owner's own.

## Gone worktrees

After each successful listing, `reconcileGone` treats a recorded path that isn't listed, isn't being deleted and isn't on disk as removed outside Overseer. It drops the path's `worktree-names` and `icebox` entries.

If the path's bound session is still open it goes in `entry.gone`, and `model` adds it as a `gone` row (`goneNode` in the view: click switches to the session, the menu offers only **Close session**). Otherwise the binding is dropped.

*Safety rationale*: Never closing these sessions automatically is deliberate: an agent that removes its own worktree would be killed before you read its reply.

## The following panel

Tern panes belong to one tab, and plugins can't add a window-level sidebar, so the window keeps exactly one Projects slot and moves it into whichever tab is shown, left of everything and full height.

`slot` finds it: the navigator canvas or a form shown in its place, adopting what a reload left open and closing any extra.

`pin` moves it with `move_to_tab` (or `unpark`) then `move(…, "left")` and steers the split ratio to the remembered share, since Tern resizes in cells and hidden tabs measure differently. `followSoon` queues `follow` on `focus`, `tab_created`, `tab_closed`, `pane_created`, `pane_closed`, `window_start` and every `tick`. Moves work across sessions.

**Edge cases**:
- **Last terminal in a tab closed**: the panel would be left alone, so `follow` parks it; the tab closes as Tern would have, and the next shown tab (or new tab) takes the panel back.
- **Overseer closes a session**: `closeManagedSession` switches away and calls `follow` first, so the panel leaves before the session closes.
- **Tern closes the panel's tab or session**: the panel dies with it. If you wanted it (`panel-open`) and its last tab is gone, `follow` opens a fresh one in the shown tab. If its tab is still there, you closed the panel yourself, and it stays closed until the shortcut.

## Create

Form or Carly → `wt.validate` → `launchCreation` opens a background tab running `worktree-operation.sh` and records a job → `trackCreations` (on `tick`) reads the tab's last lines: the newest `◎ Running` step for the row, a question on the last line (Worktrunk's `[y/N]`) or 60 s without output for **needs you?**; `model` shows the job as a **creating** row, right away if Worktrunk doesn't list the worktree yet → `checkJobs` reads the receipt → `jobFinished` scans the tab's output for warnings (`oplog.warnings`: Worktrunk's ▲/✗ marks and `warn:`/`error:` labels, not its echo of hook commands or its approval header), closes the tab on success, and writes the outcome to `ops.log` → toast, apply the display name by branch (`applyCreatedName`), and for ticket worktrees `startTicketAgent` opens the session and calls `cx.agents:start({command = "omp", prompt = …})`.

Creation succeeded means `wt` exited 0; hook warnings are only logged.

## Delete

`beginRemoval`: `wt.validate_removal` (repository identity, branch) while the row shows **checking** → clean: removal job right away.

Dirty: `wt.local_changes` (`git status --porcelain=v1 -z --no-renames` plus `git diff HEAD --numstat`, first 200 entries) fills the confirmation; **Force delete** reruns `beginRemoval` with `force`, and the helper runs in `force-remove` mode (`wt remove --force`, no cleanliness check).

Either way: removal job in a background tab → receipt → close the worktree's session only if the directory is gone, then drop its name, binding and Ice Box entry. Cancelling during a check never starts a deletion (`removalChecks`).

## Agent status

`agentActivity` reads `cx.agents:list()`: `state == "waiting_input"` → waiting, `"working"` → working, otherwise the pane's `agents-finished` mark (done or failed).

An `agent_finished` event with result `done` or `failed` sets that mark unless the agent's tab is shown; `agent_state` back to working or waiting clears it, and `markSeen` (after every `follow`) clears marks whose tab is shown or whose pane is gone.

For agent CLIs in plain terminals (`omp`, `pi`, `claude`, `codex`, `opencode`, `gemini`, `aider`, `amp`, `cursor-agent`, `crush`, `goose`) it uses the pane's `busy` flag (working only).

Per session and tab the state that most needs you wins: waiting, then failed, then done, then working.

## Links

Ticket and PR icons send `link=<url>`. `cx:open` puts a URL in a browser split beside the focused pane whatever `how` says, so `openLink` moves the new browser pane into a tab of its own.

## Remove project

Drops the project from `projects` and its in-memory cache; refuses while a creation or deletion runs for it. `worktree-names`, `icebox` and `sessions` stay, so adding the repository again brings them back.

## Ticket links

`ticketOf` takes the first uppercase `ABC-123` in the branch (`%f[%w](%u[%u%d]*%-%d+)%f[%W]`), so `chore/stage-1` doesn't count. The workspace is `project.linear`, else the slug from `linear auth whoami` (looked up once; ↻ retries a failure). URL: `https://linear.app/<workspace>/issue/<id>`.

## Visualize PR

`wt.pull_request_meta` (`gh pr view --json …`) and `wt.pull_request_diff` (`gh pr diff > file`) write the PR into a temp dir under plugin data. Then:
```sh
omp -p --no-session --no-extensions --no-skills --no-title --tools=read,grep,glob "<prompt>"
```
runs in the worktree with a 10-minute timeout. Its stdout from the first Markdown heading becomes the canvas.

*Read-only guarantee*: The tool allow-list is strictly limited to `read,grep,glob`. Never add shell, edit or write tools, and never let it call `gh`.

## Carly

`tern.carly.export("ticket_worktree", …)` validates the id, resolves the project (argument by name or path, else the current session's project), validates the branch and starts creation. It replies once creation *starts* (setup can outlast Carly's 120 s limit). `tern.carly.context` tells Carly the registered project names.
