# Understanding the factory Skill One Level Deeper (intermediate + hardening features)

> This document assumes you've read the basics (`GUIDE_for_beginners.md`), and explains
> **how factory actually works** and the **7 hardening rules added afterward**.

---

## Part 1. factory's skeleton — "why this structure"

### Core philosophy: LLMs are smart, but lazy and overconfident

The 3 big failure patterns in AI coding:
1. **Skipping** — "this is simple, let's skip the design"
2. **Self-confidence** — grading its own output as "looks good"
3. **False completion** — "this should work," without actually running it

factory's goal is to make these 3 things **structurally impossible**. It isn't just "a pile of prompts" — it's a **discipline system**.

### What the 5 stages + TCI + Audit mean

```
Stage 0-5 (vertical flow)             = "enforces order" -> can't write code without a design
TCI loop (inside each stage)          = "enforces quality" -> can't move to the next stage below an 8
Consistency Audit (between stages)    = "enforces coherence" -> can't pass with contradictions
```

The key piece is **delegating the TCI Critique step to a separate subagent** (general-purpose + Opus 4.8, with the critic protocol built in).
- If the same AI grades its own work -> bias (scores too generously)
- If a different AI grades it -> objective scores + concrete weaknesses pointed out
- This is called "self-critique blocking."

---

## Part 2. The 7 hardening rules added on top (borrowed from addyosmani/agent-skills)

> Background: building webnovel-creator on this account hit **two real incidents**.
> ① Build verification wasn't enforced by a rule, so it got waved through with "should be fine"
> ② Code was written relying on **memory** of the Next.js version / AI SDK API, and it broke the build (`CoreMessage` -> the real export was `ModelMessage`)
>
> These 7 rules were added to structurally prevent that class of incident.

### ① Anti-Rationalization (blocking excuses)

**Problem:** AIs are good at inventing plausible-sounding excuses to skip a stage.
**Fix:** Pre-write the 6 most common excuses, each paired with a rebuttal.

```
"This is simple, let's skip Stage 1"       -> ❌ Things that look simple are most often mis-designed
"Skip the critic, I'll grade it myself"    -> ❌ self-critique bias
"Tests later, declare it done for now"     -> ❌ no completion without verification evidence
```

> Analogy: taping a list of "just this once..." excuses to the fridge while dieting.

### ② Non-negotiable Verification (evidence-based) ⭐ most important

**Problem:** "looks like the build passed" can be a lie.
**Fix:** "Looks like" is **never accepted** as completion evidence. Only actual execution output counts.

```
❌ "Running npm run build should work"
✅ "npm run build -> exit 0, dist/ created" (actual log attached)
```

- A task with no tests written **cannot be declared complete at all** (classified as a Red Flag)
- If an environment constraint blocks verification -> never record a guessed pass; state "unverified — reason" explicitly

> In this session, both webnovel-creator and the battery dashboard left build logs (exit 0) in their SUMMARY — that's this rule in action.

### ③ Red Flags (self-diagnosing warning signs)

**Problem:** the AI can be going off the rails without noticing.
**Fix:** keep a list of "if this happens, something's wrong" signals and self-check against it.

```
- Blueprint has only 1 component -> insufficient decomposition
- Requirements use unmeasurable words like "fast/good/easy" -> rewrite
- A single task exceeds 500 lines -> too big, split it
- Critic score stays flat round after round -> improvement stalled, stop immediately
```

### ④ High-Risk Adversarial Review (attacking high-stakes decisions)

**Problem:** a normal review only checks "does this look okay." Not enough for decisions that matter.
**Fix:** for decisions that are hard to reverse (DB schema, auth, external libraries), separately run an "attack why this is wrong" mode.

```
Normal critique: "what score does this design get?"
Adversarial review: "find 3 scenarios where this DB schema fails"
                  -> verdict: accept | revise | reject
```

> Normal review = health checkup / adversarial review = penetration test. Only pen-test the parts that matter.

### ⑤ Incremental — thin vertical slices

**Problem:** if the AI dumps hundreds of lines at once, you can't verify it.
**Fix:** split work into **one concern at a time, ~100 lines** at a time.

```
❌ "the whole login feature in one shot"
✅ slice it: "1) DB table -> 2) one API -> 3) one UI piece," verifying each
```

The smaller the slice, the faster you spot exactly where it broke.

### ⑥ Source-Driven Decisions (no guessing, sources first) ⭐ prevents the incident from recurring

**Problem:** the AI asserts a library's API/version **from memory** -> reality differs -> errors.
**Fix:** **check the actual source** before writing code.

```
Verification priority:
1. Installed relevant skills (vercel:nextjs, vercel:ai-sdk, etc.)
2. Actual type definitions (.d.ts) in node_modules, version in package.json
3. If still uncertain, look up official docs
```

> Checking via grep whether the MSAL NAA API (`createNestablePublicClientApplication`) actually exists
> in `node_modules` before writing code against it is exactly this rule in practice.
> It preempts version incidents like Next.js 16 / `ModelMessage`.

### ⑦ ADR — documenting decisions

**Problem:** "why did we pick SQLite?" — nobody remembers 3 months later.
**Fix:** every important technical decision gets its own markdown file (`adr/ADR-001-xxx.md`).

```
What goes into one ADR:
- Context (why the decision was needed)
- Decision (what was chosen)
- Alternatives (rejected options + why)
- Consequences (tradeoffs)
```

If the design changes, the old ADR isn't overwritten — it's marked "superseded," so **the history of decisions stays traceable.**

---

## Part 2.5. Four operational enhancements — "survive interruption, don't over-do it, make it visible"

> If the 7 above are *quality* rules, these 4 are about **actually running** factory in practice.
> Two small primitives (**Pipeline Profile**, **Run State**) solve all four at once.

### Ⓐ Pipeline Profile — risk-proportional depth (prevents over/under-ceremony)

**Problem:** running full ceremony (3 critique rounds + 3 audits + high-risk review) on a one-line tool is waste. Conversely, reviewing a payment system shallowly is an accident.
**Fix:** at Stage 0, pick one profile by risk level and auto-tune the depth of later stages.

```
lite      (low-risk, small) : 1 review round, audits merged into 1, slice critique skipped
standard  (default)         : 3 review rounds, audits each       (= original behavior)
fortified (high-risk)       : 3 rounds + re-audit, forced high-risk review, every slice critiqued
```

> Key: the profile reduces **depth only**. lite still goes through all of Stages 0-5 (it never skips a stage). A completely different concept from "it's simple, let's skip it" (Anti-Rationalization).

### Ⓑ Resume — crash recovery

**Problem:** a long autonomous run cut short by context exhaustion/interruption means starting over.
**Fix:** checkpoint progress to `_run/state.json` at every stage boundary -> `/factory resume` continues from where it stopped (never re-runs completed stages).

> Analogy: a game's autosave. It writes how far you got to disk, so even after a power-off you start from that point.

### Ⓒ Run Telemetry / RUN_REPORT — observability

**Problem:** after a run, "how much review happened and what did it cost" is invisible.
**Fix:** log key events (critique scores, audits, subagent call counts, etc.) one line at a time to `_run/metrics.jsonl`, then aggregate into `RUN_REPORT.md` at the end.

> SUMMARY.md = *what was built* (results) / RUN_REPORT.md = *how / at what cost it was built* (process, cost).
> For a portfolio, it shows "what this pipeline actually did" in numbers.

### Ⓓ Stage 5 hardening — "the last mile"

**Problem:** the design stages are meticulous, but implementation was just "thin slices + evidence," comparatively loose.
**Fix:** profile-linked per-slice critique + integration-test checkpoints at module boundaries + an **implementation->design feedback loop** (on finding a design defect mid-build, don't quietly diverge — add an ADR, patch the design, re-review if needed).

---

## Part 2.6. Maintain mode — "factory after the first release"

> Everything above is **Build mode** (greenfield 0→1). But in practice you finish a build and then keep tweaking it with Claude Code — and factory had nothing to say about that phase. `/factory maintain` fills it.

**The core problem it solves — drift.** Every hand-edit after the build pushes the **actual code** away from the **design docs** (`stage3_design.json`, ADRs). If a later change plans against those stale docs, it plans against a fiction. So Maintain mode's first real step (M1) is to **read the actual code and reconcile it** with the design — but only in the area the change touches, never the whole project.

```
M0 Locate & Load    : find the project, load its artifacts (or onboard a generic repo)
M1 Reconcile  ⭐     : diff real code vs. design docs in the touched area -> drift list -> fix ADRs
M2 Scope            : classify the change, fix its blast radius, pick a profile  (you confirm)
M3 Change Design    : mini TCI for just this change (contracts, ADRs, test delta, regression risk)
M4 Implement  ⭐     : thin slices within the blast radius + regression guard on existing tests
M5 Update + Harvest : patch design docs, append a Maintenance Log line, harvest KB
```

Three things are genuinely new; the rest is Build-mode machinery reused at smaller scale:
- **M1 drift reconciliation** — the exact gap in the "build once, hand-edit forever" workflow.
- **M2 blast-radius scoping** — touch only what the change needs; nothing outside the set.
- **M4 regression guard** — record a baseline of affected tests *before* the change, re-run them after each slice so you can't silently break existing behavior.

> Analogy: Build mode is constructing the house. Maintain mode is a renovation — you first check what was actually built (vs. the original blueprint), wall off just the room you're touching, and make sure the plumbing elsewhere still works when you're done.

---

## Part 2.7. Evaluation — "design realization" acceptance grading + longitudinal trend

So far factory covered **building** and **fixing**, but it had no step that actually grades **"how much of the design the build realized."** Evaluation fills that gap. End of Build (Stage 6) · end of Maintain (M6 delta) · anytime (`/factory evaluate`) — **one grading engine, three triggers**.

**Why this is tricky — the self-praise trap**
If factory calls its own code "9/10, excellent," that's just the overconfidence bias this skill *exists to prevent* (②·③), dressed up as a score. So two things are mandatory:
1. **Grading is done by a separate Opus subagent, not the current session** (same principle as the Stage 1 critic — no self-grading).
2. **Every score is anchored to an evidence path.** A score given on "impression" is void (Red Flag).

**Four axes (each 0–10, evidence required)**
| Axis | What it measures | Evidence source |
|----|------------|----------|
| Requirements met | ratio of Stage 2 requirements·acceptance criteria proven by tests | test/run logs |
| Design conformance | code ↔ stage3_design drift (← **reuses the M1 drift detector**) | justified if an ADR explains it, penalized otherwise |
| Verification strength | acceptance-criteria test coverage + build/type/lint | reuses Stage 5 verification evidence |
| Risk handling | high-risk decisions' High-Risk Review pass·outstanding must_fix | stage3_highrisk_review.json |

**The point — the verdict is a "realization," not a number**
The overall result isn't one weighted-average number but a **realized / partially-realized / diverged** verdict + a per-axis table. It reframes the subjective "did it get better than the initial design?" into the measurable **"what % of the design was realized, and with what evidence?"**

**Longitudinal trend**
Each evaluation appends one line to `_run/eval_history.jsonl` → every Maintain fix yields a **delta vs. the previous** (improved/unchanged/regressed). Quality can regress even when tests pass (separate from the M4 regression guard), so if any axis drops, it's reported to the user.

> Analogy: if Build·Maintain are "constructing and renovating," Evaluation is the **final inspection + the periodic-inspection log**. An inspector (the separate subagent) grades the real thing against the blueprint with evidence, and that log accumulates to show whether the building got better or worse over time.

---

## Part 3. The full picture — before vs. after hardening

```
[Original factory]
5 stages automated + TCI loop + consistency checks
-> "designs well, but implementation verification is loose and guessing causes errors"

[Hardened factory]
        + Anti-rationalization   (prevents laziness)
        + Evidence verification  (prevents false completion) ⭐
        + Red Flags              (self-diagnosis)
        + Adversarial review     (defends high-risk decisions)
        + Thin slices            (verifiability)
        + Source-first           (prevents version/API guessing errors) ⭐
        + ADR records            (decision traceability)
-> "designs well, verifies implementation with evidence, and preempts guessing errors"
```

---

## Part 4. When to use it, when not to

| Good fit | Bad fit |
|------|--------|
| New 0->1 projects | One-line bug fixes |
| Apps/tools where structure matters | One-off scripts |
| Things you'll maintain later | Throwaway prototypes |

Using factory on a small task becomes **excessive process**. It's suited for "build it properly, use it for a long time." (That said, the **lite profile** makes the process much lighter for small/low-risk projects — see Part 2.5 Ⓐ.)

Note the table above is about **Build mode** (0→1). Once a project exists, ongoing bugfixes and feature additions go through **Maintain mode** (`/factory maintain`, Part 2.6) instead — which is itself profile-scaled, so a small fix stays a lite-weight pass.

---

## Part 5. The key differentiator — the idea-concreteness gate (Stage 0)

> factory's identity: **"take a non-developer from one vague idea to a finished product"**

Most AI coding tools just fill in assumptions for a vague request and start generating code. factory doesn't.

```
1. Idea input
2. A dedicated Opus 4.8 evaluator scores 5 dimensions
   (purpose, target, features, constraints, success criteria, 0-100)
3. score < 70 -> the user chooses:
   ① "Proceed as-is" (AI fills assumptions + states them)
   ② "Clarify via questions" (multiple-choice questions -> re-score, up to 3 rounds)
4. score >= 70 -> moves into full design
```

**Why this is differentiating:**
- The score is **a number**, so "how concrete is my idea" is objectively visible
- Questions are **mostly multiple-choice** -> easy even for non-developers to answer
- "Proceed as-is" is always offered -> never traps the user (helps, but never forces)
- This is agent-skills' interview/refine pattern, **quantified into a score gate and built into Stage 0**

> Analogy: an architect, given a vague commission, first asks "what's the budget? how many people will live here? by when?"
> to bring the request up to a level where a blueprint can actually be drawn.

---

## One-line summary

> If the original factory was a **"factory that enforces design,"**
> the hardened factory is a **"factory that enforces design + evidence-based verification + blocking guesswork."**
> All 7 rules exist to stop the AI from acting smart and skipping steps, or guessing.
