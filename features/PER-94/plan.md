# Plan — PER-94 (v2)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Single-repo bug fix in rustc match lowering for floating-point pattern handling where -0.0 arm is incorrectly skipped; behavior is clearly specified in the ticket so quick_fix loop fits.

## Steps

1. [ ] **analyst (inline_story)** — Convert ticket into implementable story with AC and technical notes for the match lowering fix.
   - focus: Focus on the IEEE float equality semantics (0.0 == -0.0 but distinct pattern constants), the regression from PR #122459, and the constraint that Eq candidate sorting must not discard the -0.0 arm when the 0.0 guard fails.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story produces one artifact with no cross-document consistency to check.
