---
name: prd-plan
description: Iterative PRD workflow that replaces plan mode. Investigates the real codebase, closes consequential decisions with the user, and emits one visual, self-contained HTML decision plan for approval and zero-context implementation sessions. Use this whenever the user wants to plan, scope, spec, or think through a code change before anything is written — including "plan this", "spec this out", "PRD", "how would we build X", "what would it take to", "what breaks if we", "before you code", or any feature, refactor, migration, or architectural request that is not a one-line fix. Prefer this over replying with an inline or markdown plan.
---

# PRD Plan

Plan mode fails when it guesses from the request and leaves the implementation agent to verify everything again. This skill derives the plan from the code and closes decisions with the user.

The HTML has two equal readers. The user uses it as a decision interface: compare credible paths, understand the price, and approve, reduce, investigate, or stop. Implementation agents use it as an execution contract across sessions with no prior context.

The document is not a transcript or a storage container. Its first screen must answer: **what decision is requested, what is recommended, why, what it costs, how reversible it is, and what happens next**. The rest must let an implementation agent act without redoing discovery.

Explain the consequence before the mechanism. Use programming concepts that transfer across languages. Keep exact paths, symbols, contracts, and commands for the implementation agent. Define a repository-specific term when the decision depends on it. Do not teach basic syntax.

## Hard rules

1. **Read-only on the project.** Read, grep, trace, run read-only commands. Never edit project files, never run a migration, never install a dependency. The only file ever written is the HTML plan, at a path the user approved.
2. **Evidence or silence.** Every statement about current behaviour carries a `path:line` or symbol you actually opened. Never infer file layout from framework convention — repos lie.
3. **No plan before convergence.** Emit the HTML only after all questions that change its content are closed.
4. **Surface cost early.** The moment recon shows the request is materially bigger than it looks, stop and say it — before asking anything else, with the evidence and with cheaper alternatives if any exist. The user's whole reason for using this is to decide with the real price on the table.
5. **Go to the point.** No preambles, praise, hedging, or request restatement. Give facts, paths, consequences, and tradeoffs.
6. **Visuals carry meaning.** Use visual hierarchy, comparison, and diagrams to reduce decision effort, not to decorate the document. Never invent precision, hide a caveat, or make a visual the only source of an implementation fact.

## Phase 0 — Size the request

The size decides how much recon and how many rounds of questions. Treat the user's own estimate as a hypothesis; the code gets the final vote, so **re-run this classification after recon** and tell the user if it moved.

| Tier | Signals | Recon | Questions |
|---|---|---|---|
| **S — surgical** | Contained in a handful of files, no schema change, no contract change, no new dependency | Targeted grep + read the touched functions and their tests | 0–2, one round, or none if the code answers everything |
| **M — feature** | New surface inside existing architecture, can add tables/endpoints/config, no breaking change | Read the whole vertical slice end to end, plus one precedent feature to mirror | 3–6, one or two rounds |
| **L — structural** | Schema migration, cross-cutting refactor, contract break, new dependency at the core, data backfill, auth or tenancy change | Full trace, data lifecycle, every call site, migration and rollback path, blast radius | As many rounds as it takes; do not compress this |

Tier drift upward is the single most valuable output of this skill. "You asked for a field on the user profile; that field is denormalised into three read models and a cache key, so this is L not S" is worth more than any plan.

## Phase 1 — Recon

Do not ask the user anything the repo can answer. Before the first question, know:

- **Entry points**: where the request enters and where the effect lands.
- **The data**: models, schema files, migrations directory, how state is persisted and read back.
- **Precedent**: find the closest thing already implemented in this repo and mirror its shape. A plan that fights local convention gets rejected in review no matter how good it is.
- **Tests**: what covers the affected paths today, and what does not.
- **Ownership of the seams**: who else calls the thing being changed — grep call sites, do not assume.

Language and stack are irrelevant to the method. Find the build file, find the test runner, find the entrypoint, follow the data.

## Phase 2 — Gap

State origin and destination as concrete deltas, not aspirations. For each result, describe what is true now, what must be true after, and what must move. Explain why the delta matters before you give its technical mechanism. Support the current state with evidence.

## Phase 3 — Blind spot sweep

Walk this list against the change. Report only the entries that actually bite, each with evidence. Silence on the rest — a plan padded with inapplicable risk boilerplate trains the reader to skim.

- **Data model**: migration up *and* down, backfill of existing rows, nullable-vs-default on a live table, index cost, lock duration on large tables.
- **Contracts**: API/event/CLI shape changes, consumers you do not control, versioning window, serialized data already on disk or in a queue.
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
- Each question carries: what it decides, the recommended default and why, and the result of another answer. The user can reply "defaults" and get a good plan.
- Rank by consequence. Schema and contract decisions first; naming last, or not at all.
- Never ask for information sitting in the repo. Asking "which ORM do you use" burns credibility you need later.
- When an answer invalidates recon, go back to Phase 1 for the affected area rather than patching the plan around it.

Record every closed decision along with the rejected alternative and the reason. That record is the most reused part of the final document — it stops the implementation agent from relitigating settled choices, and it stops the user from re-asking in three weeks.

## Phase 5 — Converge

Loop Phases 1–4 until all questions that change the plan are closed. Then show the user the tier, ordered phases, main risks, and genuine unknowns. Recommend whether to proceed, reduce scope, do more discovery, or stop. Give the reason. Get the go-ahead to generate the approval document; the HTML contains the final decision request.

### Build release-safe phases

Phases are ordered and can depend on earlier phases. Every phase must leave the repository usable and suitable for production. A phase does not need user-visible value. A refactor, linter, internal migration, or preparatory change is valid when its production state is explicit.

Each phase must contain:

- Starting state and dependencies.
- Outcome and plain-language impact. Write `None` when there is no user-visible effect.
- Exact files, symbols, and changes.
- Acceptance criteria and exact verification commands.
- Deployment, observation, and rollback.
- A production gate that proves the phase can end safely.

Do not use phases as arbitrary work buckets. If a phase cannot ship safely, merge it with the next phase or redesign the boundary.

### Make the plan resumable

The generated HTML is a living execution record after the user approves it. Put this protocol inside every plan so an implementation agent can obey it without loading this skill.

Use these phase states: `planned`, `in_progress`, `done`, and `blocked`. Only one phase can be `in_progress`. The first agent starts the first planned phase. Later agents start with the first phase that is not done.

Before an agent changes project files, the plan must tell it to:

1. Read the whole HTML.
2. Examine the evidence for all done phases.
3. Compare the recorded repository state with the current worktree.
4. Mark the selected phase `in_progress` and record the session start.

Keep the approved specification immutable. Give each phase a separate execution record with:

- Start and completion time.
- Agent or session identifier when available.
- Actual files and symbols changed.
- Verification commands and summarized results.
- Commit or change reference when available.
- Deviations and new decisions.
- Remaining risks and exact next action.

Do not paste full logs or diffs. Link or cite their durable location. Keep the latest handoff visible in the phase and the status dashboard.

An agent can correct a factual reference and must record the correction. It must stop for user direction when a discovery changes scope, a contract, risk, phase boundaries, or the approved outcome. Never rewrite the approved plan to make an unapproved deviation look intentional.

Mark a phase `done` only when its production gate has evidence. If the gate cannot pass, mark the phase `blocked`, record the cause, attempted alternatives, and the decision needed. Do not start a later phase while an earlier phase is active or blocked.

## Phase 6 — Emit the HTML

1. Propose a path — default `docs/plans/NNNN-<slug>.html` — and ask for confirmation or an alternative. Never write without an approved path, never overwrite an existing file silently.
2. Build the file from `references/plan-template.html`. Match the user's language and set the HTML `lang` attribute. Keep code, identifiers, commands, and quoted errors exact.
3. Write it, then report the absolute path in one line. Do not duplicate the plan in chat. The file is the deliverable.

## The HTML contract

The user reads the rendered file. Implementation agents read the whole raw file, including tags. One file must serve both without duplicating facts.

### Decision layer

Put these items first:

- The exact decision requested: approve, choose an option, reduce scope, fund discovery, or stop.
- Outcome and recommendation with its reason and confidence level.
- The main cost, tradeoff, reversibility, and what could change the recommendation.
- Execution status and next action.
- Impact by relevant area: behavior, code, data and contracts, operations, and developer workflow.
- Scope, main risks, and an ordered map of phases.
- Credible alternatives when they exist. Compare outcome, effort, risk, reversibility, and the condition under which each becomes the better choice. Do not manufacture alternatives to fill a layout.

The execution dashboard must show document status, completed phase count, active or blocked phase, open decisions, last update, and exact next action.

Write impact as consequence, reason, then technical mechanism. State `No change` for an area when that absence helps the user assess the request.

Use relative estimates such as `small`, `medium`, and `large` when the repository supports only relative confidence. Use dates, durations, file counts, or percentages only when evidence supports them; a polished chart does not justify false precision.

### Execution layer

Include only applicable sections:

- Current state with `path:line` and symbol evidence.
- Concrete change from current state to target state.
- Closed decisions with rejected alternatives and reasons.
- Release-safe phase specifications.
- Data and contract changes, compatibility, and migration when applicable.
- Risks with signals, mitigations, and real rollback paths.
- Exact verification commands and acceptance criteria.
- Unknowns with a resolution route and fallback.
- Guardrails that prevent scope creep.
- A separate execution record for each phase.

Keep each fact in one canonical location. The decision layer summarizes consequences. The execution layer owns mechanisms and evidence.

### Information density

Do not use a global word limit. Keep the decision layer brief enough to scan before the implementation detail. Add detail only when it changes a decision, explains impact, prevents a failure, or prevents repository re-exploration. Remove repeated context, full logs, long diffs, tutorials, and decorative prose.

Use tables for structured facts and prose for reasoning. Use code blocks only when a signature, type, command, or short diff sketch communicates the shape better. Never include a full implementation.

Progressively disclose by purpose, not by hiding important information. The recommendation, material caveats, active blockers, production gates, and next action stay visible. Put supporting evidence, rejected alternatives, and execution history in `<details>` when useful.

### Visual decision support

Choose a visual only when it makes a relationship faster to understand than prose or a small table:

- **Option comparison** for two or more credible paths with different cost, risk, or reversibility.
- **System or data-flow map** for three or more components, ownership seams, or a non-obvious blast radius.
- **Timeline or state transition** for migrations, rolling compatibility, queues, backfills, and rollback windows.
- **Dependency roadmap** when phase order or safe release boundaries are not obvious.
- **Before/after model** when the decision depends on a structural change that a diff table does not make clear.

For an M or L plan, expect at least one decision-useful visual when the change has multiple options, three or more interacting parts, or a stateful rollout. For an S plan, a strong hierarchy and compact comparison are usually enough. Omit a visual when it only repeats adjacent text.

Use semantic HTML and CSS for cards, matrices, timelines, and progress. Use inline SVG for bespoke topology, branching, and relationship diagrams. Every diagram must:

- Live in a `<figure>` with a decision-oriented `<figcaption>`.
- Have a meaningful SVG `<title>` and `<desc>`, visible labels, a declared direction, and a readable DOM order.
- Use text, shape, or line pattern in addition to color. Include a legend only when its encoding is not obvious.
- Link or point to the canonical evidence or phase details instead of duplicating them.
- Remain legible on a narrow screen, in dark mode, in print, and in the raw HTML an agent reads.

Motion is optional. Use CSS-only animation or transition only to communicate flow, active state, or a meaningful before/after relationship. Keep the static state complete, make motion subtle and finite, and disable it for `prefers-reduced-motion` and print. Never use motion to attract attention to decoration.

Do not use stock art, decorative icons, generic architecture clouds, gauges without a defensible scale, or charts whose data is not in the plan. A beautiful unsupported visualization is misinformation.

### File constraints

- Use one `<style>` block from the template. Do not use inline styles.
- Use semantic HTML and a small stable class vocabulary from the template.
- Use no JavaScript, event handlers, external resources, web fonts, CDN assets, or Mermaid.
- Keep the deliverable to one HTML file. Any image must be accessible inline SVG; never create a companion image, embed raster/base64 data, or depend on a renderer.
- Make the file work offline, on narrow screens, in dark mode, and when printed.
- Use text labels in addition to color. Do not make information depend only on presentation.
- Use `<details>` only for secondary evidence, rejected alternatives, or bounded history. Keep decisions, active status, production gates, and next actions visible.

Read `references/plan-template.html` when you reach Phase 6. It contains the exact style block, structure, and field guidance. Strip all HTML comments from generated plans.

## Greenfield

With no existing code, Current state becomes **Constraints** — stack, hosting, team, deadlines, things already decided elsewhere — and Precedent recon becomes a short look at whatever adjacent repos or prior art the user points to. Everything else is unchanged. Resist inventing an architecture before asking what is already fixed.
