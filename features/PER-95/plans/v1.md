# Plan — PER-95 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Type-checking bug where a never-evaluated `!`-typed constant in a zero-length array causes incorrect unreachable code elimination; needs requirements, design, and thorough testing due to soundness implications.

## Steps

1. [ ] **analyst (inline_story)** — The ticket clearly describes the bug with reproduction steps; one inline story captures the fix requirements and acceptance criteria.
   - focus: Focus on the type-checking behavior of never-type constants in zero-length arrays, the divergence analysis bug, and the safety invariant violation (reading past Vec bounds).
2. [ ] **designer (lld)** — Compiler bug fix needs a concrete implementation plan identifying which compiler passes are involved and how divergence analysis should handle zero-length array constants.
   - focus: Map out which rustc passes need changes (likely type-checking/divergence analysis), specify how `[C; 0]` with `C: !` should be handled, and identify test cases covering both the `!` type directly and via associated types.
3. [ ] **test_architect (tests)** — Safety-critical compiler bug requires comprehensive test coverage including the two variants shown, debug vs release builds, and edge cases.
   - focus: Ensure tests cover: the trait-based `!` pattern, the direct `[!; 0]` pattern, both debug and release modes, and verification that the bounds check is not eliminated.
4. [ ] **critic (consistency_check)** — Single inline story with design and tests produces multiple artifacts that must agree on the bug's root cause and fix approach.
5. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
6. [ ] **dev** _(auto)_ — Loop delivery.
7. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
8. [ ] **open_pr** _(auto)_ — Loop delivery.
9. [ ] **pr_review** _(auto)_ — Loop delivery.
10. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope is clear: fix a specific type-checking bug; no product ambiguity or user-facing scope decisions needed.
