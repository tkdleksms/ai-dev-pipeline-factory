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

Using factory on a small task becomes **excessive process**. It's suited for "build it properly, use it for a long time."

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
