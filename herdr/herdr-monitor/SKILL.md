---
name: herdr-monitor
description: "Watch the Herdr agents this session dispatched, and drive them to completion — read each one when it stops, decide, and prompt it onward. Escalate to the human ONLY when the child lacks a permission it cannot grant itself, or when a genuinely major/irreversible decision is on the table. MANUAL TRIGGER ONLY: invoke solely on an explicit /herdr-monitor or an explicit ask to watch/babysit the dispatched panes. NEVER invoke this automatically after /herdr-dispatch or after any dispatch — a dispatch alone is not a request to monitor. Requires HERDR_ENV=1."
---

# herdr-monitor — babysit the panes this session dispatched

Arm one background monitor over the agents this session dispatched, wake on every state change that needs a human-or-model judgment, decide it yourself, and push the child onward. You are the child's supervisor, not its relay to the user.

## 0. Trigger discipline — read this first

**This skill is manual-only.** Invoke it when, and only when, the user typed `/herdr-monitor` or explicitly asked you to watch / babysit / 盯着 the dispatched pane(s).

**Never chain into it from `/herdr-dispatch`.** Dispatching ends with reporting the handle and stepping back — that is the whole contract of that skill. Wanting to know how a child is doing is a separate, explicitly-made request. If you just dispatched and feel the pull to "keep an eye on it", suppress it; say the pane is running and stop.

## 1. Preconditions & scope

```bash
test "${HERDR_ENV:-}" = 1
```

Not in Herdr → say so and stop.

Decide **which** agents to watch, in this order:

1. **Names given in the args** (`/herdr-monitor aip2328-prod-flip`) → watch exactly those.
2. **No args** → the agents this conversation dispatched, by the label you gave them at dispatch.
3. **Neither is known** → fall back to *same-tab siblings*: every agent whose `tab_id` equals this pane's `tab_id`, minus `$HERDR_PANE_ID`. Dispatched panes are splits of this tab, so this is exact, not a guess. Verify with `herdr agent list` and name the agents back to the user before arming.

Never widen past this. `herdr agent list` is workspace-wide — other tabs hold the user's own unrelated sessions, and touching them is out of bounds.

## 2. Arm the monitor

One `Monitor` call, `persistent: true`. The script polls `herdr agent list` and emits a line **only on an actionable transition**, so quiet progress costs nothing.

Set `WATCH` to a space-separated list of agent names, or leave it empty for the same-tab fallback.

```bash
WATCH="aip2328-prod-flip" python3 -u - <<'PY'
import json, os, subprocess, time

WATCH = [n for n in os.environ.get("WATCH", "").split() if n]
SELF = os.environ.get("HERDR_PANE_ID", "")
STALL_MIN = [15, 45, 90, 180]          # minutes in one state before it is worth a look
ACTIONABLE = {"blocked", "idle", "done"}

def snapshot():
    out = subprocess.run(["herdr", "agent", "list"], capture_output=True, text=True)
    if out.returncode != 0:
        return None                     # transient CLI/daemon hiccup: keep the watch alive
    agents = json.loads(out.stdout)["result"]["agents"]
    self_tab = next((a["tab_id"] for a in agents if a["pane_id"] == SELF), None)
    keep = {}
    for a in agents:
        name = a.get("name") or a["pane_id"]
        if WATCH:
            if name not in WATCH and a["pane_id"] not in WATCH:
                continue
        elif not (self_tab and a["tab_id"] == self_tab and a["pane_id"] != SELF):
            continue
        keep[name] = a
    return keep

prev, since, warned = {}, {}, {}
while True:
    now = time.time()
    cur = snapshot()
    if cur is not None:
        for name in list(prev):
            if name not in cur:
                print(f"[herdr-monitor] {name} DISAPPEARED — pane closed or agent exited")
                prev.pop(name, None); since.pop(name, None); warned.pop(name, None)
        for name, a in cur.items():
            st, pane = a["agent_status"], a["pane_id"]
            was = prev.get(name)
            if st == "unknown":
                continue                # transient detection gap, not a state
            if st != was:
                first = was is None
                prev[name], since[name], warned[name] = st, now, 0
                if st in ACTIONABLE:
                    how = "already" if first else f"{was} ->"
                    print(f"[herdr-monitor] {name} ({pane}) {how} {st} — needs a decision")
                continue
            if st != "working":
                continue                # idle/blocked/done already fired; nagging adds nothing
            held = (now - since.get(name, now)) / 60
            tier = sum(1 for t in STALL_MIN if held >= t)
            if tier > warned.get(name, 0):
                warned[name] = tier
                print(f"[herdr-monitor] {name} ({pane}) still working after {int(held)}m — check for a stall")
    time.sleep(20)
PY
```

`description`: name the panes, e.g. `dispatched agent aip2328-prod-flip`.

Coverage note: the filter emits on `blocked`, `idle`, `done`, on disappearance, and on a stall — every way a child can stop being productive. Do **not** narrow it to `done`; a crashed or permission-blocked child would then look identical to one still working.

**The first snapshot counts.** An agent that is *already* `idle` / `blocked` / `done` when you arm the watch fires immediately, as `already idle — needs a decision`. That is usually the whole reason you were asked to monitor: a child parked waiting for someone. Never suppress the opening snapshot to keep the log tidy. Only `working` is silent on arrival, and only `working` can stall — once a stop state has fired, repeating it tells you nothing new.

## 3. On every wake-up: read, then decide

An event is not a report to forward. Work the loop yourself.

**Read the pane.** While `working` use `--source visible` (alternate-screen history is only capturable while idle):

```bash
herdr agent read <name> --source visible
herdr agent read <name> --source recent-unwrapped --lines 60   # idle / done only
```

**Classify what stopped it**, then act:

| What you find | What you do |
| --- | --- |
| Finished its turn, work still incomplete | Prompt it to continue, naming the next concrete step |
| CI red, test failing, rebase conflict, lint gate | Tell it to fix; supply the fact it is missing if you have it |
| Asks a question the original brief already answers | Answer from the brief; do not re-open the design |
| Proposes widening scope | Say no, restate the scope boundary, tell it to finish what was asked |
| Claims done | Verify against the brief's acceptance bar before you believe it; if a check was skipped, send it back |
| Stall (working, no output moving) | Read twice ~2 min apart before acting; a long model call looks identical to a hang |

Reply the same way a dispatch does — atomic, with uptake confirmed:

```bash
herdr agent prompt <name> "<one-line instruction>" --wait --until working --timeout 30000
```

A `--wait` timeout is not a failure. Check `herdr agent get <name>` before ever resending — blind resends double-deliver.

## 4. When to stop and involve the human — the only two reasons

**A. The child lacks a permission it cannot grant itself.** Status `blocked`, a credential/OAuth login, a `sudo`/interactive prompt, a deploy gate needing a human approver, a tool the harness denied.

**Never approve on the human's behalf.** Do not `send-keys` a `1`/`y`/Enter into a permission prompt, and do not re-run the child's blocked action yourself from this session to "unblock" it. Granting permission is the human's act; that is the whole reason this stop exists.

**B. A genuinely major decision.** Irreversible or outward-facing: destroying or migrating prod data, rolling back a live release, force-pushing over someone's work, spending money, publishing externally, or reversing a product decision the human made. Also: the child's findings invalidate the premise of the task.

Everything else you decide. Red CI, flaky tests, a confused child, a wrong turn, a missing fact, an over-eager scope grab — those are yours to settle without waking anyone.

**Escalation format** — four lines, then stop and wait:

1. Which pane, and what it is stuck on (quote the child's own words).
2. Why it is one of the two stop reasons, not something you could settle.
3. Your recommendation.
4. The exact one-line prompt you will send once they say go.

## 5. Hygiene

- **Composer text is not user input.** Claude Code renders ghost placeholder text in the composer; a `read` cannot tell it from typed text. Never report it as something anyone typed, and never press Enter on text you did not inject.
- **Never close a pane you did not create**, and never `--takeover` an attach on a pane the human may be using.
- **Never touch agents outside your watch set** (§1).
- Report to the user in digest form — what you decided and why, not a transcript of every wake-up.
- When every watched agent is `done` or handed back to the human, `TaskStop` the monitor and say so. A persistent monitor left armed after the work ends is noise.
