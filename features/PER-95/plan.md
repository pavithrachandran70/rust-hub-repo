# Plan — PER-95 (v2)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

A type-checker soundness bug where zero-length array constants of type `!` incorrectly mark subsequent code as unreachable; needs an inline story with low-level design and test cases.

## Steps

1. [ ] **analyst (inline_story)** — Single-repo compiler bug needs clear requirements with acceptance criteria for the fix.
   - focus: Focus on the type-checker's treatment of `!`-typed constants in zero-length array contexts; the divergence analysis must not affect reachability when the constant is never evaluated.
2. [ ] **designer (lld)** — Compiler bug fix needs concrete file plan identifying affected type-checking and MIR generation code paths.
   - focus: Identify the type-checking phase that incorrectly propagates divergence from unevaluated constants, and the MIR lowering that marks subsequent code unreachable; include safeguards to prevent similar issues.
3. [ ] **test_architect (tests)** — Soundness bug with multiple manifestations needs comprehensive test coverage including the provided examples and edge cases.
   - focus: Cover both the trait-based and direct `[!; 0]` patterns; test debug and release builds; verify no SIGILL and correct runtime behavior when bounds check fails.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no cross-artifact dependencies to check.
