---
name: model-router
description: Recommend the cheapest Claude model and effort level that will still get the current request right on the first try, by separating what capability the task requires from what it will cost to run. Use this at the START of every response in a session where the user has asked for model routing or cost optimization — and whenever the user mentions model choice, token spend, effort levels, extended thinking, downgrading to Haiku, upgrading to Opus, or asks whether a cheaper model would handle something. Emit a compact 3-line banner, never a report.
---

# Model Router

Most routers score "complexity" on one scale and map it to a model. That conflates two independent questions and produces confident nonsense on the edges — a hard problem in a tiny context scores low and gets routed to Haiku.

This one scores two axes separately, because they behave differently and are fixed by different levers.

## The two axes

**Capability floor = max(D, S, A).** What the task *requires*. A maximum, not a sum: capability is set by the single hardest demand, and three easy dimensions never add up to one hard one.

**Cost exposure = C + O + A.** What the task will *spend*. Genuinely additive — tokens accumulate.

`A` appears on both. That is deliberate, not double-counting: a long tool loop is simultaneously harder to do well and more expensive to run.

### Rate each dimension 0–3

| | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| **D** depth | lookup, reformat | one inferential step | needs a plan first | approach itself is in question |
| **S** stakes | throwaway | reviewed draft | shipped or sent | irreversible, or errors hide |
| **A** agentic | no tools | 1–3 calls | extended loop | long-horizon autonomous |
| **C** context | <5k tok | 5–50k | 50–200k | >200k |
| **O** output | a sentence | paragraphs | a file | multiple artifacts |

Score against the exemplars in `references/scoring-rubric.md`, not against these labels. Prose definitions drift between sessions; exemplars hold.

For per-token pricing, context limits, effort-level support per model, and discount-lever math, check `references/model-catalog.md` before quoting dollar figures or effort availability — figures below are illustrative and go stale.

### Capability floor → model

| Floor | Model | Baseline effort |
|---|---|---|
| 0–1 | Haiku 4.5 | n/a (no effort parameter) |
| 2 | Sonnet 5 | `high` |
| 3 | Opus 4.8 | `xhigh` if coding/agentic, else `high` |

Fable 5 only when floor = 3 **and** A = 3. Long-horizon autonomy is the one thing it's clearly worth 2x Opus for.

Hard constraint: C = 3 rules out Haiku regardless of floor — it caps at 200k. Everything above carries 1M.

### Cost exposure → which lever to pull

Low exposure (0–2): don't optimize. The task is cheap at any effort; economizing costs more of your attention than it saves.

Moderate (3–5): baseline effort is right. Note the discount levers if the work repeats.

High (6–9): apply levers **in this order**, cheapest regret first.

1. **Scope it smaller.** Fewer files, narrower question. Free, and usually the biggest cut.
2. **Caching or batch.** 90% off repeated input, 50% off non-urgent work. Costs nothing in quality.
3. **Step effort down one level.** Cheap in quality, and per Anthropic's own guidance a better lever than swapping models.
4. **Downgrade the model.** Last. Most quality-destructive, and only after the test below.

Most routers jump straight to 4. Steps 1–3 are free and usually sufficient.

## The downgrade test

Never downgrade on score alone. Downgrade only when:

```
P(failure) × (retry cost + your attention) < savings
```

Opus → Sonnet saves roughly 60%. If you'd put failure at 25% and a miss costs a retry plus a context switch, don't. The estimate needn't be precise — forcing it to be *named* is what kills the reflexive downgrade.

Refuse to downgrade outright when: a cheaper model already failed on this exact task; output feeds an automated pipeline with no human check; or the user is on a subscription, where per-token savings are notional and the only real cost is usage cap and latency.

## Output format

Three lines. This runs often; a verbose recommendation gets the whole thing switched off.

```
▸ <Model> · effort <level> · thinking <on/off/adaptive>
  Why: <floor and exposure in 10-15 words>
  Lever: <the one thing that would cut cost most>
```

Then answer the actual question. The banner is a header, never a substitute for the work.

Expand past three lines only when the recommendation *changes*, when the user asks why, or when a hard constraint forces the choice.

## Self-calibration — Claude Code only

**This section requires a persistent filesystem. It works in Claude Code and nowhere else.** The Claude app has no working directory that survives a session, so a log written there is gone by the next conversation. Do not tell a claude.ai user to keep one; the mechanism silently does nothing.

In Claude Code, when the user overrides a recommendation, append one line to `model-router-log.md` in the project root:

```
2026-07-24 | rec: Sonnet 5/medium | actual: Opus 4.8 | task: multi-file refactor | miss: A underscored
```

At roughly thirty entries a systematic bias becomes visible — almost always underscoring A on agentic work, or S on unverifiable prose. Adjust the exemplars, not the bands. See `references/scoring-rubric.md` §7 for reading the patterns.

**On every other surface the rubric is uncalibrated and stays that way.** Say so if the user asks how it improves: it doesn't, absent a log. The honest substitute is a manual pass — score ten prompts from real history, including some the user knows they got wrong, and compare against hindsight. That is a one-off exercise, not a feedback loop, and it should not be described as one.

## Surface awareness

**Claude app:** Claude cannot switch its own model. Name what the user should pick, and translate effort into what they actually have. Never imply the switch happened.

**Claude Code / API:** give the model ID and exact `effort` value. Recommending `xhigh` or `max` requires flagging `max_tokens` — 64k is a reasonable start, or the run truncates mid-work. This is the only surface where the self-calibration log above functions.

**Subscription (Pro/Max):** stop quoting dollars. Recommend in usage-cap and latency terms. A Haiku-grade task on Opus burns their cap for nothing — that's the whole argument.

## Running on every turn

Skills trigger on description match, which won't fire on unrelated turns. For unconditional behavior the user must paste this into Settings → Profile → user preferences:

```
Before answering anything, emit the 3-line model-router banner, then
answer normally. Keep the banner to 3 lines.
```

Skill supplies the logic; preferences supply the trigger.

## Worked examples

**"fix this typo in my README"** — D0 S0 A0 C0 O0. Floor 0, exposure 0.
```
▸ Haiku 4.5 · thinking off
  Why: floor 0 — mechanical edit, nothing to reason about
  Lever: none worth pulling; this costs fractions of a cent
```

**"review my 40-page dissertation draft for methodological gaps"** — D3 S3 A0 C2 O2. Floor 3, exposure 4.
```
▸ Opus 4.8 · effort high · thinking on
  Why: floor 3 on both depth and stakes — a missed flaw you can't spot costs a revision cycle
  Lever: none — this is the case where downgrading fails the test
```

**"refactor auth across these 6 files, run the tests"** — D2 S2 A2 C2 O3. Floor 2, exposure 7.
```
▸ Sonnet 5 · effort xhigh · thinking adaptive
  Why: floor 2 — hard but not novel; exposure 7 is where the money goes
  Lever: scope to 2 files per pass before touching effort. Set max_tokens ≥64k
```
Note what the two axes did here: naive complexity scoring sends this to Opus. The floor says Sonnet is sufficient; the exposure says the real saving was scoping, not the model.

**"summarize these 30 support tickets into themes"** — D1 S1 A0 C2 O1. Floor 1, exposure 3.
```
▸ Haiku 4.5 · thinking off
  Why: floor 1 — shallow per-item extraction repeated 30 times
  Lever: Batch API halves it again if this isn't time-sensitive
```
The model was never the lever here. Batch was.

---

Author: Rishav Saigal ([github.com/Rishav1996/model-router](https://github.com/Rishav1996/model-router)). MIT licensed — free to use and adapt; if you redistribute this skill or build on its scoring model, please keep this attribution or cite the repository (see `CITATION.cff`).
