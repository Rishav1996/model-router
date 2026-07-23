# Model catalog

Verified against Anthropic's official pricing and model docs on **24 July 2026**.
Prices change. Re-check before quoting dollar figures to anyone:
- https://platform.claude.com/docs/en/about-claude/pricing
- https://platform.claude.com/docs/en/about-claude/models/choosing-a-model
- https://platform.claude.com/docs/en/build-with-claude/effort

## Contents
1. Per-token pricing
2. Context and capability
3. Effort level support
4. Discount levers
5. Cost math cheat sheet

---

## 1. Per-token pricing (USD per million tokens, standard non-batch)

| Model | Input | Cache read | Output | Relative cost |
|---|---|---|---|---|
| Claude Haiku 4.5 | $1 | $0.10 | $5 | 0.5x |
| Claude Sonnet 5 (intro, through 31 Aug 2026) | $2 | $0.20 | $10 | 1x |
| Claude Sonnet 5 (from 1 Sep 2026) | $3 | $0.30 | $15 | 1.5x |
| Claude Sonnet 4.6 | $3 | $0.30 | $15 | 1.5x |
| Claude Opus 4.8 | $5 | $0.50 | $25 | 2.5x |
| Claude Fable 5 | $10 | $1 | $50 | 5x |

Relative column is anchored to Sonnet 5 introductory pricing. **After 31 August 2026 the anchor shifts** — Sonnet 5 goes to $3/$15 and every ratio above compresses. Recompute rather than reciting.

Output is billed at 5x input across every current tier. On output-heavy work (long documents, large code generation) output dominates the bill, so a model's *verbosity* matters as much as its rate. This is a real argument for lower effort levels on generation-heavy tasks.

**Opus 4.8 fast mode** (research preview) is $10/$50 — 2x standard. Only recommend it when latency is the actual bottleneck, never as a default.

## 2. Context and capability

| Model | Context | Notes |
|---|---|---|
| Haiku 4.5 | 200k | Cheapest current-gen. Supports extended thinking. No effort parameter. |
| Sonnet 5 | 1M at standard pricing | Default across Claude plans. Near-Opus performance. |
| Sonnet 4.6 | 1M at standard pricing | Kept for workloads pinned to its behavior. Older tokenizer. |
| Opus 4.8 | 1M at standard pricing | Flagship for agentic coding and enterprise work. |
| Fable 5 | 1M, up to 128k output | Most capable widely released. Always-on adaptive thinking. |

There is **no long-context premium** on Sonnet 4.6, Sonnet 5, Opus 4.6+, or Fable 5 — a 900k-token request bills at the same per-token rate as a 9k one. Do not warn users about long-context surcharges; that's stale advice.

**Tokenizer difference:** Opus 4.7+, Sonnet 5, Fable 5, and the Mythos models use a newer tokenizer that produces roughly 30% more tokens for the same text. Sonnet 4.6 and earlier use the previous one. When comparing Sonnet 5 at $2/$10 against Sonnet 4.6 at $3/$15, the real gap is narrower than the headline.

## 3. Effort level support

`effort` controls total token spend — thinking, tool calls, and prose. It defaults to `high`, and `high` is identical to omitting the parameter.

| Model | low | medium | high | xhigh | max |
|---|---|---|---|---|---|
| Haiku 4.5 | — | — | — | — | — |
| Sonnet 4.6 | ✓ | ✓ | ✓ | — | ✓ |
| Sonnet 5 | ✓ | ✓ | ✓ | ✓ | ✓ |
| Opus 4.8 | ✓ | ✓ | ✓ | ✓ | ✓ |
| Fable 5 | ✓ | ✓ | ✓ | ✓ | ✓ |

Per-model guidance from the docs:

- **Sonnet 5** — defaults to `high`. `medium` is the cost step-down and is roughly comparable to Sonnet 4.6 at `high`. `low` for chat and latency-sensitive work. `xhigh` for the hardest coding.
- **Sonnet 4.6** — set effort explicitly to avoid unexpected latency. `medium` is the recommended working default.
- **Opus 4.8** — start at `xhigh` for coding and agentic work, `high` for other intelligence-sensitive work. Step to `medium`/`low` only after measuring that quality holds. At `xhigh`/`max`, set a large `max_tokens` (64k is a reasonable start) or the run truncates.
- **Fable 5** — effort is the primary lever. Start at `high`. Its lower effort levels often exceed `xhigh` on prior models.

`max` on Opus adds significant cost for small quality gains on most workloads, and on structured-output tasks can cause overthinking. Reserve it for genuinely frontier problems.

Effort is a behavioral signal, not a hard token cap. At `low` Claude still thinks on hard problems — just less.

## 4. Discount levers

Often larger than the model choice itself. Check these before recommending a downgrade.

| Lever | Saving | Applies when |
|---|---|---|
| Prompt caching (cache hit) | 90% off input | Same large context reused across calls |
| Batch API | 50% off input and output | Work that isn't time-sensitive |
| Both stacked | Compounds | Bulk offline processing |

Cache write costs 1.25x base input for the 5-minute TTL, 2x for the 1-hour TTL. So a 5-minute cache pays for itself after **one** read; a 1-hour cache after **two**. Below that threshold caching is a net loss — worth saying when someone proposes caching a context they'll touch once.

Changing `effort` between requests **invalidates the cache**. Vary effort across workloads, not within a cached conversation.

## 5. Cost math cheat sheet

Cost = (input_tok × in_rate + output_tok × out_rate) ÷ 1,000,000

Worked example — 50k input, 15k output on Opus 4.8:
- Input: 50,000 × $5 / 1M = $0.25
- Output: 15,000 × $25 / 1M = $0.375
- **Total: $0.625**

Same request with 40k of the input served from cache:
- Uncached input: 10,000 × $5 / 1M = $0.05
- Cache reads: 40,000 × $5 × 0.1 / 1M = $0.02
- Output: $0.375
- **Total: $0.445** — a 29% cut with no model change

The same job on Sonnet 5 (intro pricing, ignoring tokenizer drift) runs $0.25. The question is never "which is cheaper" — it's whether the 2.5x buys a first-try success.

## Surfaces where per-token cost does not apply

Pro and Max subscriptions bundle usage; Claude Code on a subscription consumes no per-token billing. For those users, translate cost into **usage-limit consumption and latency**, not dollars. Running a Haiku-grade task on Opus still burns their cap for nothing — that's the real argument.
