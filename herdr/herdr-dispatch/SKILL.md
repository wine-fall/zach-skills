---
name: herdr-dispatch
description: Spawn a Claude Code agent in a dedicated Herdr dispatch tab (up to four panes in a 2×2 grid per tab) and hand it a self-contained task brief. Trigger on /herdr-dispatch, "开个 pane 让另一个 agent 做", "分一个子任务过去", "把这个任务派给子 agent", or any ask to run a task in a parallel agent pane. Requires HERDR_ENV=1. Args are natural language — an optional model (opus/sonnet/haiku/fable; default opus = Opus 5) plus the task; when no task is stated, derive it from the current conversation. Every dispatched agent is labelled with a shared group prefix taken from the dispatching tab, so the user can tell which sessions/tabs/panes belong to this main agent; a new tab must always carry it.
---

# herdr-dispatch — hand a task to a fresh agent in a Herdr pane

Turn the current conversation's agreed direction into a complete brief, place the child in a dedicated dispatch tab (never split the main session's tab), launch Claude Code on the requested model, and submit the brief. The child runs independently end-to-end; you report the handle and step back.

## 0. Preconditions & argument parsing

```bash
test "${HERDR_ENV:-}" = 1
```

Not in Herdr → say so and stop. Do not fall back to controlling Herdr from outside.

Parse the natural-language args:

- **Model** — a bare model word anywhere in the args (`opus` / `opus 5` / `sonnet` / `haiku` / `fable`) → pass its alias to `--model`. **No model stated → `opus`** (Opus 5). Always pass `--model` explicitly: the user's default session model drifts (they may have `/model`'d to something else) and must never leak into the child.
- **Task** — everything else. **No task stated → continuation semantics**: the task is the direction already agreed in this conversation; do not re-open it, brief it.

## 1. Establish the dispatch group prefix

Every agent this session dispatches carries ONE shared prefix, so the user can look at a flat `herdr agent list` or a crowded tab bar and tell at a glance which sessions, tabs, and panes came from this main agent.

Resolve it once per session, in this order:

1. **A prefix named in the args** ("前缀用 xxx") → use it verbatim.
2. **The dispatching tab's label, brackets stripped** — read it, never assume it:

   ```bash
   herdr pane current --current   # resolve the calling pane, workspace and tab; do not use UI focus
   herdr tab list --workspace <returned-workspace-id>   # match the returned tab_id
   ```

   `[api-main]` → prefix `api-main`. This is the default: it is the user's own naming, it names the main session rather than the moment, and it does not drift.
3. **Tab unlabeled** → fall back to the dispatching pane id minus the workspace part (`w3:p1B` → `p1B`), and say so in the hand-back, so the user can label the tab if they want something readable.

**Cap: 11 characters.** A prefix is a grouping handle, not a description — it rides in front of every label, so it stays short. Normalize whatever the source gives you, by this exact recipe, so the same session always produces the same prefix:

1. Strip brackets and surrounding whitespace; lowercase; collapse each run of internal whitespace to a single `-`.
2. **≤ 11 characters → done.**
3. Otherwise drop whole `-`-separated segments off the end until it fits — a truncated word is unreadable, a dropped word is not.
4. If the first segment alone still exceeds 11, hard-cut it to 11 characters.
5. Strip any trailing `-`.

Count **characters, not bytes** — a CJK label is three bytes per character and would otherwise be cut to nonsense.

| Source label | Prefix |
| --- | --- |
| `[api-main]` | `api-main` |
| `[cache layer rework]` | `cache-layer` |
| `[roadmap draft]` | `roadmap` |
| `Data Export — 剩余问题讲解` | `data-export` |

The cap applies to a prefix **you were given in the args** too. Shorten it the same way and say what you shortened it to, rather than silently carrying an over-long one.

**Stability beats freshness.** If this conversation has already dispatched under a prefix, reuse that exact string for every later dispatch — even if the tab has since been renamed. A prefix that changes mid-session splits one group into two, which is the whole thing it exists to prevent.

**Never derive the prefix from the task.** One main session dispatches many unrelated tasks; a task-shaped prefix stops grouping them the moment the topic moves on.

Apply it in two places:

- **Pane label — always**: `<prefix>/<task-label>`, e.g. `api-main/api412-flag-flip`. This makes a flat pane listing groupable: `herdr pane list --workspace <caller-workspace-id> | grep -F 'api-main/'`. The live agent name is a separate API handle; see §3.
- **Dispatch tab label — always**: `[<prefix>] d<N>`, e.g. `[api-main] d1`, `[api-main] d2`. A tab holds several tasks, so name it for the session and batch number, not the first task. Keep the existing 11-character prefix cap; do not repeat the full session title.

## 2. Place the agent — dedicated dispatch tabs, at most four panes each

**Never split the main session's tab.** The first child goes into a NEW tab in the caller's workspace. Later children fill this session's dispatch tabs to at most four panes in a balanced 2×2 grid; only then create the next tab. This skill's topology overrides the official Herdr skill's generic same-tab default.

Resolve the caller with `herdr pane current --current`. Preserve the task's repo cwd explicitly on **both tab creation and every split**; default to the caller's `$PWD`. Always use `--no-focus`.

Track the main pane/tab IDs, shared prefix, created dispatch tab IDs, batch numbers, and child pane IDs in this conversation. Reuse only dispatch tabs known to belong to this main session, in creation order. **A matching shortened prefix alone is not proof of ownership.** If resuming without reliable ownership records, create a new dispatch tab rather than adopting an unrelated tab. Pick an unused batch number from the live tab list; do not rename a colliding tab.

### Create a dispatch tab

When there is no eligible tab with room (or the user explicitly requests a fresh tab):

```bash
herdr tab list --workspace <caller-workspace-id>
herdr tab create --workspace <caller-workspace-id> --label "[<prefix>] d<N>" --cwd <repo-path> --no-focus
```

Read `result.tab.tab_id` and `result.root_pane.pane_id` from the response. The root pane hosts the first child; **do not create three empty panes in advance**.

### Fill an existing dispatch tab

Before every allocation, inspect its live layout using one of its known pane IDs:

```bash
herdr pane layout --pane <dispatch-pane-id>
```

Count **all live panes**, not just working agents: idle/done/blocked panes still occupy slots. Never overwrite an existing pane or close it to make room. Inspect `rect.x`, `rect.y`, width and height, not pane-ID order. Allocate sequentially, re-reading layout after each split:

| Existing panes | Action | Result |
| --- | --- | --- |
| 1 | Split the root pane **right**, ratio 0.5 | Two equal full-height columns |
| 2, two full-height columns | Split the **left** pane **down**, ratio 0.5 | Upper-left, lower-left, full-height right |
| 3, left stacked and right full-height | Split the **right full-height** pane **down**, ratio 0.5 | Upper-left, lower-left, upper-right, lower-right |
| 4 or more | Do not split; try the next owned tab, or create one | Never exceed four panes |

```bash
herdr pane split --pane <chosen-dispatch-pane-id> --direction <right-or-down> --ratio 0.5 --cwd <repo-path> --no-focus
```

Use `result.pane.pane_id` as the new child's ID. If panes were manually closed/rearranged, only split when the live geometry clearly allows a missing quadrant to be restored (e.g. split a full-height column down). Otherwise skip that tab and create a fresh one; never squeeze a fifth pane or produce stacked thirds. A geometry/size error is not permission to fall back to the main tab. Inspect state after an ambiguous failure before retrying, to avoid duplicate panes. Serialize allocations from this session so two dispatches do not both allocate the last slot.

### Label every child

Whether the child uses a new tab's root pane or a split pane:

```bash
herdr pane rename <pane_id> "<prefix>/<task-label>"
```

Keep `<task-label>` short and specific. Every pane carries the session prefix and every dispatch tab is born with `[<prefix>] d<N>`. Record the returned IDs and actual position for the hand-back.

## 3. Start the agent — use the `herdr agent` facade

Herdr ≥0.7.5 ships a live-agent facade (`herdr agent`) with named start, **atomic prompt**, and server-owned wait. Prefer it over raw `pane run` — it validates startup and closes the submit race entirely. Native agent args go after `--`:

```bash
herdr agent start <agent-name> --kind claude --pane <pane_id> --timeout 60000 -- --model <alias>
```

The official skill restricts live agent names to `[a-z][a-z0-9_-]{0,31}`: `<prefix>/<task-label>` is a **pane label**, not a valid agent name. Choose a unique short ASCII `<agent-name>` related to the group/task (e.g. `api-main-review`), checking `herdr agent list` for collisions. Do not drop or transliterate away the readable pane/tab labels merely to satisfy this API constraint. Use the returned **pane ID** for subsequent agent commands so duplicate task names cannot target the wrong child. If startup needs confirming: `herdr agent wait <pane_id> --until idle --timeout 60000`.

**Version drift note:** installed CLI is the authority, packaged docs lag. If `herdr agent` is absent in this build, print the real surface with the bare command groups (`herdr agent`, `herdr pane` — no subcommand) and fall back to §Fallback below.

## 4. Compose the brief — the child has zero context

The child is a fresh session: it cannot see this conversation. The brief must be **self-contained**, written as **one single line** (raw newlines in submitted text risk odd composer behavior). Checklist:

- **Lead with a slash command when one fits** the task end-to-end (e.g. `/ship-it <brief>`), so the child enters the right workflow immediately.
- **Task** — what to deliver, and that design is already settled ("按此执行不必重开设计") when it is.
- **Background facts** with anchors: file:line, PR numbers, URLs, measured results. Facts you verified, stated as verified.
- **Decisions already made** — enumerate them so the child doesn't re-litigate.
- **Hard constraints** — what must NOT be touched, known hazards, sequencing.
- **Acceptance bar** — what "done" means, concretely.

## 5. Submit atomically and confirm uptake

```bash
herdr agent prompt <pane_id> "<one-line brief>" --wait --until working --timeout 30000
```

`agent prompt` is atomic (text + submit as one server-owned operation) and `--until working` confirms the child actually accepted the task.

**Timeout ≠ failure.** A `--wait` timeout only means the status didn't flip within the window — the text is usually already delivered/queued (e.g. the child was still finishing its previous turn). NEVER blind-resend: check first, or you double-deliver the brief:

```bash
herdr agent get <pane_id>    # title/status changed → it was delivered; done
herdr agent read <pane_id> --source recent-unwrapped --lines 15   # else look at the composer
```

**Composer text is not user input.** Claude Code renders *suggested-next-prompt placeholder text* (ghost text) in the composer line, and a `pane read` cannot distinguish it from typed text. Never interpret composer-line text as something the user typed, never report it to the user as "you typed…", and never press Enter to "submit" text you didn't inject — Enter on a placeholder is a no-op (text stays, status stays `idle`), which is easy to misdiagnose as a dropped keystroke. The only composer text you may complete with an Enter is a brief **you yourself just injected** whose auto-submit timed out (verify via `agent get` first: still `idle` + your own text visible → one Enter finishes the submission).

## 6. Report and step back

Tell the user: pane label + pane ID, dispatch tab label + tab ID and actual position (left/right while partially filled, or upper-left/lower-left/upper-right/lower-right), the **group prefix** with the one-liner that lists its labelled panes (`herdr pane list --workspace <caller-workspace-id> | grep -F '<prefix>/'`), model, one-line task summary, and what was handed over (decisions pinned, constraints, acceptance bar). If the prefix came from the pane-id fallback, say so. Then leave it alone — it's an independent session. On request:

```bash
herdr agent read <pane_id> --source recent-unwrapped --lines 40   # progress
herdr agent wait <pane_id> --until idle --until done --timeout 600000   # block for completion
```

Follow-ups are another `agent prompt` (same confirm-uptake discipline). Never close panes you didn't create.

## Fallback (no `herdr agent` group in this build)

1. `herdr pane run <pane_id> "claude --model <alias>"`, then poll `herdr pane get <pane_id>` (~2s, up to 60s) for `agent_status: idle`.
2. `herdr pane run <pane_id> "<brief>"` — **known gotcha:** a long brief lands as a bracketed paste in the composer and the trailing Enter is consumed; the pane shows `[Pasted text #N]` and status stays `idle`. Verify with `pane get`; if still idle, `herdr pane send-keys <pane_id> enter` and re-check. Only report success once status is `working`.
