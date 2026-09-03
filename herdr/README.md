# herdr

Hand work off to parallel Claude Code sessions inside [Herdr](https://herdr.dev) — a terminal multiplexer built for coding agents — and watch them through to the finish.

Both skills require running inside Herdr (`HERDR_ENV=1`); otherwise they stop, rather than try to drive Herdr from the outside.

| Skill | What it does |
| --- | --- |
| [`herdr-dispatch`](herdr-dispatch/) | Turns a direction already settled in the current session into a self-contained brief, opens a sibling pane, starts a fresh Claude Code in it, delivers the brief atomically, then reports a handle and lets go. |
| [`herdr-monitor`](herdr-monitor/) | Watches the agents this session dispatched: when one stops, read it, decide, push it onward. Only two things send it back to a human — the child needs a permission it cannot grant itself, or a genuinely major / irreversible decision is on the table. |

## How the two divide up

`dispatch`'s contract ends at "here is the handle" — **it never chains into `monitor` on its own**. Watching is a separate, explicit request (`/herdr-monitor`). That boundary is written into section 0 of `herdr-monitor` deliberately: dispatching and supervising are two different jobs, and fusing them makes every dispatch drag along a background monitor nobody asked for.

## Shared traps

- **The group prefix.** Every agent dispatched by one main session shares a prefix taken from the dispatching tab's label, so `herdr agent list | grep '<prefix>/'` pulls them all back in one go. The prefix is not derived from the task name — tasks change, the session does not.
- **Delivery must be atomic.** Use `herdr agent prompt --wait --until working`, not hand-rolled send-keys. A `--wait` timeout does not mean the prompt failed to land: check `agent get` first, because blindly resending delivers it twice.
- **Text in the composer is not user input.** Claude Code renders ghost hint text on the input line, and `read` cannot tell the difference. Do not treat it as something a human typed, and do not hit enter on text you did not inject yourself.
