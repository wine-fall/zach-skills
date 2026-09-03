# normal

Everyday skills that are not tied to any external tool — unlike [`herdr/`](../herdr/), they work the moment they are installed, with no environment to be inside of first.

Right now this layer holds two registers of the same activity: **making something clear**. The only difference is whose thing is being explained.

| Skill | What it does |
| --- | --- |
| [`explaining-mechanisms-progressively`](explaining-mechanisms-progressively/) | Explains mechanisms **we did not write**: what a line of syntax, a syscall, or an API primitive actually does underneath. Verifies the analogy against the real terminology word by word internally, and only then ships — analogy and term mapping together, in one message. |
| [`human-words`](human-words/) | Explains **our own system's** rules: a business rule, a workflow, told to someone who will never open the code. Leads with the rule and its consequence, walks one concrete case, goes straight to the edge cases, and says who pays and what they see. |

## Where the line falls

`human-words` draws the line itself in its section 5, and the test is a single question:

> **Would this answer still be true at another company?**
> Yes → `explaining-mechanisms-progressively` (how `with` works, what `fork` copies).
> No → `human-words` (what happens to a clip between upload and export, why this run wasn't charged).

A general concept may appear inside `human-words` as a supporting sentence — define it in one plain sentence, use it, move on. Just don't let it become the subject.

## The asymmetric triggers are deliberate

- `explaining-mechanisms-progressively` fires automatically off its `description`. Ask, and it runs.
- `human-words` answers **only to a typed `/human-words`**. The user saying they didn't follow doesn't count, a topic looking tangled doesn't count, and neither does you being about to write a long explanation anyway — those are ordinary replies. This skill is the user deliberately switching register, and only they get to call it.

The contrast is worth keeping in mind when writing your own: the skills safe to auto-trigger are the ones with an objective test and a cheap false positive. The ones that have to stay manual are the ones where "the model thinks this applies" and "the user wants this" keep failing to line up.

## Shared traps

- **Self-check the analogy before it leaves.** Don't ship the first analogy that comes to mind — swap every noun in it for the real term and read it back; if a sentence turns false, the analogy is wrong. Run that loop internally and don't narrate it.
- **Don't narrate the call graph.** "A calls B, B writes to C" is the most common failure. The reader has no call graph and doesn't want one.
- **Plain is not vague.** If a detail would change what the reader decides, keep it — and say it in plain words.
