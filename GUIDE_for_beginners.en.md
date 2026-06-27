# Understanding the factory Skill, the Easy Way (for coding beginners)

## In one sentence

> **Feed it one rough idea, and it designs, reviews, and builds it into actually-working code — an automatic factory.**

That's why it's called `factory`. Put in raw material (an idea), get out a finished product (a working app).

---

## Why does this exist? (what happens without it)

If you just tell an AI "build me a webnovel app":
- It dumps out code with no design first -> the structure gets tangled later
- It grades its own work as "good" -> no objectivity
- It says "this should work" and then it doesn't
- You can't tell what got skipped along the way

`factory` is a rule set built to prevent this, by forcing the AI to follow the same order a human team would: plan -> design -> verify -> build.

---

## The 5 stages — as a house-building analogy

| Stage | What it does | House-building analogy |
|------|--------|------------|
| **0. Profile** | Figures out what kind of project this is | "House? Building? What's the budget?" |
| **1. Blueprint** | Designs the big picture: goals, components | Architectural blueprint |
| **2. PRD** | Nails down concrete feature requirements | "3 bedrooms, 2 bathrooms" detailed spec |
| **3. Design** | Designs code structure, files, build order | Construction drawings (plumbing, wiring) |
| **4. Verify** | Plans how to test it | Final-inspection checklist |
| **5. Build** | Actually writes the code + verifies it | Actual construction + inspection |

**Only stages 0 and 1 need a human to check in** — the rest run automatically. (Just point it in the right direction and it handles the rest.)

---

## factory's 3 clever mechanisms

### 1) No self-grading (TCI loop)
```
Draft (Think) -> a different AI scores it (Critique) -> fix it (Improve)
```
- If you grade your own work, you grade generously.
- So a **separate "critic-only" AI** is called in to score it 0-10, and it keeps getting fixed until it scores 8+.

### 2) Cross-stage consistency checks (Consistency Audit)
- What if the blueprint (stage 1) and the detailed spec (stage 2) don't match? -> that blows up in the code later.
- So **every time a stage finishes**, it's checked against the previous stage for conflicts.

### 3) "No excuses" + "show evidence" (recently added)
- ❌ "This is simple, let's skip the design" -> rejected
- ❌ "The build should probably pass" -> rejected. You have to show an **actual build log** before it's "done."

---

## What gets produced

Everything is saved step by step under `factory_output/<project_name>/`:
```
stage0_profile.json     <- Stage 0 result
stage1_blueprint.json   <- blueprint
stage2_prd.json         <- feature spec
stage3_design.json      <- code design
stage4_verify.json      <- test plan
adr/                    <- record of important decisions (why this tech was chosen)
SUMMARY.md              <- overall summary
```
And it also produces **actually-working code**.

---

## How do you use it?

Just type this in the chat:
```
/factory build me a battery schedule management dashboard
```
Or just type `/factory` with nothing after it, and it asks "what should I build?"

---

## When you want to change it after building (Maintain mode)

An app isn't built once and done. As you use it, "add one more button here," "fix this bug" keep coming up. For that:
```
/factory maintain add a button to toggle dark mode
```
Calling it this way puts factory into **fix-it mode**. Instead of rebuilding from scratch, it's a flow for **safely tweaking an app that already exists**.

What factory does on its own here (in plain terms):
1. **It reads what the code actually is right now first** — if you've hand-edited things since, the code has drifted from the original design notes, and it reconciles that gap first. *(This is the key part — fixing against stale design notes makes you touch the wrong place.)*
2. **It decides only what to touch** — for "dark mode," just the screen-theme area. It leaves the parts that work alone.
3. **After fixing, it re-checks that what used to work still works** — it verifies with tests that existing features didn't break, not just the new thing.

Your job is the same as when building — **just the direction**: to "I'll change this part like so, shall I proceed?" you answer "yes" or "not like that, like this." (If the change is low-risk the process stays light; if it's risky like auth or payments, it gets more thorough — it scales the rigor for you.)

---

## Checking it was built right, with a score (Evaluation)

Once it's built, you naturally wonder "did this actually turn out the way I first intended?" After finishing, factory produces a **scorecard for itself (EVALUATION.md)**. You can also call it anytime:
```
/factory evaluate
```
One important point — **if you grade your own work, you grade generously.** So the grading is done by a **different AI (a separate grader)**, and every score comes with **"the evidence for why"** (which tests passed, etc.) attached. A score with no evidence is void.

The grading looks at four things: ① do the features you wanted actually all work, ② was it built the way it was originally designed, ③ is it confirmed by tests, ④ were the risky parts handled well. And the result isn't "an N-out-of-10 code" but **"how much of the design got realized"** (done / partially done / drifted off).

Another nice part — every time you fix it with Maintain, it records **whether the score went up or down vs. last time** (the trend), so if a fix actually made something worse, it tells you.

---

## A real example (already built)

On this account, calling `/factory Korean webnovel creation system`:
- Ran the 5-stage design automatically (design score auto-improved from 5 to 8)
- Produced an actually-working Next.js web app called **webnovel-creator**
- That app is now being used to write a real webnovel ("Breaking Things Is My Job")

In other words: one sentence, "build me a webnovel app" -> an app you actually use.

---

## 3 things beginners should remember

1. **You only need to set the direction** — at stages 0 and 1, just answer "yes that's right / please fix this," and the rest is automatic.
2. **The result leaves a paper trail** — later, when you ask "why was this built this way?", check the `factory_output/` folder.
3. **It doesn't trust "it's done" — it looks for evidence** — build logs and test results are kept in the SUMMARY, so you can confirm it actually works.

---

## One-line summary (again)

> Rough idea -> (design, review, verification, all automatic) -> code that actually works.
> The human sets the big direction; factory handles the rest of the finicky process, with discipline.
