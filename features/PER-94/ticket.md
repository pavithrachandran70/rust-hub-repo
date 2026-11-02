---
key: PER-94
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: "Fix float match skipping the\_ `-0.0` \_arm when a\_ `0.0` \_guard fails"
---

Description

```
Bug in rustc match lowering for floating-point patterns. Reported upstream as https://github.com/rust-lang/rust/issues/165311. Fix it on this fork (https://github.com/pavithrachandran70/rust) and open the pull request here.

## Reproduction

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

Seen on rustc 1.99.0, 1.100.0-beta.5, and 1.101.0-nightly (2026-10-28). Expected result is `2, 2`. Actual result is `3, 3`, and the assertion fails. Both `0.0` and `-0.0` fall through to the wildcard after the `0.0` guard fails.

## Cause hypothesis

Likely regression from [#122459](<https://github.com/rust-lang/rust/pull/122459>) ("match lowering: sort `Eq` candidates in the failure case too"), merged 2024-03-31 and released in 1.79. Bisection points at nightly-2024-04-01. Before that change, an `Eq` candidate with a different constant stayed in the list and was still tried after a failed guard. After it, a failed equality test is treated as failure of that candidate, so the `-0.0` arm is never tried. `0.0` and `-0.0` compare equal in IEEE floats, but they are distinct pattern constants.

## Acceptance criteria

* `f(0.0, false)` and `f(-0.0, false)` both return `2`.
* `f(0.0, true)` and `f(-0.0, true)` still return `1`.
* A non-zero value still returns `3`.
* Add a regression test in the compiler test suite for this match.
* Keep the change limited to match lowering and that test.

## Out of scope

* Rewriting the float matcher, changing `==` for `f32` and `f64`, or editing unrelated pattern code.
* Opening the pull request against `rust-lang/rust`. The pull request goes to `pavithrachandran70/rust`.
