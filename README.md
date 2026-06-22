# factory

An autonomous development pipeline skill for Claude Code: take a single rough idea from a non-developer to a verified, working product in one session.

- **Korean (canonical, actively used)**: [SKILL.md](SKILL.md) · [GUIDE_for_beginners.md](GUIDE_for_beginners.md) · [GUIDE_advanced.md](GUIDE_advanced.md)
- **English (reference translation)**: [SKILL.en.md](SKILL.en.md) · [GUIDE_for_beginners.en.md](GUIDE_for_beginners.en.md) · [GUIDE_advanced.en.md](GUIDE_advanced.en.md)
- **Real run example**: [examples/stock_guidance_kr/](examples/stock_guidance_kr/) — full Stage 0–4 design output (idea gate, blueprint, PRD, architecture, code design, ADRs, verification plan) plus `SUMMARY.md` with build/test evidence.

## What it does

1. **Stage 0 — Idea Gate**: scores idea concreteness (0–100) across 5 dimensions and asks clarifying questions if it's too vague to design from.
2. **Stage 1 — Blueprint**: Think → Critique → Improve loop, with critique delegated to a separate subagent (no self-grading).
3. **Stage 2 — PRD + Architecture**, **Stage 3 — Code Design** (with ADRs and high-risk adversarial review), **Stage 4 — Verification Plan**, each audited for consistency against the previous stage.
4. **Stage 5 — Implementation**: builds in thin vertical slices, with every "done" backed by actual build/test execution evidence — never a guess.

See [GUIDE_for_beginners.en.md](GUIDE_for_beginners.en.md) for a plain-language walkthrough, or [GUIDE_advanced.en.md](GUIDE_advanced.en.md) for the design rationale and the 7 hardening rules (anti-rationalization, evidence-based verification, source-driven decisions, etc.).
