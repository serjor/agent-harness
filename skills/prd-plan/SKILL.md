---
name: prd-plan
description: Iterative PRD workflow that replaces plan mode. Investigates the real codebase, maps the gap between what exists and what is being asked for, hunts blind spots, interrogates the user until every consequential decision is closed, then emits one self-contained HTML plan that an implementation agent executes verbatim. Use this whenever the user wants to plan, scope, spec, or think through a code change before anything is written — including "plan this", "spec this out", "PRD", "how would we build X", "what would it take to", "what breaks if we", "before you code", or any feature, refactor, migration, or architectural request that is not a one-line fix. Prefer this over replying with an inline or markdown plan.
---

# PRD Plan

Plan mode fails in a specific way: it guesses a plan from the request instead of deriving it from the code, then dumps markdown that the implementation agent has to re-verify from scratch. This skill inverts that. Spend the tokens on investigation and on the user, spend as few as possible on prose.

The user is a product manager who also ships code. Talk to them that way — no jargon translation, no explaining what a foreign key is, no hedging. When the code contradicts what they asked for, say so with the file open in front of you.

## Hard rules

1. **Read-only on the project.** Read, grep, trace, run read-only commands. Never edit project files, never run a migration, never install a dependency. The only file ever written is the HTML plan, at a path the user approved.
2. **Evidence or silence.** Every statement about current behaviour carries a `path:line` or symbol you actually opened. Never infer file layout from framework convention — repos lie.
3. **No plan before convergence.** The HTML is emitted only once no open question would change its content.
4. **Surface cost early.** The moment recon shows the request is materially bigger than it looks, stop and say it — before asking anything else, with the evidence and with cheaper alternatives if any exist. The user's whole reason for using this is to decide with the real price on the table.
5. **Go to the point.** No preambles, no recaps, no "great question", no restating the request back. Facts, paths, tradeoffs.

## Phase 0 — Size the request

The size decides how much recon and how many rounds of questions. Treat the user's own estimate as a hypothesis; the code gets the final vote, so **re-run this classification after recon** and tell the user if it moved.

| Tier | Signals | Recon | Questions |
|---|---|---|---|
| **S — surgical** | Contained in a handful of files, no schema change, no contract change, no new dependency | Targeted grep + read the touched functions and their tests | 0–2, one round, or none if the code answers everything |
| **M — feature** | New surface inside existing architecture, may add tables/endpoints/config, no breaking change | Read the whole vertical slice end to end, plus one precedent feature to mirror | 3–6, one or two rounds |
| **L — structural** | Schema migration, cross-cutting refactor, contract break, new dependency at the core, data backfill, auth or tenancy change | Full trace, data lifecycle, every call site, migration and rollback path, blast radius | As many rounds as it takes; do not compress this |

Tier drift upward is the single most valuable output of this skill. "You asked for a field on the user profile; that field is denormalised into three read models and a cache key, so this is L not S" is worth more than any plan.

## Phase 1 — Recon

Do not ask the user anything the repo can answer. Before the first question, know:

- **Entry points**: where the request enters and where the effect lands.
- **The data**: models, schema files, migrations directory, how state is persisted and read back.
- **Precedent**: find the closest thing already implemented in this repo and mirror its shape. A plan that fights local convention gets rejected in review no matter how good it is.
- **Tests**: what covers the affected paths today, and what does not.
- **Ownership of the seams**: who else calls the thing being changed — grep call sites, don't assume.

Language and stack are irrelevant to the method. Find the build file, find the test runner, find the entrypoint, follow the data.

## Phase 2 — Gap

State origin and destination as concrete deltas, not aspirations. For each thing that must become true: what is true now (with evidence), what must be true after, and what has to move for that. This is the section that lets an implementation agent skip re-exploration, so it earns its tokens.

## Phase 3 — Blind spot sweep

Walk this list against the change. Report only the entries that actually bite, each with evidence. Silence on the rest — a plan padded with inapplicable risk boilerplate trains the reader to skim.

- **Data model**: migration up *and* down, backfill of existing rows, nullable-vs-default on a live table, index cost, lock duration on large tables.
- **Contracts**: API/event/CLI shape changes, consumers you don't control, versioning window, serialized data already on disk or in a queue.
- **State in flight**: in-flight requests, queued jobs enqueued under the old schema, cron jobs, retries, idempotency.
- **Concurrency**: new races, transaction boundaries, ordering assumptions, locks.
- **Cache & derived data**: invalidation, cache keys embedding the old shape, search indexes, read models, materialized views.
- **Auth & tenancy**: permission checks on new surfaces, data leakage across tenants, row-level scoping.
- **Compatibility**: rolling deploy where old and new code run simultaneously, mobile clients that never update, third-party integrations.
- **Operational**: observability for the new path, feature flag, rollback that is actually reversible, cost at production volume.
- **Failure modes**: what happens on partial failure, on timeout, on bad input.
- **Blast radius**: everything found in the call-site grep that the user has probably forgotten exists.

## Phase 4 — Interrogation

Questions cost the user time, so each one must earn it. A question is worth asking only if different answers produce different plans.

- Ask in **batches sized to the tier** (see Phase 0). Never drip-feed one question at a time on an L change and never open an S change with an interview.
- Each question carries: what it decides, the recommended default and why, and what changes if they pick otherwise. The user should be able to reply "defaults" and get a good plan.
- Rank by consequence. Schema and contract decisions first; naming last, or not at all.
- Never ask for information sitting in the repo. Asking "which ORM do you use" burns credibility you need later.
- When an answer invalidates recon, go back to Phase 1 for the affected area rather than patching the plan around it.

Record every closed decision along with the rejected alternative and the reason. That record is the most reused part of the final document — it stops the implementation agent from relitigating settled choices, and it stops the user from re-asking in three weeks.

## Phase 5 — Converge

Loop Phases 1–4 until no open question would change the plan. Then, before writing anything, put the shape in front of the user in a few lines: tier, phases, the one or two things that could go wrong, anything left genuinely unknown. Get the go-ahead.

## Phase 6 — Emit the HTML

1. Propose a path — default `docs/plans/NNNN-<slug>.html` — and ask for confirmation or an alternative. Never write without an approved path, never overwrite an existing file silently.
2. Build the file from `references/plan-template.html`.
3. Write it, then report the absolute path in one line. Do not summarise the plan back into chat; the file is the deliverable.

## The HTML contract

The implementation agent reads this file **whole, raw, tags and all**. Markup it doesn't need is context it can't spend on the code, so the file is optimised for that reader while still being pleasant for a human to review.

**Token discipline**

- One `<style>` block, copied from the template, styling by element selector so the body carries almost no `class` attributes. No inline styles.
- Zero JavaScript, zero external resources — no CDN, no web fonts, no image files, no Mermaid. One file, renders offline, prints correctly. Diagrams follow the rules below.
- Semantic tags only — `h2`, `p`, `ul`, `table`, `code`, `pre`. No wrapper `div` scaffolding.
- Tables for structured facts, prose for reasoning, code blocks only where the shape is not obvious from a sentence. Never paste a full implementation; a signature, a type, or a three-line diff sketch is the ceiling.
- Budget by tier: **S ≈ 400 words, M ≈ 1200, L ≈ 2500.** Over budget means you are explaining instead of specifying. Cut restatement first, then background, then adjectives.

**Diagrams** — optional, and most plans don't need one. Two rules make them safe to allow:

*Never load-bearing.* Anything a diagram shows must also exist in the prose or tables. The picture is for the human reviewing the plan; the implementation agent reads a picture worse than it reads a sentence, so a fact that only lives in a diagram is a fact that gets dropped.

*ASCII inside `<pre>` by default.* Box-drawing characters are cheap, render offline, print, survive copy-paste into a terminal, and an agent parses them as plain text. Reach for one only when the relationship is topological or temporal and prose would cost more than the drawing:

- four or more components with non-linear connections between them
- a sequence with interleaving, async hops, or retries
- a rolling-deploy or migration timeline where several states coexist for a window
- a before/after where the *shape* changed, not just the values

Skip it when the content is a linear list of steps (that's an `<ol>`), a three-node hierarchy (that's a sentence), or anything a table already says. A diagram that restates the table below it is pure cost.

Hand-written inline `<svg>` is permitted only for the rare case where crossing edges make ASCII unreadable, and only kept under roughly 25 elements — beyond that the coordinate noise costs the reader more than the layout gains. Place diagrams inside the section they explain, never in a section of their own.

**Sections** — in this order, omitting any that genuinely does not apply:

| Section | Contents |
|---|---|
| Goal | 1–3 sentences: what is true when this ships. Not the request rephrased. |
| Scope | In, and an explicit **Out** list. The Out list prevents the most common implementation agent failure: helpful scope creep. |
| Current state | Table — concern, `path:line`, what it does today. |
| Gap | What must become true, and what moves for it. |
| Decisions | Table — question, decision, why + rejected alternative. |
| Plan | Phases. Each: one-line goal, table of files and the change per file, optional signature or diff sketch, and a concrete **done when**. |
| Data & contracts | Only if applicable: migration up/down, backfill, API shape before/after, compatibility window. |
| Risks | Table — risk, impact, mitigation, rollback. Only real ones. |
| Verification | Tests to add or update, exact commands to run, manual checks, acceptance criteria. |
| Unknowns | Only if any survive: what is unresolved, how the implementer resolves it, and the fallback if they can't. |
| Do not | Guardrails for the implementation agent: files not to touch, patterns not to introduce, refactors not to bundle in. |

Read `references/plan-template.html` when you reach Phase 6. It holds the exact style block, the section skeleton, and a worked fragment showing the intended density. Its HTML comments are guidance — strip every one of them from the output.

## Greenfield

With no existing code, Current state becomes **Constraints** — stack, hosting, team, deadlines, things already decided elsewhere — and Precedent recon becomes a short look at whatever adjacent repos or prior art the user points to. Everything else is unchanged. Resist inventing an architecture before asking what is already fixed.
