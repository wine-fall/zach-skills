---
name: herdr-remote-dispatch
description: Spawn a Claude Code agent on a REMOTE Herdr machine (a saved SSH machine) and hand it a self-contained task brief. Trigger on /herdr-remote-dispatch, "派到云端机器上做", "在远端机器开个 agent", "dispatch this to the cloud box", or any ask to run a task on a saved Herdr machine rather than locally. Requires HERDR_ENV=1 and at least one machine in `herdr machine list`. Args are natural language — an optional machine label, an optional model (opus/sonnet/haiku/fable; default opus = Opus 5), and the task; when no task is stated, derive it from the current conversation. For local panes use herdr-dispatch instead.
---

# herdr-remote-dispatch — hand a task to an agent on a remote Herdr machine

Same contract as `herdr-dispatch`, except the agent lands on a **saved SSH machine** instead of beside you. Every control command carries `--machine`, the geometry rules change because you are filling a remote workspace rather than splitting your own, and a fresh remote checkout walks through first-run prompts that a local pane never shows.

Read `herdr-dispatch` for the parts that are identical: prefix normalization, how to compose a brief, and the composer-text warning. This file only states what differs, plus the steps you must not skip.

## 0. Preconditions

```bash
test "${HERDR_ENV:-}" = 1
herdr machine list
```

Not in Herdr → say so and stop. No saved machines → say so and stop; do not offer to add one as part of a dispatch.

Pick the machine from the args when one is named, otherwise the single enabled machine. **Two or more enabled machines and no name in the args → ask which one.** Dispatching long work to the wrong host is not something the user can undo by reading a label afterwards.

Hold the label in a variable and use it everywhere. Quote it: labels can contain spaces.

```bash
M="<machine-label>"
```

Confirm the machine actually answers before building anything on it — a saved profile can be enabled and unreachable:

```bash
herdr --machine "$M" workspace list
```

An error here is the whole report. Say the machine is unreachable and stop; do not fall back to a local pane, because the user asked for the remote one.

## 1. Prefix

Identical to `herdr-dispatch` §1, with one addition: reuse the prefix this conversation already dispatched under, local or remote. One session is one group even when its agents are split across machines — that is exactly when a shared prefix earns its keep.

Resolve it from the **local** dispatching tab, never from the remote workspace:

```bash
herdr tab list --workspace "${HERDR_PANE_ID%%:*}"
```

## 2. Place the agent on the remote machine

You are not splitting your own pane here, so the local "main pane keeps the left side" rule does not apply. You are filling a remote workspace that already has its own panes, which may belong to the user.

Read the live remote layout; never assume `w1:p1` is free:

```bash
herdr --machine "$M" workspace list
herdr --machine "$M" tab list --workspace <workspace-id>
herdr --machine "$M" pane list --workspace <workspace-id>
```

### Four panes to a tab, as a 2×2 grid

A dispatch tab holds **at most four agents, laid out 左上 / 右上 / 左下 / 右下** (top-left / top-right / bottom-left / bottom-right). Agents arrive one at a time, so each new one takes the next free quadrant. The split sequence below was verified on a live remote tab and produces four even panes; any other order gives you columns or stripes.

| slot | how it is created |
| --- | --- |
| 左上 | the tab's root pane — it already exists when you create the tab |
| 右上 | `pane split --pane <左上> --direction right` |
| 左下 | `pane split --pane <左上> --direction down` |
| 右下 | `pane split --pane <右上> --direction down` |

**Identify the quadrants from geometry, never from the order ids were handed to you.** Pane ids are opaque and a pane that moved gets a new one. Read the layout and sort by rect:

```bash
herdr --machine "$M" pane layout --pane <any-pane-in-the-tab>
```

`rect.x == 0` is left, the larger `x` is right; `rect.y == 0` is top, the larger `y` is bottom. The free quadrant is the one no pane occupies.

**A fifth agent means a new tab.** Do not split a quadrant again: four panes already divide a terminal into 60×20 cells, and a fifth makes every pane unreadable.

**Get the layout right when you create the pane — it cannot be rearranged afterwards.** The split tree is fixed by creation order, and `pane move` with a `--target-pane` in the SAME tab is a no-op: it returns `changed: false` and nothing shifts. The only rearrangement herdr actually performs is a move **between** tabs. So rescuing a badly stacked tab means moving each pane into a fresh tab in the right order (the first becomes that tab's root, i.e. 左上), which is disruptive when agents are mid-task. Cheaper to split correctly the first time.

```bash
herdr --machine "$M" tab create --workspace <workspace-id> --label "[<prefix>] <task-label>" --cwd <remote path> --no-focus
```

**Never take over an existing remote pane.** A pane with a shell in it may be the user's session, a login flow mid-prompt, or a tunnel. If the target tab is not one you created, create your own tab instead of filling its gaps.

### Sync the repo before you dispatch into it

The agent must start on current code, in its own worktree. Do this yourself, before the pane exists, so the child opens directly in the right place.

**1. Sync the main checkout.** Not a worktree — the main one. `git worktree list` prints it first.

```bash
herdr --machine "$M" pane run <existing-pane> "cd <repo> && git status --porcelain | head -5 && git pull --ff-only"
```

**Stop if the main checkout is dirty.** A pull onto uncommitted work either refuses or leaves a mess the child then inherits, and those changes are the user's, not yours. Report it and ask.

**2. Create the worktree from fresh `origin/main`,** following the repo's own convention rather than inventing one. Many repos state it in `AGENTS.md` / `CLAUDE.md` (for example: new work goes in `.claude/worktrees/<branch>`, branched from fresh `origin/main`). Take the branch-naming shape from the worktrees already there (`git worktree list`) instead of guessing.

```bash
herdr --machine "$M" pane run <existing-pane> "cd <repo> && git worktree add .claude/worktrees/<branch> -b <branch> origin/main"
```

**3. Dispatch with `--cwd` set to the worktree**, not the main checkout. Verified: a worktree carries the full tree, so the project's skills, `AGENTS.md` and `.mcp.json` all load from it — the child gets the repo's conventions and edits files that cannot collide with another agent's.

**A fresh worktree is not ready to run anything yet.** Most work repos document a per-worktree setup step, and on a remote box there are often machine-specific steps that must come first (registry auth before any install, prebuilt shared packages before any test), or the failures are misleading. Read the repo's own setup instructions and put whichever steps the task needs in the brief rather than letting the child rediscover them.

**Read-only work skips all of this.** An investigation that only reads code belongs in the main checkout; a worktree for it is hundreds of MB and a branch nobody will merge.

### The cwd decides what the agent knows

`--cwd` is not a convenience. Claude Code loads project skills, `AGENTS.md` / `CLAUDE.md`, and `.mcp.json` **from the working directory**, so the cwd silently decides which conventions the child has. Start it one level too high and it loads none of them, reports no error, and quietly works like a generic agent in a repo that has a whole set of skills telling it how to work.

So: **the cwd is the repository the task is about.** Never the home directory, never the parent that holds the checkouts, never a path you assumed.

List what is actually on the machine instead of guessing, and confirm the repo matches the task:

```bash
herdr --machine "$M" pane run <existing-pane> "ls <checkouts-dir> && for r in <checkouts-dir>/*/; do echo "$r $(ls $r/.claude/skills 2>/dev/null | wc -l) skills"; done"
```

Two further consequences worth planning for:

- **`--cwd` takes a REMOTE path.** A macOS path like `/Users/...` does not exist on a Linux box. A local path here is the single most common way a remote dispatch wastes its first ten minutes.
- **The first-run prompt chain is per-repo.** A repo carrying `.mcp.json` adds the MCP approval prompt after the folder-trust one; a repo without it shows only folder trust. This is why §3 says read the pane between keystrokes rather than sending a fixed sequence.

When a task spans two repos, dispatch in the one it will mostly edit and name the other's absolute remote path in the brief. One agent cannot hold two sets of project instructions, and the second repo's conventions will not load just because its path is mentioned.

### Personal repos: work on the box, push from the laptop

The remote box may be a **shared work machine**: other people can get a root shell on it, and its `gh` is logged in as the user's **work** GitHub account. A task on one of the user's **personal open-source repos** (the ones kept in their personal-repo directory on the laptop) therefore goes to a separate personal directory on the box, never the work checkouts directory. The personal flow keeps every personal credential on the laptop and moves only commits. It relies on two small scripts the user keeps on their own machines (not part of this repo):

- **`personal-clone <owner/repo>`** (on the box) clones a personal repo anonymously into the box's personal directory and points it at a personal git config: a throwaway identity (`box <box@invalid>`), GitHub credentials cleared, every push URL rewritten to nowhere, and a `gh` that refuses to run there.
- **`box-take`** (on the laptop) lists branches waiting on the box, fetches one straight out of the box's repo over ssh, rebases it onto upstream, re-authors every commit as the user's personal account, and stops before pushing.

The flow:

1. **The repo must already be cloned on the box**, anonymously: `personal-clone <owner/repo>` (one time).
2. **Sync and worktree as usual, but the branch is `box/<slug>` and the worktree is `.worktrees/<slug>`** inside the repo, so `box-take` can find it. `--cwd` is that worktree.
3. **The brief must say**: commit everything, do not push or open a PR (it will fail, by design, and retrying wastes time), run every test and review the change needs **on the box** (the full suite at low priority, e.g. `CI=true nice -n 10`, since the box is shared), and finish with a short handoff (what changed, what was tested, what is left). Also tell it the box is Linux; anything that truly needs macOS is listed in the handoff, not run.
4. **Handover happens on the laptop**, from the laptop's copy of the repo: `box-take` lists waiting branches, `box-take <slug>` brings one over, ready to push. After it is merged, `box-take --cleanup <slug>`.
5. **One principle: the laptop only does the light part — pull the change down (`box-take`), push it, open the PR. Everything else happens on the box.** That means no test suites, no `codex review`, no sims or benchmarks, no `install`/`build` — not even "one quick check" after `box-take`, whatever its closing hint says. Reading the diff (`git log`, `git diff`, `grep` for personal data) is fine. Why: a full local test run alongside a `codex review` once drove the laptop's load average past 300 and froze it, and the user's rule is explicit: "任何的测试都要放在远端去跑，本地只做提交代码和开 PR". If a review or test turns something up, send it back to the box agent to fix and re-test there, then `box-take` again. Codex's own sandbox can fail on such a box (bwrap), so a box review feeds the diff over stdin: `git diff origin/main...HEAD | codex review -c model="<model>" -`. Something that genuinely needs macOS: ask the user first; never run it on your own.

Never "fix" a failed push from the box by giving it credentials, and never turn ssh agent forwarding back on for the box's SSH host: with forwarding on, the box could authenticate to GitHub as the personal account through the laptop's agent, which is exactly how this was found.

## 3. Start the agent, then clear the first-run prompts

```bash
herdr --machine "$M" agent start <task-label> --kind claude --pane <pane_id> --timeout 60000 -- --model <alias>
```

**Expect this to come back `blocked`, not `idle`, the first time a cwd is used on that machine.** `agent start` reports `agent_not_ready`, and `agent prompt` then refuses with `agent_blocked`. That is not a failure — Claude Code is asking something. Verified on a fresh remote checkout, in this order:

1. **Folder trust** — "Is this a project you created or one you trust?" with `No, exit` preselected. Answering requires moving the selection:

   ```bash
   herdr --machine "$M" pane send-keys <pane_id> down enter
   ```

2. **MCP server approval** — appears when the repo ships `.mcp.json`. The safe default, `Continue without using this MCP server`, is already selected:

   ```bash
   herdr --machine "$M" pane send-keys <pane_id> enter
   ```

**Read the pane between every keystroke.** Do not send a blind `down enter` twice; the second prompt's options are in a different order and the wrong choice is one keypress away.

```bash
herdr --machine "$M" pane read <pane_id>
```

Only answer prompts that are Claude Code's own first-run questions. A prompt asking to trust a directory the user did not name, or to enable an MCP server that reaches a live system, goes back to the user instead.

Then confirm the agent is actually ready:

```bash
herdr --machine "$M" agent get <task-label>     # want idle or done
```

## 4. Brief and submit

Compose the brief exactly as `herdr-dispatch` §4 describes, with two remote-specific additions:

- **Every path in the brief must be a remote path.** The child reads files on that machine, and a local path sends it hunting for files that are not there.
- **State the machine's limits** when the task could run into them, so the child does not burn turns rediscovering them: what it cannot reach, what is installed, whether a tunnel the task depends on is held open by someone's laptop and will drop.

```bash
herdr --machine "$M" agent prompt <task-label> "<one-line brief>" --wait --until working --timeout 30000
```

Timeout handling and the composer-text warning are identical to `herdr-dispatch` §5. Check with `agent get` before ever resending.

## 5. Report and step back

Report what `herdr-dispatch` §6 asks for, plus:

- **The machine label**, first, because the user cannot see which host the pane is on from the pane name alone.
- **Where it sits remotely**: workspace, tab (and whether you created the tab), pane id, and the current pane count in that tab.
- **The one-liner to look in on it**, with the machine flag spelled out:

  ```bash
  herdr --machine "<label>" agent list
  herdr --machine "<label>" agent read <task-label> --source recent-unwrapped --lines 40
  ```

- **Any dependency that outlives your attention**: a reverse tunnel, a port-forward, or a credential with a known expiry. Say what breaks when it drops, since the user is the only one who can restore it.

Then leave it alone.

## Things that bite on a remote machine and not locally

- **Replacing the host kills every pane.** When the machine is a container that gets redeployed, a redeploy replaces it: agents die, files on persistent storage survive. Brief the child to commit early if its host can be replaced under it.
- **Remote IDs are a separate namespace.** `w1:p1` exists on both machines and means different panes. Never pass a remote id to a local command or the reverse.
- **`herdr agent list` without `--machine` lists the local machine only.** Listing a dispatch group that spans machines takes one call per machine.
- **The remote agent's credentials are the remote machine's.** It does not inherit your local logins, API keys, or ssh agent. If the task needs an identity, confirm the machine already holds it rather than briefing the child to log in.
- **Round-trip latency is real.** Reading a remote pane is slower than a local one; prefer `agent wait` over polling loops.
