# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a Claude Code **plugin** (and marketplace of one) containing a single skill, `model-router`. It recommends the cheapest Claude model and effort level for a given task, using a two-axis scoring system rather than a single complexity score. It's installable three ways: as a plugin via `/plugin marketplace add Rishav1996/model-router` + `/plugin install model-router`, as a raw skill folder copied into `~/.claude/skills/`, or via the standalone `model-router.skill` archive.

There is no build, lint, or test tooling. `model-router.skill` is a zip archive (renamed) that must be kept in sync with the skill source under `skills/model-router/`.

## Files

- [.claude-plugin/plugin.json](.claude-plugin/plugin.json) — the plugin manifest (name, version, metadata). `skills/` is auto-discovered by directory convention; no explicit path needed here.
- [.claude-plugin/marketplace.json](.claude-plugin/marketplace.json) — makes this repo installable as its own marketplace; its one `plugins[]` entry points `source` at `.` (the repo root, since the plugin manifest lives here too).
- [skills/model-router/SKILL.md](skills/model-router/SKILL.md) — the skill definition itself (frontmatter `name`/`description` + body). This is what gets loaded when the skill triggers. Contains the two-axis scoring model, the capability-floor → model table, the cost-exposure → lever table, the downgrade test, output format, and worked examples.
- [skills/model-router/references/model-catalog.md](skills/model-router/references/model-catalog.md) — reference doc: per-token pricing, context limits, effort-level support per model, discount levers (caching/batch), cost math. Meant to be read on demand, not loaded every turn.
- [skills/model-router/references/scoring-rubric.md](skills/model-router/references/scoring-rubric.md) — reference doc: exemplar anchors for each of the five scoring dimensions (D/S/A/C/O), tie-break rules, common misscoring patterns, and guidance for reading the self-calibration override log.
- `model-router.skill` — a zip archive bundling `model-router/SKILL.md`, `model-router/references/scoring-rubric.md`, and `model-router/references/model-catalog.md` (for the non-plugin, drop-into-a-skills-folder install path). **Rebuild it from `skills/model-router/` whenever that source changes** rather than editing the archive directly.
- [CITATION.cff](CITATION.cff) — machine-readable citation metadata; GitHub renders a "Cite this repository" button from it automatically. Keep it in sync with the author/repo info in `.claude-plugin/plugin.json` and the attribution footer at the bottom of `skills/model-router/SKILL.md`.

## Architecture: how the skill's logic fits together

The core idea (in SKILL.md) is that "how capable a model must be" and "how much the task will cost" are separate axes, scored independently, because they're driven by different levers:

- **Capability floor = max(D, S, A)** — Depth, Stakes, Agentic-loop. A *maximum*: the hardest single dimension sets the floor, they don't sum.
- **Cost exposure = C + O + A** — Context, Output volume, Agentic-loop. *Additive*: tokens accumulate.
- `A` (agentic loop) deliberately appears in both formulas — a long tool loop is both harder to get right and more expensive to run.

Flow when the skill is invoked:
1. Score the five dimensions (0–3 each) against the **exemplars** in scoring-rubric.md, not the abstract band labels in SKILL.md — exemplars are the stable reference; prose definitions drift.
2. Capability floor picks the model (Haiku/Sonnet/Opus/Fable) and baseline effort.
3. Cost exposure picks which optimization lever to reach for first — scoping > caching/batching > stepping effort down > downgrading the model, in that order. Model downgrade is last-resort and gated by the explicit `P(failure) × cost < savings` test in SKILL.md, never done on score alone.
4. Output is a strict 3-line banner (model/effort/thinking, why, lever) — the skill is meant to run on every turn, so verbosity there gets it disabled.

model-catalog.md and scoring-rubric.md are pulled in only when needed (ambiguous scoring, user pushback, or a need for actual pricing/effort-support facts) — SKILL.md is the always-loaded part.

## Editing this skill

- Keep SKILL.md's ban on "prose definitions" honest: dimension scoring guidance belongs in scoring-rubric.md's exemplar tables, not as new adjective-based rules in SKILL.md.
- Pricing/model facts in model-catalog.md are dated ("Verified against ... on 24 July 2026") and expected to go stale — update the verification date when you touch prices, and don't let SKILL.md duplicate numbers that live in model-catalog.md.
- The self-calibration log (`model-router-log.md`) mechanism described in SKILL.md only works in Claude Code (persistent filesystem); don't extend that mechanism's claims to the Claude app or API surfaces, which SKILL.md explicitly says stay uncalibrated.
- After editing any file under `skills/model-router/`, repackage `model-router.skill` (zip of `model-router/SKILL.md` + `model-router/references/{scoring-rubric.md,model-catalog.md}`) so the distributed archive matches source.
- If you bump the skill's behavior in a user-visible way, bump `version` in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` together — they're currently kept in lockstep for this single-plugin repo.
