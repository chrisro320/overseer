# Behavior reference

Precise user-facing behavior, panel indicators, context menus, and architectural safety guarantees.

## The panel

1. **Show Projects**: <kbd>⌘</kbd><kbd>⌥</kbd><kbd>⇧</kbd><kbd>P</kbd> (Super+Alt+Shift+P on Linux), or **Show or hide Projects** in the command palette. Press it again while the panel has focus to hide it.
2. **Add a repo**: **+** in the panel's header opens the folder picker.
3. **Open a worktree**: click its row. Its session opens, and the panel comes with it.
4. **New worktree**: **+** next to a project. Type a branch name and, if you like, a display name. The row appears straight away as **creating** and becomes a normal row when setup is done.
5. **Everything else is a right-click away** (see [The menus](#the-menus)).

The panel stays on the left of whichever tab you're looking at, across every session, and keeps the width you give it. Each project is a local Git repository, each worktree is a row, and the worktree's open tabs are listed under it. Click a tab row to jump to that tab; hover a worktree row for its branch, path, line counts, ahead/behind and PR.

| You see | It means |
|---|---|
| ● green | A session is open |
| ● green, pulsing | An agent is working |
| **?** yellow | An agent is waiting for your answer |
| **✔** green · **!** red | An agent finished · failed while you were elsewhere (clears when you open its tab) |
| ○ hollow | No session yet; click to open one |
| **✱** amber | Uncommitted changes |
| **conflicts** | A merge or rebase is stuck |
| **creating** | Setup is running; the row says how long and which step |
| **needs you?** | Setup is asking a question, or has printed nothing for a minute; click to see it |
| **▾ 3** | Three sub-worktrees; click to fold them |
| **2 new** | Overseer messages for this worktree's agent since you last looked |
| **worktree gone** | Removed outside Overseer; its session is still open |
| Linear logo · PR icon | The ticket the branch names (`feat/ENG-123-…`) · the PR's state |
| 🧊 | Parked in the Ice Box |
| **done** · **failed** on a tab | That tab's agent turn ended while you were elsewhere |
| **overseer** on a tab | The session Overseer's [coordination](flows.md#coordination) messages go to |

Tab names leave out what the row already says: a GitHub pull request tab reads `#123 <title>`, without its author and repository. Hover a tab for its full name.

## The menus

### On a worktree

| Menu item | Description |
|---|---|
| **Visualize PR** | A reviewer's outline of the branch's pull request ([below](#visualize-a-pr)) |
| **Rename…** | Changes the row's label; the branch stays as it is |
| **Add to Ice Box** | Parks the worktree at the bottom; 🧊 in the header shows or hides parked ones |
| **Move under…** · **Make top-level** | Makes it a [sub-worktree](#sub-worktrees) of another, or undoes that |
| **Brief sub-worktrees…** | Sends your message to the agents of all its sub-worktrees |
| **Close session** | Stops its programs and closes its tabs at once; the files stay |
| **Delete worktree** | Removes a clean worktree at once. With uncommitted or untracked changes it first lists them, like `git status --short`, and asks you to **Force delete**. Branches are always kept, and the session closes once the directory is gone. A worktree with sub-worktrees asks whether they go too. |

### On a project

| Menu item | Description |
|---|---|
| **Change base branch…** | The branch new worktrees start from |
| **Set Linear workspace…** | Where ticket links point |
| **Remove project…** | Stops showing it. The repository, worktrees and sessions stay, and adding it again brings back its names and Ice Box. |

### On a tab

**Make overseer session** picks which of a worktree's agents gets Overseer's messages.

### On an externally removed worktree

A worktree removed outside Overseer (`wt remove`, `wt merge`, `git worktree remove`) shows as **worktree gone** while its session is still open, so you can read what its agent last said. Its menu offers only **Close session**.

## Start work from a ticket

Ask Tern's assistant, Carly:

> Create an overseer worktree for ENG-123

Overseer creates `feature/ENG-123` from the project's base branch and opens its session. Then it starts omp with:

> *Please read ticket ENG-123 to understand your task and ask any questions you have before we start*

The project is the one you're in, or name it: *"…for ENG-123 in api"*. If the branch already exists, Carly tells you instead.

## Sub-worktrees

A task's worktree can own the worktrees of its sub-tasks. When an agent splits a job into one branch per fix, the fixes sit under the task instead of scattering through the list:

```text
● Space/Website Fixxer  ▾ 3
  feature/space-website-lurker
  │ ○ fix/website-coingecko-caps
  │ ○ fix/website-drain-on-term ✱
  │ ○ fix/website-logging-levels
```

- **Agents make them.** With the [omp integration](#omp-integration), an agent in a task's worktree creates the worktrees of its sub-tasks as sub-worktrees, and can hand each one a brief so a fresh agent starts on it there. Work started from the primary checkout, or unrelated to the task, gets an ordinary top-level worktree.
- **You arrange them** with **Move under…** and **Make top-level**.
- **Each is a full worktree**, with its own session, menu, PR and ticket. Nesting is one level deep.
- **Fold them** with the `▾ N` badge. Folded, it takes the color of the sub-worktree agent that most needs you; the worktree you're in always shows.
- **Deleting the owner asks.** **Delete only this** moves its sub-worktrees to the top level; **Delete all** also removes the ones without uncommitted changes and keeps the rest. Parking the owner in the Ice Box parks them too.

The hierarchy lives in Git config (Worktrunk's per-branch `overseer-parent` variable), so `wt`, agents and every Overseer install see the same tree. Branches stay independent: nesting groups the panel, not Git history.

## Coordination

With the [omp integration](#omp-integration) and pi-intercom, the agents of a task and its sub-worktrees stay aware of each other, whichever side is doing the work:

- **Before each prompt**, an agent hears what changed in its family since it last looked: new commits, files both branches touch, and what the others are working on or just finished.
- **Overseer wakes the task's agent** when a sub-worktree's agent is waiting for an answer, its turn failed, or it finished with a pull request open. A sub-worktree built on its parent's branch hears when that branch moves, so it can rebase.
- **Agents message each other** with the `overseer_family` tool, or ask and wait for an answer over pi-intercom.
- **You brief them all** with **Brief sub-worktrees…**.
- **The panel shows it:** **N new** counts Overseer's messages since you last opened that session, and a sub-worktree's row says when its agent last checked in.

Messages inform; they never act. Overseer decides when to send one from Tern's agent states and Git, never from what an agent wrote, and sends a session at most one a minute. Each worktree has one **overseer session** that receives them: its only omp session, or the one you pick with **Make overseer session**.

## Visualize a PR

Overseer fetches the PR's metadata and full diff from GitHub, then has omp write an outline following HumanLayer's [visual-pr](https://github.com/humanlayer/skills) skill: why the change exists, what to watch for, and Mermaid diagrams, file-tree diffs and pseudocode. The local checkout only adds surrounding context. It takes a minute or two, and unchanged PRs reopen instantly from a cache.

It's **read-only by construction**: omp runs with only read/grep/glob tools and no shell, so it can't push, comment or edit anything.

## omp integration

With [omp](https://omp.sh) installed, there's nothing to set up: each time Overseer loads, it links its omp half into omp. That half is `omp/`, an omp plugin package named `overseer-omp`:

| Piece | What it does |
|---|---|
| `overseer-subworktree` skill | Teaches agents to create worktrees the right way: sub-worktrees for their task's sub-tasks, top-level worktrees otherwise |
| `overseer-worktrees` rule | Stops an agent that types a raw `git worktree add` or `wt switch --create` and points it at the skill |
| `overseer-rails` extension | Keeps the agents of a worktree family informed (needs pi-intercom) |

They load in omp sessions started after the link (`/new` restarts one); `omp plugin list` shows it. `omp plugin disable overseer-omp` turns them off, and Overseer leaves it that way. Overseer also removes the symlinks into `~/.omp/agent` that earlier versions had you make, so omp doesn't load them twice. Using omp without Tern? Link it yourself: `omp plugin link "$PWD/omp"`.

## Architecture

- **Worktrunk owns the worktrees.** Overseer calls `wt` and never passes `--yes` or `--no-hooks`, and passes `--force` only after you confirm the listed changes. Your hooks and approval prompts run in a background tab you can open from the row.
- **Tern owns the sessions.** Overseer remembers which session belongs to which worktree and switches between them.
- **Git config holds the hierarchy.** Sub-worktree links (`overseer-parent`) and each worktree's overseer session (`overseer-lead`) are Worktrunk branch variables that agents read too.
- **The panel follows you.** Tern plugins can't pin a sidebar to the window, so Overseer keeps one Projects panel per window and moves it into whichever tab you're on.
- **Everything else is a read.** PR badges, ticket links, agent status and change counts come from `gh`, `linear`, `git` and Tern's own agent state.

## Safety

- Creation never fetches, pulls, or checks out anything in your primary checkout.
- Deletion rechecks the repository and branch right before it runs, and cleanliness too unless you confirmed a force delete. It never deletes a branch. The primary checkout can't be deleted.
- Ignored files are removed with a deleted worktree's directory, so copy out anything you need first.
- Branch names and paths are passed as arguments, never as shell text.
- What agents say to each other is information only. Nothing Overseer does (creating, deleting, merging, approving) is triggered by an agent's message.

## Troubleshooting

- **A new worktree seems stuck.** Its row says what setup is doing. **needs you?** means setup is asking a question, often a Worktrunk hook approval, or has printed nothing for a minute. Click the row to open its setup tab.
- **What did Overseer just do?** **Show Overseer's operation log** in the command palette lists every creation, deletion, brief and coordination message, with when it happened, how long it took and how it ended. The same entries are in Tern's log (`~/Library/Logs/Tern/tern.log`; search for `Overseer:`).
- **The panel froze for a few seconds.** Tern pauses a plugin's timers when one of its callbacks runs too long (5 s, doubling on repeats). Overseer logs `Tern paused its … hook`, keeps the panel following you and finishes jobs from a second clock meanwhile, and catches up when the pause ends. A draw that comes close logs `slow draw` with where its time went.
- **A creation or deletion seems stuck after its tab says it's done.** Overseer restarts its tick when it's more than 5 s late; each restart shows in the operation log as `tick restarted`, with how long it had stopped.
- **Agents aren't coordinating.** Check that pi-intercom is installed for omp, that `omp plugin list` shows `overseer-omp`, and that the agents' sessions started after Overseer linked it (`/new` restarts one). With several omp sessions in a worktree, pick one with **Make overseer session**.
