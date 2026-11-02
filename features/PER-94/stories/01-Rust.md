# Story — Fix float match skipping the `-0.0` arm when a `0.0` guard fails

## Metadata

```yaml
id: PER-94
type: story
title: Fix float match skipping the `-0.0` arm when a `0.0` guard fails
epic: N/A
status: draft
assignee: 
priority: high
severity: major
created: 2025-01-20
updated: 2025-01-20
tracker_id: 
remote: 
hitl: false
```

## Description

Fix a rustc match lowering bug where floating-point pattern matching incorrectly skips the `-0.0` arm when a `0.0` guard fails. The bug causes both `0.0` and `-0.0` to fall through to the wildcard arm instead of matching the `-0.0` pattern.

### Reproduction Case

```rust
#![forbid(unsafe_code)]

#[inline(never)]
fn f(x: f64, b: bool) -> u8 {
    match x {
        0.0 if b => 1,
        -0.0 => 2, // must match whenever x == -0.0, i.e. for x = 0.0 and x = -0.0
        _ => 3,
    }
}

fn main() {
    let r0 = f(0.0, false);
    let r1 = f(-0.0, false);
    println!("f(0.0, false) = {r0}, f(-0.0, false) = {r1}  (expected 2, 2)");
    assert_eq!((r0, r1), (2, 2), "wrong match arm");
}
```

**Expected:** `f(0.0, false) = 2, f(-0.0, false) = 2`  
**Actual:** `f(0.0, false) = 3, f(-0.0, false) = 3` (assertion fails)

Observed on rustc 1.99.0, 1.100.0-beta.5, and 1.101.0-nightly (2026-10-28).

### Root Cause Hypothesis

Regression from PR #122459 ("match lowering: sort `Eq` candidates in the failure case too"), merged 2024-03-31 and released in rustc 1.79. Bisection points at nightly-2024-04-01.

**Before the change:** An `Eq` candidate with a different constant stayed in the candidate list and was still tried after a failed guard.

**After the change:** A failed equality test is treated as failure of that candidate entirely, so the `-0.0` arm is never tried.

IEEE float semantics treat `0.0 == -0.0` as true, but they are **distinct pattern constants**. The current match lowering logic incorrectly eliminates the `-0.0` arm from consideration after the `0.0` guard fails.

## Technical Specification

### Affected Component

**File:** `compiler/rustc_mir_build/src/build/matches/simplify.rs` (or related match lowering module)  
**Function:** Candidate sorting and filtering logic for `Eq` tests with guards

### Implementation Notes

1. **Preserve distinct float constants as separate candidates:** When a guard on a float pattern fails, the match lowering must continue testing other float constant patterns even if they compare equal via IEEE semantics.

2. **Guard failure vs. equality failure:** Distinguish between:
   - Guard failure: candidate's pattern matched but guard returned false → continue to next candidate
   - Equality failure: candidate's pattern did not match → skip candidate

3. **Float constant identity:** `-0.0` and `0.0` are distinct bit patterns (`0x8000000000000000` vs `0x0000000000000000` for f64). Pattern matching must treat them as separate cases even though `0.0 == -0.0` evaluates to true.

4. **Detecting distinct floating-point zeros:** The helper function must verify that the test constant and case constant are **different** zeros (one positive, one negative), not just that both are zeros. Check both `is_zero()` **and** that the bit patterns differ (e.g., `test_leaf != case_leaf` or explicit sign bit comparison). This prevents incorrectly treating `(0.0, 0.0)` or `(-0.0, -0.0)` pairs the same as `(0.0, -0.0)`.

5. **Minimal change scope:** Modify only the candidate filtering/sorting logic for float patterns with guards. Do not change:
   - Float equality operators (`==` for `f32`/`f64`)
   - Unrelated pattern matching code paths
   - The broader float matcher infrastructure

### Regression Test

Add a test case in `tests/ui/match/` covering:
- Float patterns with guards
- Distinct `-0.0` and `0.0` constants
- Guard success and failure paths
- Expected assertion that both `f(0.0, false)` and `f(-0.0, false)` return `2`
- **Documentation:** Include a comment linking to the upstream bug report (https://github.com/rust-lang/rust/issues/165311) and noting this is a regression from PR #122459, so future maintainers understand what regression the test prevents

## Acceptance Criteria

### AC-1: Correct matching after guard failure for 0.0
**Given** a function with match arms: `0.0 if b => 1`, `-0.0 => 2`, `_ => 3`  
**When** called with `f(0.0, false)` (guard fails)  
**Then** the function returns `2` (matches the `-0.0` arm)

**Verification:** Run the reproduction case; assert `f(0.0, false) == 2`

---

### AC-2: Correct matching after guard failure for -0.0
**Given** the same function  
**When** called with `f(-0.0, false)` (guard fails)  
**Then** the function returns `2` (matches the `-0.0` arm)

**Verification:** Run the reproduction case; assert `f(-0.0, false) == 2`

---

### AC-3: Guard success still works for 0.0
**Given** the same function  
**When** called with `f(0.0, true)` (guard succeeds)  
**Then** the function returns `1` (matches the first arm with guard)

**Verification:** Run test; assert `f(0.0, true) == 1`

---

### AC-4: Guard success still works for -0.0
**Given** the same function  
**When** called with `f(-0.0, true)` (guard succeeds)  
**Then** the function returns `1` (matches the first arm with guard)

**Verification:** Run test; assert `f(-0.0, true) == 1`

---

### AC-5: Non-zero values still match wildcard
**Given** the same function  
**When** called with a non-zero value (e.g., `f(1.0, false)`)  
**Then** the function returns `3` (matches the wildcard arm)

**Verification:** Run test; assert `f(1.0, false) == 3` and `f(-1.0, true) == 3`

---

### AC-6: Regression test added
**Given** the fix is implemented  
**When** the rustc compiler test suite runs  
**Then** a new regression test in `tests/ui/match/` exists and passes

**Verification:** Confirm test file exists (e.g., `tests/ui/match/float-pattern-guard-zero.rs`) and passes in CI

## Out of Scope

- Rewriting the float matcher beyond the specific match lowering bug
- Changing IEEE float equality semantics (`==` for `f32` and `f64`)
- Editing unrelated pattern matching code paths
- Opening a pull request against `rust-lang/rust` upstream (PR goes to `pavithrachandran70/rust` only)

## Notes

- **Pull request target:** `pavithrachandran70/rust` (this fork), not `rust-lang/rust`
- **Upstream issue:** https://github.com/rust-lang/rust/issues/165311
- **Keep changes minimal:** Only touch match lowering logic and add the regression test

## References

- Ticket: PER-94
- Requirements: `requirements.md`
- Plan: `plan.md`
- Suspected regression PR: https://github.com/rust-lang/rust/pull/122459

---

## Tests

### Test 1: Guard failure returns to `-0.0` arm for `0.0` input
**Covers:** AC-1  
**Type:** unit  
**File:** `tests/ui/match/float-pattern-guard-zero.rs`

**Description:** Verify that when `f(0.0, false)` is called, the guard on the `0.0 if b` arm fails, and the match continues to test the `-0.0` arm, which succeeds.

```rust
assert_eq!(f(0.0, false), 2);
```

---

### Test 2: Guard failure returns to `-0.0` arm for `-0.0` input
**Covers:** AC-2  
**Type:** unit  
**File:** `tests/ui/match/float-pattern-guard-zero.rs`

**Description:** Verify that when `f(-0.0, false)` is called, the guard on the `0.0 if b` arm fails, and the match continues to test the `-0.0` arm, which succeeds.

```rust
assert_eq!(f(-0.0, false), 2);
```

---

### Test 3: Guard success matches first arm for `0.0` input
**Covers:** AC-3  
**Type:** unit  
**File:** `tests/ui/match/float-pattern-guard-zero.rs`

**Description:** Verify that when `f(0.0, true)` is called, the guard on the `0.0 if b` arm succeeds, and the function returns `1`.

```rust
assert_eq!(f(0.0, true), 1);
```

---

### Test 4: Guard success matches first arm for `-0.0` input
**Covers:** AC-4  
**Type:** unit  
**File:** `tests/ui/match/float-pattern-guard-zero.rs`

**Description:** Verify that when `f(-0.0, true)` is called, the guard on the `0.0 if b` arm succeeds (since `0.0 == -0.0`), and the function returns `1`.

```rust
assert_eq!(f(-0.0, true), 1);
```

---

### Test 5: Non-zero values match wildcard
**Covers:** AC-5  
**Type:** unit  
**File:** `tests/ui/match/float-pattern-guard-zero.rs`

**Description:** Verify that non-zero float values fall through to the wildcard arm and return `3`, regardless of the guard parameter.

```rust
assert_eq!(f(1.0, false), 3);
assert_eq!(f(-1.0, true), 3);
```

---

### Test 6: Regression test file exists and passes
**Covers:** AC-6  
**Type:** integration  
**File:** `tests/ui/match/float-pattern-guard-zero.rs`

**Description:** Confirm the regression test file is created in the rustc compiler test suite at `tests/ui/match/float-pattern-guard-zero.rs`, includes documentation linking to upstream issue https://github.com/rust-lang/rust/issues/165311 and noting the regression from PR #122459, and passes when the test suite is run.

**Verification:** CI runs `./x test tests/ui/match/float-pattern-guard-zero.rs` and reports success. Manual review confirms the test file contains:
- The reproduction function `f(x: f64, b: bool) -> u8`
- All five test assertions (AC-1 through AC-5)
- A comment documenting the upstream issue and regression PR
