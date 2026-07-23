# model-router

A Claude Code / Claude API skill that recommends the cheapest Claude model and effort level that will still get the current request right on the first try — by scoring capability requirements and cost exposure as two separate axes instead of one blended "complexity" score.

Most routers score task complexity on a single scale and map it to a model. That conflates two independent questions and produces confident nonsense on the edges — a hard problem in a tiny context scores low and gets routed to Haiku, while a long trivial task in a huge context scores high and gets routed to Opus.

This skill scores two axes separately:

- **Capability floor** = `max(D, S, A)` — Depth of reasoning, Stakes, Agentic-loop complexity. A *maximum*, not a sum: the hardest single dimension sets the floor.
- **Cost exposure** = `C + O + A` — Context load, Output volume, Agentic-loop complexity. Genuinely *additive*: tokens accumulate.

The floor picks the model. The exposure picks which lever to pull to save money (scope it smaller → cache/batch → step effort down → downgrade the model, in that order) — and includes an explicit downgrade test so the skill never quietly swaps in a cheaper model on vibes alone.

## What's in this repo

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | The skill definition — scoring model, model/effort mapping, output format, worked examples. This is what gets loaded when the skill triggers. |
| [`model-catalog.md`](model-catalog.md) | Reference doc: per-token pricing, context windows, effort-level support per model, discount levers (caching/batch), cost math. |
| [`scoring-rubric.md`](scoring-rubric.md) | Reference doc: exemplar anchors for each scoring dimension, tie-break rules, common misscoring patterns. |
| [`model-router.skill`](model-router.skill) | Packaged distributable — a zip archive bundling the three files above under `model-router/`, ready to drop into a skills directory. |

## Install

**Claude Code:** unzip `model-router.skill` (or copy the three source files) into your skills directory as `model-router/`, with `SKILL.md` at the top level and the other two files under `references/`.

**Claude app:** paste the contents of `SKILL.md` into a project's custom instructions, or reference it manually — the app has no persistent skills directory, so the self-calibration log feature (see below) won't function there.

## Output format

The skill is designed to run on every turn without becoming noise — it emits a compact 3-line banner, then answers the actual question:

```
▸ Sonnet 5 · effort xhigh · thinking adaptive
  Why: floor 2 — hard but not novel; exposure 7 is where the money goes
  Lever: scope to 2 files per pass before touching effort. Set max_tokens ≥64k
```

## Self-calibration (Claude Code only)

In Claude Code, when you override a recommendation, the skill appends a line to `model-router-log.md` in your project root. After ~30 entries, a systematic bias becomes visible (usually under-scoring the agentic-loop dimension) — the fix is to adjust the exemplars in `scoring-rubric.md`, not the score bands. This requires a persistent filesystem and only works in Claude Code; on the Claude app or API, the rubric stays uncalibrated.

## Staleness warning

Pricing, context windows, and model IDs in `model-catalog.md` are dated ("verified against Anthropic's official pricing and model docs on 24 July 2026") and **will go stale** as Anthropic ships new models and pricing changes. Before trusting a dollar figure or model ID from this skill for anything consequential, re-verify against:

- https://platform.claude.com/docs/en/about-claude/pricing
- https://platform.claude.com/docs/en/about-claude/models/choosing-a-model
- https://platform.claude.com/docs/en/build-with-claude/effort

Re-verifying and updating the catalog every few weeks (or whenever a new model ships) is the main maintenance burden of this repo.

## License

MIT — see [LICENSE](LICENSE).
