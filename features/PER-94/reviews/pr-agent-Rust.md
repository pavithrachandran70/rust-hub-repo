```json
[
  {
    "path": "compiler/rustc_mir_build/src/builder/matches/buckets.rs",
    "line": 349,
    "body": "**AC gap**: The new logic correctly handles `-0.0` vs `0.0` as distinct patterns when both are zeros, but the `float_zeros_share_eq_test` helper does not validate that the values are actually different bit patterns. AC-1 and AC-2 require that when the `0.0 if b` guard fails, both `f(0.0, false)` and `f(-0.0, false)` match the `-0.0` arm. The current implementation returns `Some(TestBranch::Success)` for **both** `0.0` and `-0.0` when tested against a `0.0` pattern constant, which is correct for the IEEE equality semantics. However, the logic does not explain why the `-0.0` arm is tried at all after the `0.0` guard fails — the candidate sorting/filtering logic that was the suspected root cause is not visible in this diff. The fix appears to make both zeros succeed the same test, which would cause both to match the first arm with a passing guard (AC-3, AC-4), but it's unclear how a failed guard on `0.0` now routes to the `-0.0` arm instead of skipping it. The requirements state the bug was in candidate filtering after guard failure, but this diff only changes the test branching logic for floats. Either the candidate filtering change is missing from the diff, or the fix is incomplete."
  },
  {
    "path": "compiler/rustc_mir_build/src/builder/matches/buckets.rs",
    "line": 400,
    "body": "**Verification gap**: The `float_zeros_share_eq_test` helper correctly identifies when both values are floating-point zeros (including opposite signs), but it does not validate that one is positive and one is negative. The function name and comment imply it detects when `0.0` and `-0.0` share an equality test outcome, but the implementation just checks `is_zero()` on both — it would return `true` for `(0.0, 0.0)` or `(-0.0, -0.0)` as well. For the bug fix, we need to know that the test constant and the case constant are **different** zeros (one positive, one negative), otherwise we're treating `0.0` tested against `0.0` the same as `0.0` tested against `-0.0`. The logic should verify `test_leaf.to_fXX().is_zero() && case_leaf.to_fXX().is_zero() && test_leaf != case_leaf` (or check the sign bit explicitly). As written, this would cause both `0.0` and `-0.0` to take the success branch when tested against the same constant, but does not explain how the other zero's arm becomes reachable after a guard failure."
  },
  {
    "path": "tests/ui/match/float-pattern-guard-zero.rs",
    "line": 1,
    "body": "**AC-6 coverage gap**: The regression test covers AC-1 through AC-5 via assertions, which is correct. However, AC-6 requires 'a new regression test covering float pattern matching with guards exists and passes' — the test should include a comment linking it to the bug report or the original ticket (PER-94 or the upstream rust-lang/rust issue) so future maintainers understand what regression it prevents. The requirements specify this is a regression from PR #122459, and the test should document that context."
  },
  {
    "path": "tests/ui/match/float-pattern-guard-zero.rs",
    "line": 16,
    "body": "**Edge case**: The test validates `f(-1.0, true)` returns `3`, which exercises the wildcard arm. However, it does not test `f(-1.0, false)` or any other non-zero value with `b=false`. AC-5 states 'when called with a non-zero value (e.g., `f(1.0, false)`)', so the test should include `assert_eq!(f(1.0, false), 3);` explicitly to match the example from the requirements. The current `f(1.0, false)` at line 19 is missing, but line 20 tests a different case (`-1.0, true`). This is a minor gap — the wildcard arm is still exercised, but not with the exact example case from AC-5."
  }
]
```
