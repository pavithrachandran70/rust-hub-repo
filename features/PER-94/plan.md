# Plan — PER-94 (v1)

- made from: ticket
- source: LLM planner
- complexity: moderate
- needs plan review: yes
- track hint: standard

This is a targeted compiler bug fix in match lowering for float patterns with a known root cause and clear reproduction; analyst inline_story alone produces the implementable story with detailed AC and tech notes from the ticket.

## Steps

1. [ ] **analyst (inline_story)** — Single behavioral fix in one repo with detailed requirements, reproduction, cause hypothesis, and acceptance criteria already in the ticket.
   - focus: Preserve the upstream issue link, reproduction case, cause hypothesis pointing to PR #122459, and all five acceptance criteria as-is; tech notes should reference the match lowering code path and the float equality edge case.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Fixed execution step.
4. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
5. [ ] **open_pr** _(auto)_ — Fixed execution step.
6. [ ] **pr_review** _(auto)_ — Fixed execution step.
7. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Scope is unambiguous: fix a compiler bug in match lowering with clear reproduction and acceptance criteria.
- **architect** — No new components, API contracts, or cross-repo changes; this is a localized bug fix in existing match lowering logic.
- **scrum_master** — Work is contained in a single story for one repo with no breakdown needed.
- **designer** — The ticket provides cause hypothesis and points to the likely regression; implementation approach is sufficiently clear for Dev.
- **test_architect** — Acceptance criteria already include adding a regression test; the reproduction case is the test template.
- **critic** — Only one spec artifact (the inline story) will be produced; nothing to cross-check.
