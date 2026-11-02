```json
[
  {
    "path": "compiler/rustc_mir_build/src/builder/matches/buckets.rs",
    "line": 407,
    "body": "**AC violation: Helper function does not check that zeros have different signs**\n\nThe `float_zeros_share_eq_test` function checks `test_leaf.to_f64().is_zero() && case_leaf.to_f64().is_zero()` but does not verify the zeros have **different** signs. This means `(0.0, 0.0)` and `(-0.0, -0.0)` incorrectly return `true`, treating identical zeros as if they were the opposite-sign pair.\n\n**AC-1/AC-2 gap:** When `test_val == case_val` (both are the same zero), the first branch at line 349 already returns `TestBranch::Success` with `fully_matched = true`, so the helper is never called for identical zeros. However, the helper's logic is still **semantically wrong** per the story's Implementation Notes #4: it must verify \"that the test constant and case constant are **different** zeros (one positive, one negative), not just that both are zeros.\"\n\n**Story requirement:** \"Check both `is_zero()` **and** that the bit patterns differ (e.g., `test_leaf != case_leaf` or explicit sign bit comparison). This prevents incorrectly treating `(0.0, 0.0)` or `(-0.0, -0.0)` pairs the same as `(0.0, -0.0)`.\"\n\n**Fix:** Add `&& test_leaf != case_leaf` to each match arm:\n```rust\nty::FloatTy::F16 => {\n    test_leaf.to_f16().is_zero() && case_leaf.to_f16().is_zero() && test_leaf != case_leaf\n}\n```\n(Repeat for F32, F64, F128.)\n\nWithout this check, the helper would incorrectly claim two identical zeros \"share an eq test\" when in reality they are the exact same candidate and should not be treated as distinct."
  },
  {
    "path": "tests/ui/match/float-pattern-guard-zero.rs",
    "line": 1,
    "body": "**AC-6 gap: Missing documentation comment content**\n\nThe story's Technical Specification § Regression Test mandates: \"**Documentation:** Include a comment linking to the upstream bug report (https://github.com/rust-lang/rust/issues/165311) and noting this is a regressi[on test]\". The file includes the bug link (line 1) and PR link (line 2) but the sentence cuts off mid-word: `noting this is a regressi`. The word \"regression\" is incomplete.\n\n**Fix:** Complete the sentence in the module doc comment:\n```rust\n//! Regression test for https://github.com/rust-lang/rust/issues/165311.\n//! Regression from https://github.com/rust-lang/rust/pull/122459.\n//! Ensures that -0.0 and 0.0 remain distinct match candidates after a guard fails.\n```"
  }
]
```
