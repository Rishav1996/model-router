# Scoring rubric — exemplars, tie-breaks, failure modes

Read this when a request is ambiguous, when the user challenges a recommendation, or when two dimensions are within a point of each other.

## Contents
1. Why exemplars instead of definitions
2. Exemplar anchors per dimension
3. Estimating context without a token counter
4. Tie-breaks
5. Common misscores
6. Multi-turn drift
7. Reading the override log

---

## 1. Why exemplars instead of definitions

Prose definitions produce different scores for the same prompt on different turns, because phrases like "multi-step" and "high stakes" are elastic. Exemplars are rigid: a prompt either resembles the anchor or it doesn't.

**Score by nearest neighbour.** Find the anchor the request most resembles and take its level. Do not reason from the abstract label. If a request sits between two anchors, take the higher one on D and S, the lower one on C and O.

---

## 2. Exemplar anchors

### D — depth of reasoning

Not "is this topic hard" but "how many non-obvious inferential steps sit between the prompt and a correct answer."

| Level | Anchors |
|---|---|
| **0** | "Fix this typo." · "Convert this JSON to YAML." · "What does `git rebase -i` do?" |
| **1** | "Summarize this article." · "Write a function that validates an email." · "Classify these tickets as bug/feature/question." |
| **2** | "Why is this query slow?" · "Design a schema for multi-tenant billing." · "Should we use Kafka or SQS here?" |
| **3** | "Why does my Granger causality test show significance on shuffled data?" · "Find the methodological flaw in this paper." · "Reconcile these two studies that reached opposite conclusions." |

Note the D3 anchor is nine words. Prompt length tracks context load, not depth — a detailed spec usually makes a task *shallower* by removing ambiguity.

### S — stakes

Specifically: how expensive is an error **you won't notice**.

| Level | Anchors |
|---|---|
| **0** | Scratch code. A draft nobody reads. Anything obviously broken if wrong. |
| **1** | An internal doc that gets reviewed. A PR that goes through code review. |
| **2** | A published post. An email to a client. Merged code. |
| **3** | A migration script that could silently drop rows. A citation in a submitted paper. A statistical claim in a dissertation. |

S3 is the level that most justifies spending, because the user cannot cheaply verify the output. Silent failure is the whole criterion.

### A — agentic loop

| Level | Anchors |
|---|---|
| **0** | Pure conversation. No tools. |
| **1** | One search. Read a file, answer about it. |
| **2** | Read several files, edit, run tests, iterate on failures. |
| **3** | Autonomous run over 30 minutes. Multi-agent orchestration. |

A is disproportionately expensive: every tool result re-enters context next turn, so spend compounds. Assume A2–A3 total token use is several times the visible prompt.

### C — context load

Count everything re-sent: history, attachments, project knowledge, retrieved docs. Not just the current message.

| Level | Anchors |
|---|---|
| **0** | A question with no attachments, early in a thread. |
| **1** | One paper attached. A 20-turn conversation. |
| **2** | A thesis chapter. A dozen source files. A 50-turn thread. |
| **3** | A whole repo. A dozen papers. A thread past 100 turns. |

### O — output volume

Output bills at 5x input, so O carries real weight.

| Level | Anchors |
|---|---|
| **0** | A yes/no with a reason. |
| **1** | An explanation. A code review comment. |
| **2** | A full module. A report. A long document. |
| **3** | A multi-file refactor. A generated site. Several artifacts at once. |

O3 with low D is legitimate — boilerplate is voluminous but shallow. It argues against `max` effort, which increases verbosity, and for explicit structure in the prompt instead.

---

## 3. Estimating context without a token counter

| Artifact | Approx tokens |
|---|---|
| Page of prose | 500–650 |
| 100 lines of code | 1,000–1,500 |
| Typical PDF paper | 8–15k |
| 40-page thesis chapter | 20–25k |
| Medium repo (50 files) | 150–400k |

Roughly 4 characters per token in English. State estimates as estimates — "about 60k of context" is useful, a fake-precise "61,400" is not.

---

## 4. Tie-breaks

The two axes resolve most cases on their own. When a single dimension is genuinely between levels:

**On the capability floor (D, S, A):**
1. **Verifiability decides.** Can the user tell instantly if it's wrong? Code that compiles or doesn't, a query that returns rows or doesn't — break downward, a cheap failure is cheap. Prose, analysis, and citations fail silently — break upward.
2. **S outranks D.** A moderately hard task with severe consequences needs the better model more than a very hard task with none.
3. Never break the floor downward on the basis of C or O. Those live on the other axis; letting them touch the floor is the exact bug the two-axis split fixes.

**On cost exposure (C, O, A):** break upward. Underestimating spend is the more common and more expensive error, and the response to high exposure is scoping and caching — both harmless if unnecessary.

Rule 1 does the most work. "Can they tell if it's wrong?" resolves more cases than any score arithmetic.

---

## 5. Common misscores

**Long prompt scored as deep.** Length is C, not D.

**Urgency scored as stakes.** "URGENT" means the user is stressed, not that errors are costly. Score S on consequence, not tone.

**Familiar domain scored as easy.** A question being routine for Claude doesn't make it D0. Score the inference required, not the model's comfort.

**Agentic work scored as A1.** The most common miss in practice. "Fix the failing test" sounds like one step and is usually A2 — read, hypothesize, edit, run, repeat.

**Ignoring accumulated context.** By turn 30, C has silently climbed. Re-score periodically.

**"Make it faster" read as "make it cheaper."** Fast mode costs 2x. Latency and cost pull opposite ways; ask which was meant.

---

## 6. Multi-turn drift

Within a conversation the floor stays roughly flat while exposure climbs — the task doesn't get harder, but the context does.

Two guardrails:

- **Don't re-recommend every turn.** Surface a change only when the *floor* moves. Repeating a banner because exposure ticked up is noise, and noise gets the skill disabled.
- **Prefer effort adjustment to model switching mid-thread.** Switching re-sends the conversation, and on the API invalidates prompt caching. On a long thread the switch costs more than it saves.

The natural place for a real switch is a task boundary — new problem, fresh thread, different file. Flag it there.

---

## 7. Reading the override log (Claude Code only)

The override log requires a persistent filesystem and exists only in Claude Code. On other surfaces skip this section — there is nothing to read, and the rubric stays uncalibrated.

Once `model-router-log.md` has ~30 entries, look for the pattern, not the individual misses.

| Signal | Diagnosis | Fix |
|---|---|---|
| Repeated upgrades on tool-using tasks | A anchors set too low | Move "fix the failing test" class of work firmly to A2 |
| Repeated upgrades on writing and analysis | S anchors too generous | Anything the user can't verify unaided is S≥2 |
| Repeated downgrades | Floor is inflating — probably scoring D off prompt length | Re-read the D3 anchors; they're short |
| Overrides scattered with no pattern | Rubric is fine; the task mix is just varied | Change nothing |

Adjust the **exemplars**, never the band boundaries. Boundaries are arbitrary either way; the anchors are what determine whether two sessions score the same prompt alike, and that stability is the only thing making this rubric better than intuition.
