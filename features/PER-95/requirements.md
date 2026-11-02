# Requirements — PER-95

**Type:** story  
**Epic:** (none — standalone inline story)  
**Title:** Fix type-checker incorrectly treating `!`-typed zero-length array constants as diverging

---

## Overview

The Rust type-checker incorrectly marks code after a zero-length array constant of type `!` as unreachable, even though such constants are never evaluated per the Reference. This causes safe code to exhibit undefined behavior (out-of-bounds reads at `-O`, `SIGILL` in debug builds).

---

## Requirements

### **REQ-1: Type-checking phase must not propagate divergence from unevaluated constants**

**Given** a zero-length array constant `[C; 0]` where `C` has type `!`  
**When** the type-checker analyzes reachability and divergence  
**Then** the code following the array binding must not be marked unreachable, because `C` is never evaluated

---

### **REQ-2: MIR lowering must preserve reachability for code after zero-length diverging-typed constants**

**Given** a zero-length array constant `[C; 0]` where `C` has type `!`  
**When** MIR is lowered from HIR  
**Then** subsequent statements must remain in the control-flow graph and execute at runtime when their basic block is reached

---

### **REQ-3: Trait-associated constants with diverging return types in zero-length arrays must not affect reachability**

**Given** a trait-associated constant `<fn() -> ! as Tr>::C` used in `[<fn() -> ! as Tr>::R; 0]`  
**When** the type-checker resolves the constant's type to `!`  
**Then** the code after the array binding must not be treated as unreachable

---

### **REQ-4: Direct `!`-typed inline constants in zero-length arrays must not affect reachability**

**Given** a direct inline constant `[const { panic!() }; 0]` of type `[!; 0]`  
**When** the type-checker analyzes the binding  
**Then** the code after the array binding must not be treated as unreachable

---

### **REQ-5: Debug and release builds must exhibit consistent reachability behavior**

**Given** a zero-length array constant `[C; 0]` where `C` has type `!`  
**When** the code is compiled with debug assertions or optimizations enabled  
**Then** subsequent code must execute when its basic block is reached, and must not trap with `SIGILL` or perform out-of-bounds memory accesses

---

### **REQ-6: Bounds checks after zero-length diverging-typed constants must execute correctly**

**Given** a bounds check `if i >= v.len()` followed by a zero-length array constant `[C; 0]` where `C` has type `!`, then an indexing operation `v[i]`  
**When** the bounds check fails (index out of bounds)  
**Then** the code must panic or trap via the bounds check, not via undefined behavior from an elided bounds check

---

## Non-Requirements

- This fix does not address divergence analysis for non-zero-length arrays with `!`-typed constants (those constants *are* evaluated and correctly diverge).
- This fix does not change the evaluation semantics of zero-length arrays (they remain unevaluated per the Reference).
- This fix does not introduce new warnings or lints for zero-length arrays with diverging-typed constants.

---

## Success Criteria

1. The provided example with trait-associated constants compiles and panics on bounds-check failure in both debug and release modes, with no `SIGILL` and no out-of-bounds read.
2. The direct `[!; 0]` example compiles and panics on bounds-check failure in both debug and release modes, with no `SIGILL` and no out-of-bounds read.
3. Regression tests confirm that non-zero-length arrays with `!`-typed constants still correctly diverge.
4. The fix does not introduce performance regressions in type-checking or MIR lowering.

---

## Open Questions

- None.

---

## Notes

- This is a soundness bug affecting stable Rust (1.100+) and nightly.
- The root cause is in the type-checker's divergence propagation; MIR lowering inherits the incorrect reachability marking.
- Edge cases to test: nested zero-length arrays, multiple zero-length arrays in sequence, and zero-length arrays in match arms.
