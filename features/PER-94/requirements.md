# Requirements — PER-94

## Overview

Fix a rustc match lowering bug where floating-point pattern matching incorrectly skips the `-0.0` arm when a `0.0` guard fails. The bug causes both `0.0` and `-0.0` to fall through to the wildcard arm instead of matching the `-0.0` pattern. This is a regression likely introduced in PR #122459 (merged 2024-03-31, released in rustc 1.79), which changed how `Eq` candidates are sorted in the failure case. The fix preserves distinct float constants (`-0.0` vs `0.0`) as separate match candidates after guard failures.

## Technical Context

**Suspected regression:** PR #122459 ("match lowering: sort `Eq` candidates in the failure case too") introduced a change where a failed equality test is treated as failure of that candidate entirely. Before this change, an `Eq` candidate with a different constant remained in the candidate list and was still tried after a failed guard. Since IEEE floats treat `0.0 == -0.0` as true, but they are distinct pattern constants, the current logic incorrectly eliminates the `-0.0` arm from consideration.

**Location:** Match lowering logic in the rustc compiler, specifically the handling of floating-point pattern constants with guards.

**Constraint:** Keep changes limited to match lowering and the regression test. Do not modify unrelated pattern code, float comparison operators, or the broader float matcher.

## Acceptance Criteria

### AC-1: Correct matching after guard failure for 0.0
**Given** a function with match arms: `0.0 if b => 1`, `-0.0 => 2`, `_ => 3`  
**When** called with `f(0.0, false)` (guard fails)  
**Then** the function returns `2` (matches the `-0.0` arm)

### AC-2: Correct matching after guard failure for -0.0
**Given** the same function  
**When** called with `f(-0.0, false)` (guard fails)  
**Then** the function returns `2` (matches the `-0.0` arm)

### AC-3: Guard success still works for 0.0
**Given** the same function  
**When** called with `f(0.0, true)` (guard succeeds)  
**Then** the function returns `1` (matches the first arm with guard)

### AC-4: Guard success still works for -0.0
**Given** the same function  
**When** called with `f(-0.0, true)` (guard succeeds)  
**Then** the function returns `1` (matches the first arm with guard)

### AC-5: Non-zero values still match wildcard
**Given** the same function  
**When** called with a non-zero value (e.g., `f(1.0, false)`)  
**Then** the function returns `3` (matches the wildcard arm)

### AC-6: Regression test added
**Given** the fix is implemented  
**When** the rustc compiler test suite runs  
**Then** a new regression test covering float pattern matching with guards exists and passes

## Success Signal

- All six acceptance criteria pass
- The reproduction case from the ticket (`f(0.0, false)` and `f(-0.0, false)` both return `2`) works correctly
- The regression test is committed alongside the fix
- No changes to unrelated pattern matching code or float comparison operators

## Out of Scope

- Rewriting the float matcher beyond the specific match lowering bug
- Changing IEEE float equality semantics (`==` for `f32` and `f64`)
- Editing unrelated pattern matching code paths
- Opening the pull request against `rust-lang/rust` upstream (PR goes to `pavithrachandran70/rust` only)

## Open Questions

None — reproduction case, suspected cause, and acceptance criteria are clearly defined in the ticket.
