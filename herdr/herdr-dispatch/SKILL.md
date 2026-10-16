---
name: herdr-dispatch
description: Spawn a sibling Claude Code agent in a new Herdr pane and hand it a self-contained task brief. Trigger on /herdr-dispatch, "开个 pane 让另一个 agent 做", "分一个子任务过去", "把这个任务派给子 agent", or any ask to run a task in a parallel agent pane. Requires HERDR_ENV=1. Args are natural language — an optional model (opus/sonnet/haiku/fable; default opus = Opus 5) plus the task; when no task is stated, derive it from the current conversation. Every dispatched agent is labelled with a shared group prefix taken from the dispatching tab, so the user can tell which sessions/tabs/panes belong to this main agent; a new tab must always carry it.
---

# herdr-dispatch — hand a task to a fresh agent in a Herdr pane

Turn the current conversation's agreed direction into a complete brief, open a sibling Herdr pane, launch Claude Code on the requested model, and submit the brief. The child runs independently end-to-end; you report the handle and step back.

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
   herdr tab list --workspace "${HERDR_PANE_ID%%:*}"   # take the row whose tab_id matches this pane's tab
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

- **Agent / pane label — always**: `<prefix>/<task-label>`, e.g. `api-main/api412-flag-flip`. This is what makes a flat listing groupable: `herdr agent list | grep 'api-main/'`.
- **Tab label — whenever the dispatch lands in a NEW tab**: `[<prefix>] <task-label>`, e.g. `[api-main] api412-flag-flip`, matching the bracketed convention already in use.

## 2. Place the agent — same-tab right column by default, a new tab only when it must

The dispatching (main) pane permanently keeps the left side. Every dispatched agent lives in ONE right-hand column, stacked vertically. Decide the split target from the live layout, not from memory:

```bash
herdr pane layout --pane "$HERDR_PANE_ID"
```

- **No pane to the right of the main pane** (no rect with `x` > the main pane's `x`) → first dispatch: split the main pane right.

  ```bash
  herdr pane split --current --direction right --no-focus
  ```

- **Right column exists** → split the **bottom-most right-column pane** (largest `rect.y` among panes with `x` > main's) downward, so newcomers stack under it. Never split the main pane again — that would carve a second column or shrink the left side.

  ```bash
  herdr pane split --pane <bottom-right-pane-id> --direction down --no-focus
  ```

Read `result.pane.pane_id` from the JSON — never construct or guess an ID. Label it with the prefix from §1:

```bash
herdr pane rename <pane_id> "<prefix>/<task-label>"
```

### New tab — only when the right column cannot take another pane

Stay in the current tab by default; a dispatched pane beside its parent is easier to follow. Open a new tab only when **the user asked for one**, or **the right column already holds 3 panes** (a fourth would be too short to read).

```bash
herdr tab create --workspace "${HERDR_PANE_ID%%:*}" --label "[<prefix>] <task-label>" --cwd <repo path> --no-focus
```

Read `result.tab.tab_id` and `result.root_pane.pane_id` from the JSON. The tab is born labelled, so it needs no rename — but still label the pane, so the prefix also shows up in `herdr agent list`:

```bash
herdr pane rename <root_pane_id> "<prefix>/<task-label>"
```

A new tab is the one case where the grouping is otherwise invisible: nothing on screen ties that tab back to this session. The bracketed prefix is what ties it. Never open an unprefixed tab.

## 3. Start the agent — use the `herdr agent` facade

Herdr ≥0.7.5 ships a live-agent facade (`herdr agent`) with named start, **atomic prompt**, and server-owned wait. Prefer it over raw `pane run` — it validates startup and closes the submit race entirely. Native agent args go after `--`:

```bash
herdr agent start <task-label> --kind claude --pane <pane_id> --timeout 60000 -- --model <alias>
```

Subsequent `agent` commands target `<task-label>` (or the pane id). If startup needs confirming: `herdr agent wait <task-label> --until idle --timeout 60000`.

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
herdr agent prompt <task-label> "<one-line brief>" --wait --until working --timeout 30000
```

`agent prompt` is atomic (text + submit as one server-owned operation) and `--until working` confirms the child actually accepted the task.

**Timeout ≠ failure.** A `--wait` timeout only means the status didn't flip within the window — the text is usually already delivered/queued (e.g. the child was still finishing its previous turn). NEVER blind-resend: check first, or you double-deliver the brief:

```bash
herdr agent get <task-label>    # title/status changed → it was delivered; done
herdr agent read <task-label> --source recent-unwrapped --lines 15   # else look at the composer
```

**Composer text is not user input.** Claude Code renders *suggested-next-prompt placeholder text* (ghost text) in the composer line, and a `pane read` cannot distinguish it from typed text. Never interpret composer-line text as something the user typed, never report it to the user as "you typed…", and never press Enter to "submit" text you didn't inject — Enter on a placeholder is a no-op (text stays, status stays `idle`), which is easy to misdiagnose as a dropped keystroke. The only composer text you may complete with an Enter is a brief **you yourself just injected** whose auto-submit timed out (verify via `agent get` first: still `idle` + your own text visible → one Enter finishes the submission).

## 6. Report and step back

Tell the user: pane label + pane ID (and where it sits — same tab, or a new tab and its label), the **group prefix** with the one-liner that lists everything under it (`herdr agent list | grep '<prefix>/'`), model, one-line task summary, and what was handed over (decisions pinned, constraints, acceptance bar). If the prefix came from the pane-id fallback, say so. Then leave it alone — it's an independent session. On request:

```bash
herdr agent read <task-label> --source recent-unwrapped --lines 40   # progress
herdr agent wait <task-label> --until idle --until done --timeout 600000   # block for completion
```

Follow-ups are another `agent prompt` (same confirm-uptake discipline). Never close panes you didn't create.

## Fallback (no `herdr agent` group in this build)

1. `herdr pane run <pane_id> "claude --model <alias>"`, then poll `herdr pane get <pane_id>` (~2s, up to 60s) for `agent_status: idle`.
2. `herdr pane run <pane_id> "<brief>"` — **known gotcha:** a long brief lands as a bracketed paste in the composer and the trailing Enter is consumed; the pane shows `[Pasted text #N]` and status stays `idle`. Verify with `pane get`; if still idle, `herdr pane send-keys <pane_id> enter` and re-check. Only report success once status is `working`.
