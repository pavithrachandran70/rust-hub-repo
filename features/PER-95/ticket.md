---
key: PER-95
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: "A\_ `!` -typed constant in a zero-length array\_ `[C; 0]` \_is never evaluated,\
  \ but type checking treats it as diverging, so safe code executes\_ `unreachable`"
---

Description: In `[C; 0]`, a constant `C` of type `!` is never evaluated, as the Reference specifies. Type checking still treats `C` as diverging, so the code after it is compiled as unreachable. That code does run. The safe snippet below has no feature gates and reads past the end of a `Vec` at `-O`. Debug builds trap with `SIGILL`.

```
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
    let i = std::hint::black_box(4); // 4, hidden from the optimizer
    println!("v[{i}] = {}", get(&v, i));
}
```

On beta and nightly (stable `!` since 1.100), the same bug is shorter:

```
fn get(v: &[u32], i: usize) -> u32 {
    if i >= v.len() {
        let _: [!; 0] = [const { panic!() }; 0];
    }
    v[i]
}
```
