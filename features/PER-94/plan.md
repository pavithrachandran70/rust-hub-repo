# Plan — PER-94 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Single-repo bug fix in rustc match lowering for float pattern constants with a clear reproduction case and acceptance criteria; quick_fix loop fits.

## Steps

1. [ ] **analyst (inline_story)** — Convert the ticket into an implementable story with Given/When/Then acceptance criteria and technical notes.
   - focus: Focus on the match lowering logic for float patterns with guards; include technical notes on the suspected regression from PR #122459 and the need to preserve distinct float constants (-0.0 vs 0.0) after guard failures.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Ticket clearly states scope, acceptance criteria, and what is out of scope; no product ambiguity.
- **test_architect** — Single straightforward story with acceptance criteria that map directly to test cases; no edge case elaboration needed.
- **critic** — Only one spec document (the story); no cross-artifact consistency to check.

## Planner notes

- attempt 1: designer is not a step of loop quick_fix
