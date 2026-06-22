---
name: factory
description: An autonomous development pipeline that takes a non-developer from a vague idea to a finished product. Stage 0 scores idea concreteness (0-100) and asks clarifying questions if it falls short, then designs through 5 stages (Profile -> Blueprint -> PRD -> Design -> Verify) and autonomously implements in the same session. Includes cross-stage consistency audits, anti-rationalization guardrails, evidence-based verification, high-risk adversarial review, source-driven decisions, and ADR documentation discipline. No external plugin dependency (critique uses Opus 4.8).
---

# Factory

A workflow that takes a user's rough idea, produces high-quality design artifacts, and autonomously implements them in the same session.

> This is an English reference translation of `SKILL.md` (the canonical, actively-loaded skill). If the two ever diverge, `SKILL.md` is the source of truth.

## Invocation
- `/factory <project idea>` — uses the argument directly as Stage 0 input
- `/factory` — if no argument, asks the user for an idea

## Execution Mode (hybrid)
- **Stage 0, 1**: requires user confirmation (asks for edits/approval)
- **Stage 2-5**: proceeds automatically (only calls the user when stuck or a major decision is needed)

## Output Path
All JSON/MD is saved under `./factory_output/<project_slug>/`.
- `project_slug` is extracted from the idea in snake_case (e.g. "battery monitoring" -> `battery_monitoring`)
- ADRs are saved as individual markdown files under `factory_output/<project_slug>/adr/` (see Stage 3).

## Critique Pattern (shared) — built-in critic protocol

Every TCI loop and consistency-audit critique step **must be delegated to a separate subagent** (`Agent` tool, `subagent_type: "general-purpose"`, `model: "opus"`). Self-critique by the current session evaluating its own work is forbidden (bias prevention).

> **No dependency**: instead of an external critic agent, the protocol below is injected directly as a prompt. Works in any environment.
> **Model**: critique needs top performance, so call with `model: "opus"` (Claude Opus 4.8). (If the calling environment's default Opus is already 4.8, `"opus"` is enough; use `claude-opus-4-8` if you need to be explicit.)

Pass the prompt below verbatim when calling (substitute only the `<...>` placeholders):
```
You are the Critic — not a helpful assistant, but the final quality gate.
A false approval (letting something flawed pass) costs 10-100x more than a false rejection. Your job is to stop resources from being spent on flawed work.
A standard review evaluates "what is there." You also evaluate "what is missing."

Target: <ARTIFACT_TYPE> from <STAGE_NAME>
<ARTIFACT_JSON>

Rubric: <RUBRIC>
Passing target: score >= <TARGET>

Follow this investigation protocol, in order:
[Phase 1 — Pre-mortem] Before detailed review, predict 3-5 places this work is likely to fail.
[Phase 2 — Verification] Check every claim, reference, and assumption against the actual content.
[Phase 3 — Multi-perspective] For code: security/junior-dev/ops. For plans/designs: implementer/stakeholder/skeptic.
[Phase 4 — Gap analysis] Explicitly find "what's missing" (not evaluating what's there — finding what isn't). This is the most important phase.
[Phase 4.5 — Self-audit] Rate each finding's confidence (HIGH/MED/LOW). Downgrade or move to open_questions anything low-confidence, refutable, or merely a preference.
[Phase 4.75 — Reality check] Check whether severity has been inflated. Re-calibrate against a realistic worst case and mitigating factors.

Severity: critical (blocks execution) / major (substantial rework) / minor (suboptimal but works). critical/major findings must include concrete evidence.
An unsupported claim is an "opinion," not a finding. Acknowledge strengths in one sentence each and move on — don't pad with praise.

Respond ONLY in this JSON format:
{
  "predictions": ["3-5 pre-mortem predictions"],
  "score": 0-10,
  "strengths": ["key strengths (concise)"],
  "weaknesses": ["..."],
  "whats_missing": ["gap analysis — what's missing"],
  "action_items": [{"priority": "critical|major|minor", "issue": "...", "evidence": "evidence", "recommendation": "concrete fix"}],
  "open_questions": ["low-confidence/refutable items"],
  "is_passing": true|false
}
```

Once the response is received, the current session parses it and proceeds to the improve step. (`predictions`, `whats_missing`, `open_questions` are reference-only; improvements come from `action_items`.)

## Idea Concreteness Scorer (Stage 0 only)

Factory's identity is **taking a non-developer from a single vague idea to a finished product**. So Stage 0 quantitatively scores whether the user's idea is "concrete enough to start building," and asks clarifying questions if it isn't.

This evaluation targets **user input** (not a factory artifact), so it isn't subject to self-critique bias, but for quality it's still delegated to a **separate subagent** (`Agent`, `subagent_type="general-purpose", model="opus"` = Opus 4.8). It uses a **dedicated prompt** with a different purpose/output than the critic prompt above.

Pass the prompt below when calling (substitute `<IDEA>`):
```
You are an "Idea Concreteness Evaluator." Assess whether the user's idea for what they want to build is concrete enough to start development, and generate questions to fill in what's missing.
The target user may be a non-developer, so avoid jargon in questions and prefer multiple-choice options that are easy to answer.

User idea:
<IDEA>

Score 5 dimensions, each 0-20 (total 0-100):
1. purpose (clarity of purpose) — why build this, what problem does it solve
2. target_users (who uses it)
3. core_features (core feature scope) — what does it do, is the scope defined
4. constraints (constraints/environment) — where does it run, what platform/budget/tech constraints
5. success_criteria — what does "done" look like

For any dimension that is weak (<=12 points), generate one question to fill that gap. Each question should include 2-4 easy-to-pick options for a non-developer, plus room for "other (free text)."

Respond ONLY in this JSON format:
{
  "dimension_scores": {"purpose": 0-20, "target_users": 0-20, "core_features": 0-20, "constraints": 0-20, "success_criteria": 0-20},
  "total_score": 0-100,
  "weak_dimensions": ["dimension keys scoring <=12"],
  "summary": "1-2 sentence summary of the idea's current state (in the user's language)",
  "clarifying_questions": [
    {"dimension": "dimension_key", "question": "plain-language question", "options": ["option1", "option2", "option3"]}
  ]
}
```

The current session receives `total_score` and `clarifying_questions` and runs the Stage 0 gate with them.

## Workflow

### Stage 0 — Idea Gate + Profile (interactive) ⭐ key differentiator

So that even a non-developer can start from a vague idea, factory first passes an **idea-concreteness score gate**, then extracts a profile.

**0-A. Score the idea's concreteness**
1. Send the user's idea to the "Idea Concreteness Scorer" (above) subagent (Opus 4.8) -> receive `total_score` (0-100), `weak_dimensions`, `clarifying_questions`.
2. Show the user a one-line summary of the score and weak dimensions. (e.g. `Idea concreteness score: 45/100 — target users and success criteria are unclear`)

**0-B. Gate (threshold = 70)**
- **score >= 70**: concrete enough -> proceed straight to 0-C (profile).
- **score < 70**: present options via `AskUserQuestion` (always put "proceed as-is" first):
  - ① **Proceed as-is** — AI fills the gaps with reasonable assumptions, states them explicitly, then proceeds to 0-C.
  - ② **Clarify via questions** — present `clarifying_questions` via `AskUserQuestion` (with multiple-choice options) -> update the idea with the answers -> **re-score** (repeat 0-A).

**0-C. Clarification loop guard**
- At most **3 rounds** of re-scoring. After 3 rounds, if still below 70, auto-switch to "proceed as-is" (with assumptions stated).
- The user may choose "proceed as-is" at any round (never force more questions).
- Non-developer consideration: at most **4 questions** at a time, multiple-choice preferred.

**0-D. Profile extraction** (after passing the gate)
From the now-concrete idea, extract:
- `project_category` (web / cli / agent / data_pipeline / library / ...)
- `risk_tier` (low / mid / high)
- `autonomy_level` (frequency of human involvement: high / mid / low)
- `target_users` (one-line description)
- `domain_hints` (list of keywords requiring domain knowledge)

-> Summarize in 4-6 lines for the user and ask **"let me know what to change or add."**
-> Save `stage0_profile.json` (including a `concreteness` field with score history: `{initial_score, final_score, rounds, assumptions_filled[]}`).

### Stage 1 — Blueprint (TCI loop + interactive)
Think -> Critique -> Improve, repeated:
1. **Think** (current session): draft blueprint — `goals`, `components` (name/purpose), `decisions`, `open_questions`
2. **Critique** (**delegated to subagent**): call `Agent(subagent_type="general-purpose", model="opus")` -> pass the built-in prompt from "Critique Pattern" above -> receive score + action_items
3. **Improve** (current session): rewrite the blueprint applying the action_items
- Exit condition: critic score >= 8, or 3 iterations reached
- RUBRIC: "clarity of goals, adequacy of component decomposition, specificity of open questions, justification of decisions"

-> Show the final blueprint and score history to the user and **request approval**.
-> Save `stage1_blueprint.json`, `stage1_history.json`.

### Stage 2 — PRD + Architecture (automatic, single TCI pass)
- **Knowledge Recall**: before writing architecture, consult `_knowledge/index.md` -> drill into <=5 relevant lessons (see "Knowledge Base" below).
- **PRD**: `product_goal`, `functional_requirements[]`, `non_functional_requirements[]`, `user_stories[]` (id/title/priority/acceptance_criteria)
- **Architecture**: `modules[]` (name/purpose/interfaces[]), `system_context`
- After drafting, **call the critique subagent once** -> receive action_items, current session improves
  - RUBRIC: "PRD-Blueprint consistency, requirement testability, measurability of non-functional requirements, clarity of module boundaries"
-> Save `stage2_prd.json`.

**Consistency Audit (1->2)** — call a general-purpose subagent (`Agent` tool, `subagent_type="general-purpose", model="opus"`):
```
Prompt:
"Audit the consistency between these two stages. Report only blocker-level inconsistencies.
Respond ONLY in JSON: {score: 0-10, blockers: [{issue, recommendation}]}

STAGE1_BLUEPRINT: <JSON>
STAGE2_PRD_ARCH: <JSON>"
```
If blockers exist:
1. Apply the recommendations to the PRD/architecture
2. Re-audit (at most 1 more time) — if blockers remain, report to the user

-> Save `consistency_1to2.json`.

### Stage 3 — Code Design (automatic, single TCI pass)
- **Knowledge Recall**: before writing the design, consult `_knowledge/index.md` -> drill into relevant gotcha/pattern pages -> feed into Source-Driven Decisions.
- `modules[]` (name/files/key_tests/implementation_notes)
- `interface_contracts[]` (caller/callee/signature/error_modes)
- `design_adrs[]` (decision/rationale/alternatives)
- `implementation_order[]` (step-by-step build order)
- `definition_of_done[]`
- `code_skeletons[]` (optional: key file signatures)
- After drafting, **call the critique subagent once** -> receive action_items, current session improves
  - RUBRIC: "completeness of interface contracts, dependency ordering of the implementation plan, rigor of ADR rationale, verifiability of definition of done"
-> Save `stage3_design.json`.

**ADR Document Separation (traceability)** — also emit each entry in `design_adrs[]` as an individual markdown file.
- Path: `factory_output/<project_slug>/adr/ADR-<NNN>-<slug>.md` (NNN starts at 001)
- Format for each ADR file:
  ```markdown
  # ADR-<NNN>: <decision title>
  - **Status**: accepted | superseded by ADR-NNN
  - **Date**: <YYYY-MM-DD>
  - **High-risk**: yes/no (yes + verdict if it went through High-Risk Review)

  ## Context
  <why this decision was needed>

  ## Decision
  <what was chosen>

  ## Alternatives
  <options considered and rejected, with reasons>

  ## Consequences
  <tradeoffs / impact>
  ```
- Keep a one-line-per-entry ADR index (number, title, status) in `adr/README.md`.
- If the design changes during implementation, add a new ADR and update the old one's status to `superseded` (never overwrite).

**High-Risk Adversarial Review (doubt-driven, conditional)** — if `design_adrs` includes any **high-risk decision** (DB schema, auth/authz, external dependency or library choice, data migration, security boundary), run a one-time adversarial review **separate from** the normal critique, via `Agent` (`subagent_type="general-purpose", model="opus"`).
```
Prompt: "Adversarially review this high-risk design decision. Start from the assumption that this decision is wrong, and attack it.
Respond ONLY in JSON: {decision, attack_vectors:[...], failure_scenarios:[...], must_fix:[{issue, recommendation}], verdict:'accept|revise|reject'}
DECISION: <ADR JSON>"
```
If verdict is `revise`/`reject`, fix the ADR and re-review (at most once more). Skip this step if there are no high-risk decisions.
-> Save `stage3_highrisk_review.json` (when run).

**Consistency Audit (2->3)** — same pattern, between Stage 2 architecture and Stage 3 design.
-> `consistency_2to3.json`.

### Stage 4 — Verification Plan (automatic)
- `test_matrix[]` (level: unit/integration/e2e, target, expected_outcome)
- `release_gates[]`
- `monitoring_plan[]`
- `rollback_plan`
-> Save `stage4_verify.json`.

**Consistency Audit (3->4)** — blockers are **warning-only** (no automatic remediation).
-> Save `consistency_3to4.json` + print a blocker summary to the screen.

### Stage 5 — Implementation (automatic, same session)
With Stage 1-4 outputs as context:
1. Register Stage 3's `implementation_order` via `TaskCreate`
2. **Thin vertical slices (incremental)**: each task covers exactly one concern, targeting ~100 lines of change. Split large tasks into smaller slices.
3. Implement each task -> run its `key_tests` -> **only proceed once you have passing evidence**
4. **Call the user when stuck or when a major decision is needed** (schema/auth/external dependency etc.)
5. Once everything is complete, run final verification against Stage 4's `test_matrix`
6. Append the final result + **Verification Evidence** to `SUMMARY.md`
7. **Knowledge Harvest**: right after the SUMMARY, accumulate 0-3 reusable lessons into `_knowledge/` (see "Knowledge Base" below).

**Non-negotiable Verification (evidence-based — most important)**
- "Looks like it passed" / "should work" is **not** evidence of completion. Only actual execution evidence counts.
- When each task completes, record the real output for whichever of these apply:
  - Build: command + exit code + tail of output (e.g. `npm run build` -> exit 0)
  - Tests: command run + pass/fail counts
  - Typecheck/lint: command + result
- A task with no tests written **must not be declared complete** (Red Flag).
- If an environment constraint prevented verification, never record a guessed pass — explicitly mark it **"unverified — reason"** and report it to the user.

## Source-Driven Decisions (no guessing — sources first)
**Before deciding on or writing code against** any library/framework/API/external dependency, check the actual source instead of relying on memory. (Applies throughout Stage 2 architecture, Stage 3 design, and Stage 5 implementation.)

Verification priority:
1. **Installed relevant skills** first (e.g. authoritative source skills like `vercel:nextjs`, `vercel:ai-sdk`, `vercel:shadcn`)
2. **Actual installed copy**: type definitions (`.d.ts`) in `node_modules`, exact version in `package.json`
3. If that's insufficient, **look up official docs** (e.g. via WebFetch)

Rules:
- Never assert API signatures, method names, option names, or versions **without verifying them**. (Ties into the "I already know the library version/API" entry under Anti-Rationalization.)
- If verification disagrees with memory, **always follow the actual source**.
- For high-risk decisions (subject to High-Risk Adversarial Review), this source check is a **mandatory prerequisite**.
- Record the key verified facts (version, signature, etc.) as evidence in the relevant ADR or design_notes.

## Knowledge Base (cross-project compounding)
Every time factory finishes a run, it accumulates reusable lessons, and new runs consult them. This applies the Karpathy "LLM Wiki" pattern — knowledge compounds across projects instead of evaporating.

**Location**: `factory_output/_knowledge/` (alongside the project directories, shared across all projects)
- `index.md` — catalog of all pages (one line per page). `log.md` — chronological history (append-only). `pages/` — individual lessons (lesson/gotcha/pattern).

**Context-bloat guard (mandatory discipline)**: accumulation lives on disk; only **`index.md` + at most 5 relevant pages** ever enter context. Never read the whole KB at once -> context cost stays roughly constant regardless of KB size. If `index.md` grows too large, split by category and use Lint to supersede stale pages.

**① Recall** — performed at Stage 0-D (once domain_hints are finalized), and at the start of Stage 2 architecture / Stage 3 design:
- Read only `_knowledge/index.md` first -> drill into **at most 5** pages relevant to the current project.
- Use found lessons as input to Source-Driven Decisions; when applied, cite the source (`_knowledge/pages/<page>`) in design_notes/ADR.
- If there's no KB yet (first run) or no relevant pages, silently skip.

**② Harvest** — performed right after writing the SUMMARY at the end of the run:
- Extract only **reusable** lessons from this run (library gotchas, validated decision patterns, security/architecture principles). Exclude project-specific trivia.
- Cap at **0-3** pages per run (to avoid noise). If a lesson overlaps an existing page, update it rather than creating a new one.
- Save each page to `pages/` in the format defined by `_knowledge/README.md` (including frontmatter) -> update the `index.md` catalog -> append a `harvest` entry to `log.md`.

**③ Lint (periodic check)** — when the KB has roughly 15+ pages, or the user requests it: check for contradictions, stale facts, orphaned pages, and duplicates; update or supersede them (mark `status: superseded by ...` instead of deleting). Log a `lint` entry in `log.md`.

## Anti-Rationalization (blocking excuses to skip stages)
During autonomous execution, **reject all** of the following excuses if they come up. Giving in to excuses is factory's most common failure mode.

| Excuse | Rebuttal |
|------|------|
| "This is simple, let's skip Stage 1/Blueprint" | ❌ Things that look simple are the most frequently mis-designed. The prior stage is mandatory. |
| "Skip the critic call, I'll evaluate it myself" | ❌ Self-critique bias. Always use a separate subagent (general-purpose + opus). |
| "Tests/verification later, declare implementation done for now" | ❌ Never declare Stage 5 done without Stage 4 verification evidence. |
| "Consistency audit is a waste of time" | ❌ Cross-stage inconsistencies blow up during implementation. The audit is mandatory. |
| "Record it as passing as if I'd run the build/tests" | ❌ Never record "pass" without actual execution output (logs). |
| "I already know the library version/API, no need to check" | ❌ No guessing. If uncertain, check the actual file/docs before deciding. |

## Red Flags (signs you're going wrong — fix immediately)
- Blueprint has only 1 `component` -> insufficient decomposition, rewrite
- PRD requirements use unmeasurable language ("fast", "good", "easy") -> rewrite with quantitative criteria
- `acceptance_criteria` is empty or just "it works" -> rewrite in verifiable form
- A single task/commit exceeds ~500 lines or mixes multiple concerns -> split into thin vertical slices
- Stage 5 declares "done" with no tests -> treat as incomplete
- Critic score stays the same round after round -> improvement has stalled, stop immediately and report to the user

## Rules
- **Every critique is delegated via the `Agent` tool to a general-purpose subagent** (`general-purpose` + `model="opus"`) — both inside TCI loops and for cross-stage consistency. Use the built-in prompt from "Critique Pattern" above. No self-critique, no external plugin dependency.
- Think/Improve/Implement are performed directly by the current session (shared context)
- At the start of each stage, report progress in one line: `[Stage 2] Writing PRD...`, `[Stage 2 Critique] Calling critic subagent...`
- At any step requiring user confirmation, explicitly say "let me know what to change or add"
- Output JSON: keys in English, values may be in the user's language
- All stages run in the same session — no separate executor handoff
- If the TCI loop's score stops improving, exit immediately (prevent infinite loops)
- If the critic's response fails to parse as JSON, call it once more; if it fails a second time, report to the user

## Output
- **At the end of each stage**: a 3-5 line summary of key decisions + saved file paths
- **At the very end**: the `SUMMARY.md` path + implementation results + suggested next actions

## SUMMARY.md Template
```markdown
# <project_name>

## Overview
<Stage 2 product_goal>

## Stage-by-Stage Summary
- **Stage 0 Idea Gate**: concreteness score <initial>-><final>/100 (<N> rounds), <count> assumptions filled
- **Stage 0 Profile**: <category/risk/autonomy>
- **Stage 1 Blueprint**: final score <N>, <count> components
- **Stage 2 PRD**: <N> functional requirements, <N> stories
- **Stage 3 Design**: <N> modules, <N> implementation steps, <N> ADRs (see `adr/`)
- **Stage 4 Verify**: <N> tests, <N> gates
- **High-Risk Review**: <ran? / verdict / must_fix count> (if any high-risk ADRs)

## Consistency Audit Results
- 1->2: <score>, blockers: <count>
- 2->3: <score>, blockers: <count>
- 3->4: <score>, blockers: <count>

## Implementation Results
<completed tasks / remaining tasks / issues found>

## Verification Evidence
- Build: <command -> exit code / output summary>
- Tests: <command -> N passed / N failed>
- Typecheck/lint: <command -> result>
- Unverified items: <item + reason (if any)>

## Knowledge Base
- Recall: <list of referenced lesson pages, or none>
- Harvest: <list of accumulated lessons, or none> (`factory_output/_knowledge/`)

## Next Actions
<recommended follow-up work>
```
