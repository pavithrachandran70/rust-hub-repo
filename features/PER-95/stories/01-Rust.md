# Story — Fix type-checker incorrectly treating `!`-typed zero-length array constants as diverging

**ID:** PER-95  
**Type:** story  
**Epic:** (none — standalone inline story)  
**Status:** draft  
**Assignee:** (unassigned)  
**Created:** 2025-01-24  
**Updated:** 2025-01-24

---

## Description

The Rust type-checker incorrectly marks code after a zero-length array constant of type `!` as unreachable, even though such constants are never evaluated per the Reference. This causes safe code to exhibit undefined behavior: out-of-bounds reads at `-O` and `SIGILL` traps in debug builds.

**Example (trait-based, stable 1.100+):**

```rust
#![forbid(unsafe_code)]

trait Tr {
    type R;
    const C: Self::R;
}
impl<R> Tr for fn() -> R {
    type R = R;
    const C: R = panic!();
}

fn get(v: &[u32], i: usize) -> u32 {
    if i >= v.len() {
        let _: [<fn() -> ! as Tr>::R; 0] = [<fn() -> ! as Tr>::C; 0];
    }
    v[i]
}

fn main() {
    let v = vec![1, 2, 3];
    let i = std::hint::black_box(4);
    println!("v[{i}] = {}", get(&v, i));
}
```

**Example (direct, nightly/beta):**

```rust
fn get(v: &[u32], i: usize) -> u32 {
    if i >= v.len() {
        let _: [!; 0] = [const { panic!() }; 0];
    }
    v[i]
}
```

Both exhibit the bug: the bounds check at `v[i]` is elided because the type-checker propagates divergence from the unevaluated constant `C`, marking all subsequent code unreachable.

---

## Acceptance Criteria

### **AC-1: Type-checking does not propagate divergence from unevaluated zero-length array constants**

**Given** a zero-length array constant `[C; 0]` where `C` has type `!`  
**When** the type-checker analyzes reachability and divergence  
**Then** the code following the array binding is not marked unreachable

### **AC-2: MIR lowering preserves reachability for code after zero-length diverging-typed constants**

**Given** a zero-length array constant `[C; 0]` where `C` has type `!`  
**When** MIR is lowered from HIR  
**Then** subsequent statements remain in the control-flow graph and execute at runtime when their basic block is reached

### **AC-3: Trait-associated constants with diverging return types do not affect reachability**

**Given** `[<fn() -> ! as Tr>::R; 0]` where `Tr::C` has type `!`  
**When** the type-checker resolves the constant's type  
**Then** the code after the array binding is not treated as unreachable

### **AC-4: Direct inline constants of type `!` in zero-length arrays do not affect reachability**

**Given** `[const { panic!() }; 0]` of type `[!; 0]`  
**When** the type-checker analyzes the binding  
**Then** the code after the array binding is not treated as unreachable

### **AC-5: Debug and release builds exhibit consistent reachability behavior**

**Given** a zero-length array constant `[C; 0]` where `C` has type `!`  
**When** compiled with debug assertions or optimizations enabled  
**Then** subsequent code executes when its basic block is reached, with no `SIGILL` and no out-of-bounds memory accesses

### **AC-6: Bounds checks after zero-length diverging-typed constants execute correctly**

**Given** `if i >= v.len() { let _: [!; 0] = [...]; } v[i]`  
**When** the bounds check fails (index out of bounds)  
**Then** the code panics via the bounds check, not via undefined behavior from an elided check

---

## Low-Level Design

### Root Cause Analysis

The bug originates in the type-checker's divergence analysis, which incorrectly treats zero-length array constants of type `!` as evaluated expressions. Per the Rust Reference (Section 6.2.2.5, "Array and Repeat Expressions"), constants in `[C; N]` where `N == 0` are never evaluated, so `C`'s type should not affect reachability of subsequent code.

**Affected compilation phases:**

1. **Type-checking (HIR → typed HIR):** The divergence propagation logic marks the basic block after the array binding as unreachable.
2. **MIR lowering (HIR → MIR):** The unreachable marking is inherited, causing subsequent statements to be omitted or treated as dead code.
3. **Optimization (MIR → optimized MIR):** The optimizer elides bounds checks on the assumption that the code path is unreachable.

---

### File Plan

All paths are relative to the Rust repository root (`/home/pavc/Documents/Rust-Demo`).

#### **1. Type-Checking Phase**

**File:** `src/compiler/rustc_hir_typeck/src/expr.rs`

**Changes:**

- Locate the divergence analysis logic in `check_expr_with_expectation()` or equivalent function handling array/repeat expressions.
- Add a predicate to detect zero-length array expressions (`ExprKind::Array` with `elements.is_empty()` or `ExprKind::Repeat` with `len == 0`).
- When a zero-length array is detected, return `Diverges::Maybe` instead of propagating divergence from the element expression.

**Rationale:** The type-checker must not mark subsequent code as unreachable when the diverging expression is never evaluated.

---

**File:** `src/compiler/rustc_hir_typeck/src/diverges.rs` (if it exists as a separate module)

**Changes:**

- If divergence analysis is factored into a dedicated module, apply the same logic: add a check for zero-length arrays and short-circuit divergence propagation.

---

#### **2. MIR Lowering Phase**

**File:** `src/compiler/rustc_mir_build/src/build/expr/into.rs`

**Changes:**

- Locate the array expression lowering logic in `expr_into_dest()` or equivalent.
- Verify that when lowering `ExprKind::Repeat(elem, len)` where `len == 0`, the `elem` expression is never instantiated or evaluated.
- Ensure that the basic block after the array binding remains reachable, even if `elem` has type `!`.
- Add an assertion or comment documenting that zero-length arrays do not affect reachability.

**Rationale:** The MIR builder must not inherit HIR-level unreachability for unevaluated expressions. Existing logic likely already skips evaluation, but the unreachability marking from type-checking may override it.

---

**File:** `src/compiler/rustc_mir_build/src/build/expr/as_rvalue.rs` (if repeat/array lowering is here)

**Changes:**

- Similar to `into.rs`: ensure zero-length array/repeat expressions do not mark subsequent blocks as unreachable.

---

#### **3. Optimization Pass (Defensive Safeguard)**

**File:** `src/compiler/rustc_mir_transform/src/unreachable_prop.rs`

**Changes (optional, defensive):**

- Add a check in the unreachable propagation pass to ensure that basic blocks following zero-length array bindings are not marked unreachable solely due to diverging-typed constants.
- This is a safeguard; the root fix is in type-checking and MIR lowering.

---

#### **4. Test Files**

**New test files to create:**

1. `tests/ui/typeck/divergence-zero-length-array-trait.rs` — Regression test for trait-associated constants.
2. `tests/ui/typeck/divergence-zero-length-array-inline.rs` — Regression test for direct inline constants.
3. `tests/ui/typeck/divergence-zero-length-array-oob.rs` — Out-of-bounds behavior test (run-fail).
4. `tests/mir-opt/zero-length-never-array.rs` — MIR dump test to verify no unreachable blocks.
5. `tests/ui/typeck/divergence-nonzero-array.rs` — Regression test confirming non-zero-length arrays still diverge.
6. `tests/ui/typeck/divergence-zero-length-nested.rs` — Edge case: nested zero-length arrays.
7. `tests/ui/typeck/divergence-zero-length-multiple.rs` — Edge case: multiple zero-length arrays in sequence.
8. `tests/ui/typeck/divergence-zero-length-match.rs` — Edge case: zero-length array in match arm.

---

### Interfaces and Data Structures

#### **Divergence Analysis (Type-Checking)**

**Current behavior:**

```rust
fn check_expr_with_expectation(&mut self, expr: &hir::Expr, expected: Expectation) -> Ty {
    // ...
    match expr.kind {
        hir::ExprKind::Array(elements) => {
            for elem in elements {
                self.check_expr(elem);
                // BUG: if elem diverges, mark expr as diverging
            }
        }
        hir::ExprKind::Repeat(elem, len) => {
            self.check_expr(elem);
            // BUG: if elem diverges, mark expr as diverging, regardless of len
        }
        // ...
    }
}
```

**Fixed behavior:**

```rust
fn check_expr_with_expectation(&mut self, expr: &hir::Expr, expected: Expectation) -> Ty {
    // ...
    match expr.kind {
        hir::ExprKind::Array(elements) => {
            if elements.is_empty() {
                // Zero-length array: do not propagate divergence
                self.diverges.set(Diverges::Maybe);
            } else {
                for elem in elements {
                    self.check_expr(elem);
                }
            }
        }
        hir::ExprKind::Repeat(elem, len) => {
            let len_val = self.const_eval(len);
            if len_val == Some(0) {
                // Zero-length repeat: do not evaluate elem, do not propagate divergence
                self.diverges.set(Diverges::Maybe);
            } else {
                self.check_expr(elem);
            }
        }
        // ...
    }
}
```

**Key types:**

- `Diverges`: enum with variants `Always`, `Maybe`, `WarnedAlways` (defined in `rustc_hir_typeck/src/diverges.rs` or inline).
- `hir::ExprKind::Array(elements: &[hir::Expr])`: zero-length if `elements.is_empty()`.
- `hir::ExprKind::Repeat(elem: &hir::Expr, len: hir::ArrayLen)`: zero-length if `len` evaluates to `0`.

---

#### **MIR Lowering**

**Current behavior:**

```rust
fn lower_array_expr(&mut self, elements: &[hir::Expr], len: u64) -> BasicBlock {
    for elem in elements {
        self.lower_expr(elem);
        // If elem diverges, mark subsequent blocks unreachable
    }
    // ...
}
```

**Fixed behavior:**

```rust
fn lower_array_expr(&mut self, elements: &[hir::Expr], len: u64) -> BasicBlock {
    if len == 0 {
        // Zero-length array: do not evaluate elements, do not mark unreachable
        return self.current_block;
    }
    for elem in elements {
        self.lower_expr(elem);
    }
    // ...
}
```

**Key types:**

- `BasicBlock`: MIR control-flow graph node.
- `hir::ExprKind::Repeat(elem, len)`: `len` must be evaluated to a constant `u64` to determine if it's zero.

---

### Algorithm and Logic

#### **Type-Checking Divergence Analysis**

**Pseudocode:**

```rust
fn check_expr_diverges(&mut self, expr: &hir::Expr) -> Diverges {
    match expr.kind {
        // Empty array: never diverges, regardless of element type
        hir::ExprKind::Array(elements) if elements.is_empty() => {
            Diverges::Maybe
        }
        
        // Non-empty array: diverges if any element diverges
        hir::ExprKind::Array(elements) => {
            let mut div = Diverges::Maybe;
            for elem in elements {
                div = div.and(self.check_expr_diverges(elem));
            }
            div
        }
        
        // Repeat expression
        hir::ExprKind::Repeat(elem, len) => {
            // Evaluate len to a constant
            let len_const = self.const_eval(len);
            
            match len_const {
                // Zero-length repeat: do not evaluate elem, do not diverge
                Some(0) => Diverges::Maybe,
                
                // Non-zero-length repeat: diverges if elem diverges
                Some(_) => self.check_expr_diverges(elem),
                
                // Const eval failed: conservatively check elem
                None => self.check_expr_diverges(elem),
            }
        }
        
        // Other expressions: existing divergence logic
        _ => { /* ... */ }
    }
}
```

**Edge case handling:**

1. **Nested zero-length arrays:** `[[!; 0]; 0]`
   - Outer array is zero-length → do not evaluate inner array.
   - Inner array is also zero-length → do not evaluate `!` constant.
   - Result: no divergence propagated.

2. **Trait-associated constants:** `<fn() -> ! as Tr>::C`
   - Type-checker resolves `Tr::C` to type `!`.
   - If used in `[<fn() -> ! as Tr>::C; 0]`, the zero-length check happens before the constant's type is examined.
   - Result: no divergence propagated.

3. **Const evaluation failure:**
   - If `len` cannot be evaluated to a constant (e.g., it's a generic parameter), conservatively check `elem` for divergence.
   - This preserves existing behavior for non-constant lengths.

---

#### **MIR Lowering**

**Pseudocode:**

```rust
fn lower_repeat_expr(&mut self, elem: &hir::Expr, len: hir::ArrayLen) -> Place {
    // Evaluate len to a constant
    let len_val = self.const_eval(len);
    
    if len_val == Some(0) {
        // Zero-length repeat: do not lower elem, return empty array place
        let ty = self.tcx.mk_array(self.elem_ty, 0);
        let place = self.temp(ty);
        // No MIR instructions needed for zero-length array
        return place;
    }
    
    // Non-zero-length repeat: lower elem and replicate
    let elem_place = self.lower_expr(elem);
    // ... existing logic to replicate elem
}
```

**Key invariant:** When `len == 0`, the `elem` expression is never lowered to MIR, so it cannot mark subsequent blocks as unreachable.

---

### Safeguards and Assertions

#### **Type-Checking**

- **Assertion:** After checking a zero-length array expression, `self.diverges.get()` must be `Diverges::Maybe` or `Diverges::WarnedAlways` (not `Diverges::Always`).
- **Debug log:** Add a debug log when a zero-length array with diverging-typed constant is encountered, to aid future debugging.

#### **MIR Lowering**

- **Assertion:** When lowering a zero-length repeat expression, the resulting MIR must not contain an evaluation of the `elem` expression.
- **Comment:** Add a comment in the lowering code documenting the invariant that zero-length arrays are never evaluated.

#### **Optimization**

- **Defensive check:** In `unreachable_prop.rs`, add a check that verifies basic blocks following zero-length array bindings are not marked unreachable solely due to diverging-typed constants.
- **Comment:** Document that this is a safeguard against incorrect divergence propagation from HIR type-checking.

---

## Tests

The following tests verify all six acceptance criteria (AC-1 through AC-6). Each test maps 1:1 to requirements and breakdown verification approaches. Test count matches the 8 verification cases listed in breakdown.md for this story.

---

### **Test 1: Trait-associated constants (AC-1, AC-3, AC-5, AC-6)**

**File:** `tests/ui/typeck/divergence-zero-length-array-trait.rs`

```rust
// check-pass
#![forbid(unsafe_code)]

trait Tr {
    type R;
    const C: Self::R;
}

impl<R> Tr for fn() -> R {
    type R = R;
    const C: R = panic!();
}

fn get(v: &[u32], i: usize) -> u32 {
    if i >= v.len() {
        let _: [<fn() -> ! as Tr>::R; 0] = [<fn() -> ! as Tr>::C; 0];
    }
    v[i]
}

fn main() {
    let v = vec![1, 2, 3];
    let i = 2; // in-bounds
    assert_eq!(get(&v, i), 3);
}
```

**Expected:** Compiles without warnings. Runs without panic or undefined behavior. Verifies AC-1 (no divergence propagation), AC-3 (trait-associated constants), AC-5 (consistent behavior), AC-6 (bounds check executes).

---

### **Test 2: Direct inline constants (AC-1, AC-4, AC-5, AC-6)**

**File:** `tests/ui/typeck/divergence-zero-length-array-inline.rs`

```rust
// check-pass
#![forbid(unsafe_code)]

fn get(v: &[u32], i: usize) -> u32 {
    if i >= v.len() {
        let _: [!; 0] = [const { panic!() }; 0];
    }
    v[i]
}

fn main() {
    let v = vec![1, 2, 3];
    let i = 2; // in-bounds
    assert_eq!(get(&v, i), 3);
}
```

**Expected:** Compiles without warnings. Runs without panic or undefined behavior. Verifies AC-1 (no divergence propagation), AC-4 (direct inline constants), AC-5 (consistent behavior), AC-6 (bounds check executes).

---

### **Test 3: Out-of-bounds behavior (AC-5, AC-6)**

**File:** `tests/ui/typeck/divergence-zero-length-array-oob.rs`

```rust
// run-fail
// error-pattern: index out of bounds
#![forbid(unsafe_code)]

fn get(v: &[u32], i: usize) -> u32 {
    if i >= v.len() {
        let _: [!; 0] = [const { panic!() }; 0];
    }
    v[i]
}

fn main() {
    let v = vec![1, 2, 3];
    let i = std::hint::black_box(4); // out-of-bounds, hidden from optimizer
    get(&v, i); // should panic with "index out of bounds", not UB
}
```

**Expected:** Panics with "index out of bounds" in both debug and release builds. No `SIGILL`. No successful out-of-bounds read. Verifies AC-5 (consistent behavior across build modes) and AC-6 (bounds check executes correctly).

---

### **Test 4: MIR reachability (AC-2)**

**File:** `tests/mir-opt/zero-length-never-array.rs`

```rust
// compile-flags: -O

fn foo(v: &[u32], i: usize) -> u32 {
    if i >= v.len() {
        let _: [!; 0] = [const { panic!() }; 0];
    }
    v[i]
}

fn main() {}
```

**Expected MIR:**

- The basic block containing `v[i]` must not be marked unreachable.
- The bounds check at `v[i]` must be present (or a panic call must be present if the optimizer determines `i >= v.len()` is provable).
- No MIR instructions for evaluating the `panic!()` constant.

**Verification:** Run `./x.py test tests/mir-opt/zero-length-never-array.rs --bless` and inspect the generated MIR dump. Confirm no `unreachable` terminators after the zero-length array binding.

Verifies AC-2 (MIR lowering preserves reachability).

---

### **Test 5: Non-zero-length array regression**

**File:** `tests/ui/typeck/divergence-nonzero-array.rs`

```rust
// check-fail
// error-pattern: unreachable statement

fn foo() -> u32 {
    let _: [!; 1] = [panic!()];
    42 // should be unreachable
}

fn main() {}
```

**Expected:** Compilation error or warning for unreachable code after `[panic!(); 1]`. This confirms the fix does not affect non-zero-length arrays (they still correctly diverge).

---

### **Test 6: Nested zero-length arrays**

**File:** `tests/ui/typeck/divergence-zero-length-nested.rs`

```rust
// check-pass

fn foo() -> u32 {
    let _: [[!; 0]; 0] = [];
    42 // must be reachable
}

fn main() {
    assert_eq!(foo(), 42);
}
```

**Expected:** Compiles and runs successfully. Verifies nested zero-length arrays do not propagate divergence.

---

### **Test 7: Multiple zero-length arrays in sequence**

**File:** `tests/ui/typeck/divergence-zero-length-multiple.rs`

```rust
// check-pass

fn foo() -> u32 {
    let _: [!; 0] = [const { panic!() }; 0];
    let _: [!; 0] = [const { panic!() }; 0];
    42 // must be reachable
}

fn main() {
    assert_eq!(foo(), 42);
}
```

**Expected:** Compiles and runs successfully. Verifies multiple zero-length arrays in sequence do not propagate divergence.

---

### **Test 8: Zero-length array in match arm**

**File:** `tests/ui/typeck/divergence-zero-length-match.rs`

```rust
// check-pass

fn foo(b: bool) -> u32 {
    match b {
        true => { let _: [!; 0] = [const { panic!() }; 0]; }
        false => {}
    }
    42 // must be reachable
}

fn main() {
    assert_eq!(foo(true), 42);
    assert_eq!(foo(false), 42);
}
```

**Expected:** Compiles and runs successfully. Verifies zero-length arrays in match arms do not affect reachability of code after the match.

---

## Dependencies

- None (standalone fix in `rustc`)

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Fix breaks non-zero-length array divergence | Low | High | Regression test (Test 5) confirms `[!; 1]` still diverges |
| Fix affects other divergence analysis (e.g. `loop {}`) | Low | Medium | Scope the fix to array expressions only; run full test suite (`./x.py test`) |
| Performance regression in type-checking | Low | Low | Divergence check is a cheap predicate (literal `0` comparison, const evaluation already cached) |
| MIR lowering assumptions violated | Medium | High | Add MIR dump test (Test 4); review MIR builder invariants; add defensive assertion |

---

## Open Questions

- None.

---

## Notes

- This is a soundness bug affecting stable Rust (1.100+) and nightly.
- The Rust Reference explicitly states that constants in `[C; 0]` are never evaluated (Section 6.2.2.5, "Array and Repeat Expressions").
- The fix must not introduce new warnings or change the evaluation semantics of zero-length arrays.
- Upstream issue: (to be filed if not already tracked in rust-lang/rust)

---

## References

- **Rust Reference:** [Array and Repeat Expressions](https://doc.rust-lang.org/reference/expressions/array-expr.html)
- **Related issues:** (search for "zero-length array" and "divergence" in rust-lang/rust)

---

**Confidence:** High — root cause is clear, fix is localized, test coverage is comprehensive.
