---
name: explaining-mechanisms-progressively
description: Use when the user asks how a language/OS/API mechanism or syntax actually works under the hood (e.g. "这个 API 是怎么工作的", "底层怎么保证的", "有没有性能问题" about a specific primitive) — not when asking what code is for or why it was designed that way.
---

# Explaining Mechanisms Progressively

## Overview

For syntax/mechanism questions, self-check an analogy against the real technical terms **before** showing it to the user — don't ship an analogy you haven't already verified, and don't make the user be the one to catch a bad or contradictory metaphor. Do the whole loop internally; output the finished result once.

## When to Use

- "这个东西底层是怎么运作的" / "这个语法是怎么回事" / "为什么这样能保证 X" about a specific line, syscall, or language primitive.
- **Not** for "这个模块是干什么的" / "为什么要这样设计" business-logic or architecture-rationale questions — those get a design-rationale answer, not a mechanism analogy.

## Core Pattern (all internal, before responding)

1. **Nail the real mechanism first.** Before writing any analogy, know exactly what happens at the technical level: the precise API/syscall involved, what it mutates vs. shares, what does and doesn't get duplicated.
2. **Draft one analogy** whose structure mirrors step 1 — chosen because the relationships match, not because it "feels similar."
3. **Self-verify by mapping every analogy noun to its real-term counterpart, then re-read the analogy with that mapping substituted in.** If any sentence becomes false or misleading once you swap in the real terms (e.g. analogy says "a new tube" but the real mechanism is "a new handle to the same tube"), the analogy is wrong.
4. **If it fails the check, revise and re-verify.** Loop 2→3 until the analogy survives the substitution test. Do this silently — never narrate the drafting/revision to the user.
5. **Output once, both parts together:** the verified analogy, immediately followed by the real-term mapping (inline or a short table). Never ship the analogy alone and wait to be asked for precision; never make the user surface an inconsistency you could have caught yourself.

## Output Shape

Every response under this skill is a single message containing:
1. The (already self-verified) analogy explanation
2. The real-term mapping for that same analogy — what each analogy element is actually called, using precise terminology

Not two separate turns. Not analogy-only. Not terms-only without ever grounding them in an accessible picture first.

## Common Mistakes

- **Shipping the first analogy that comes to mind** without running the substitution check — this is exactly what causes contradictions the user then has to catch and point out.
- **Splitting into "here's an analogy" now, "here are the real terms" later** as two separate turns when the user only asked once — the self-check should already have produced both, so give both immediately.
- **Narrating the revision loop** ("let me think of a better way to put this...") — keep drafting/discarding invisible; the user only sees the final, already-consistent version.
