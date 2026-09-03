---
name: human-words
description: "Explain how something in OUR OWN system works, in plain human words — the rule it follows, who it affects, what it costs — for a reader who will never open the code. Covers business rules AND the code/workflow logic behind them; excludes only general technical knowledge (what k8s is, a Python syntax question). MANUAL TRIGGER ONLY: fires solely when the user types /human-words. Never invoke it because an explanation seems wanted, because a topic looks confusing, or because the user asked 'why does it do that' — an ordinary answer stays an ordinary answer unless the command was typed. With an argument, explain that. With none, explain the most recent piece of our own system's logic discussed in this conversation."
---

# human-words — say the business rule in plain words

## 0. Trigger discipline

Fires **only** on a typed `/human-words`. Never reach for it on your own: not when a topic looks tangled, not when the user says "没懂", not when you are about to write a long explanation anyway. Those are ordinary replies. This skill is the user deliberately changing the register, and only they can call it.

## 1. Pick the subject

**With an argument** → explain exactly that.

**With none** → scan back through this conversation for the most recent piece of **our own system's logic** and explain it. If the recent context holds nothing but general technical questions, say so, name the two or three candidates you did find, and ask which. Do not manufacture a subject.

### What is in scope

**Business rules and code logic are not opposites here.** A rule the product follows and the workflow that implements it are the same subject seen from two distances, and both belong to this skill. In scope:

- how a workflow actually runs — its stages, what each one decides, where it can bail out
- why a rule is the way it is, and what it means for a user, their content, their money, or their time
- what a change does to any of the above
- how data moves through our system and who decides what along the way

Out of scope is **general technical knowledge** — things true independent of anything we built:

- "k8s 是什么", "Temporal 是什么", "什么是 unix socket"
- a Python / TypeScript syntax question, a library's API
- why one specific error is being thrown right now (that is debugging, not explaining)

### The test — would the answer still be true at another company?

> **Yes** → not this skill. (What k8s is. How Python's `with` works.)
> **No** → this skill. (How a clip goes from upload to export. Why this run wasn't charged. What the planner does when it finds no usable footage.)

A general concept that the real subject *depends on* stays in scope as a supporting sentence: explaining how our workflow runs on Temporal may need one plain-words sentence saying what Temporal is. Define it, use it, move on — do not let it become the topic.

## 2. Write it

Answer in the language the user asked in.

- **Lead with the rule and its consequence.** First sentence states what happens and to whom. Never open with the pipeline, the history, or "简单来说".
- **Use real-world nouns.** 用户、这条片子、一次导出、他的钱包、这个项目。Not `task doc`, `activity`, `configmap`, `HPA`, `revision`. Where a technical name is genuinely the thing, define it once in ordinary words and then use the ordinary word for the rest of the explanation.
- **Walk one concrete instance.** A single real case with real values teaches more than an abstraction. Prefer a case that actually came up in this conversation.
- **Go straight to the edges.** The business logic almost always lives in what happens when it goes wrong, when the input is empty, when the user does it twice, when they cancel halfway. A rule explained only on the happy path has not been explained.
- **Say who pays and what they see.** Whenever money, credits, quota, or user-visible output is involved, state it explicitly — that is usually the part the reader actually needed.
- **Keep the load-bearing detail.** 通俗 is not 含糊. If a detail changes what the reader would decide, keep it and say it in plain words; drop the detail only when dropping it changes nothing.

## 3. Do not

- **Do not narrate the call graph.** "先调 A,A 再调 B,B 写进 C" is the single most common failure. The reader does not have a call graph and does not want one.
- **Do not use a function, file, or field name as the backbone** of a sentence. An identifier may appear only as a pointer for someone who later wants to go look — never as the explanation itself.
- **Do not open with a preamble.** No "这个其实很简单", no "让我用一个比喻".
- **Do not stretch an analogy past where it holds.** An analogy is allowed when it carries real weight, but you must say where it breaks. An analogy that is cuter than it is true does more damage than no analogy.
- **Do not smooth over uncertainty.** If part of the rule is something you believe rather than something you checked, say which part, in plain words. A fluent explanation of a rule that turns out not to exist is the worst possible output here.

## 4. Length and shape

Prose. No section headings for anything short — headings are for documents, and this is an explanation. A table only when the rule genuinely differs across two or three cases and the contrast is the point. Stop when the reader could restate the rule and predict what the product does in a case you did not mention; every sentence past that is decoration.

## 5. Not to be confused with

- `eli5` — dead-simple picture explainer, any topic, five-year-old register. This one talks to an adult colleague.
- `explaining-mechanisms-progressively` — how a language/OS/API primitive works under the hood: a mechanism that exists whether or not we built anything. That skill owns the general concept; this one owns our system's own logic, and will happily explain code and workflows — just never in code's own vocabulary.
- The repo's **Explanation default** — already the standing register for a reader who shaped the architecture and keeps the mechanism's technical depth. `/human-words` goes one step further out: the reader may be a PM, a support engineer, or someone in a meeting, and the subject is the rule rather than the mechanism.
