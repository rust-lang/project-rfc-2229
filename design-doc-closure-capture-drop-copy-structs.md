# Closure capture: drop and copy structs

* Rust PR: https://github.com/rust-lang/rust/pull/88477
* Hackmd draft: https://hackmd.io/AC5EJng9SDStdsjG2M9cUQ?view

Testing revealed an interesting example.
 
When you have the following situation:
 
* A struct `Wrap` that implements `Drop`
* and a `move` closure that accesses some non-Copy field of that struct by reference (but doesn't access the struct as a whole)

The current rules generate a capture rule that is a guaranteed compilation error. Consider:

```rust
struct Wrap<T>(T);

impl<T> Drop for Wrap<T> {
 fn drop(&mut self) { }
}

fn main() {
 let w = Wrap(format!("Hello, world!"));
 let c = move || drop(&w.0);
}
```

Here, the closure accesses `w.0` by reference. Because of the `move` clause, this becomes a "by-value" capture, and is hence desugared to something like:

```rust
let c = TheClosureType { cap0: w.0 };
```

This in turn generates a compilation error, because it is illegal to move a single field from a struct that implements `Drop`. 

## Fixing the error 

The only fix under the current rules is to add a "dummy use" of `w`. The migration would suggest `let _ = &w` as the dummy use, but a clearer choice might be `drop(w)`:

```rust
struct Wrap<T>(T);

impl<T> Drop for Wrap<T> {
    fn drop(&mut self) { }
}

fn main() {
    let w = Wrap(format!("Hello, world!"));
    let c = move || {
        drop(&w.0);
        drop(w); // dummy-use that forces us to capture `w` as a whole
    };
}
```


In the future, nikomatsakis intends to re-open the discussion around explicit capture closures; in such cases, one could write something like `move(w) || drop(&w.0)`.

## Related scenarios

The scenario described above is surprisingly narrow. Here are a number of related scenarios that do not trigger the error.

### non-move closure

```rust
fn main() {
    let w = Wrap(format!("Hello, world!"));
    let c = || drop(&w.0);
}
```

A non-move closure is not an error because `w.0` is captured by reference.

### field that is moved in the closure

```rust
fn main() {
    let w = Wrap(format!("Hello, world!"));
    let c = move || drop(w.0);
}
```

A field that is moved within the closure wouldn't have compiled at all. Even if we capture `w`, the closure itself will still get an error when it attempts to move from `w.0`.

### field of `Copy` type

```rust
fn main() {
    let w = Wrap(22);
    let c = move || drop(w.0);
}
```

Fields of copy type do not present an error because capturing `w.0` is legal. Ironically (perhaps) we may still suggest a migration because Rust 2018 would have captured `w` and thus the destructor would have run at a different time.

## The problem: Rust 2021 causes *less code* to compile in this case

Consider the experience of a new user approaching Rust 2021. This sort of example now requires an explicit `drop(w)`, whereas the older capture system (Rust 2018) did not. This seems unfortunate. In virtually every other case, the new Rust 2021 system simply allows more code to work (ignoring the question of changing drop order). This is the one known exception where the rules cause *less code* to compile.

## Alternatives

There are a few alternatives to consider. For each alternative, we will consider various scenarios:

* Base: The "base example" 
* Copy: The "field of copy type" example
* AddedDrop: What happens to the base example when the `Drop` impl is at first absent, but later added
* AddedCopy: What happens to an example for `Wrap<T>` when `T: Copy` is added

We ignore non-move closures and fields that are moved in the closure because they operate the same in all proposals.

### Summary table

| Proposal | Base | Copy | AddedDrop | AddedCopy |
| --- | --- | --- | --- | --- |
| Current rules | Requires `drop(w)` | Minimal capture of `w.0` | Base example compiles without drop, does not compile afterwards | No change; to have compiled before, `drop(x)` was needed
| Truncate at Drop | Captures `w` | Captures `w`, which is not minimal | Base example changes from capturing `w.0` to capturing `w` | No change; still captures `w`
| Truncate at Drop unless Copy | Captures `w` | Minimal capture of `w.0` | Base example changes from capturing `w.0` to capturing `w` | Base example changes from capturing `w` to capturing `w.0` 

Observe: We actually have to make some amount of choice here. **If we stick with the current rules, we can only adopt "truncate at drop, consider copy" in the future** and retain compatibility.

### Current rules

The current rules do not consider whether a type implements `Drop` when deciding what gets captured. As a result:

**Base:** We capture `w.0` by value in the base example, yielding a compilation eror for an illegal move.

**Copy:** We capture `w.0` by value in the "field of copy type" example, which is legal, and `w` is dropped outside of the closure.

**AddedDrop:** Adding `Drop` makes no difference to the 

**AddedCopy:** the base example would not have compiled, so adding `T: Copy` would make it compile. Alternatively, if the user added `drop(w)` in the base code, then we would always capture `w`.

### Truncate at drop

We could modify the rules so that when a place is captured by-value, we truncate at any structs that implement `Drop` whose field is captured. This would cause us to capture `w` instead of `w.0` in all examples where `Drop` is implemented:

**Base:** We capture `w`, so the code compiles with no additional annotation.

**Copy:** We capture `w`, so the code compiles, but it doesn't have a "minimal" capture (which would be `w.0`). There are other scenarios where our captures are not minimal, of course, but this could result in still needing a workaround like `let v = w.0` and using `v`.

**AddedDrop:** Adding `Drop` changes the path that is captured. Consider this example:

```rust
struct Wrap<T>(T);

fn main() {
    let w = Wrap(format!("Hello, world!"));
    let c = move || {
        // Captures w.0 by value
        drop(&w.0);
    };
}
```

when `Drop` is added, the closure would now capture `w` by value instead. This is not a change in when the `Drop` code runs, however, because there was no drop code to run before. 

It *could* be a change if there are other fields of `Wrap`. In this example, `w.1` is dropped when the block is exited:

```rust
struct Wrap<T>(T, String);

fn main() {
    let w = Wrap(format!("Hello,"), format!("World!"));
    let c = move || {
        // Captures w.0 by value
        drop(&w.0);
    };
    
    // w.0 is dropped when `c` is dropped
    // w.1 is dropped when block is exited
}
```

But if a `Drop` impl is added to `Wrap`, `w.1` will be dropped when the closure is dropped instead.

**AddedCopy:** Adding `Copy` makes no difference in these rules. The examples will capture the same paths either way.

### Truncate at drop, consider Copy

We could only truncate at `Drop` if the place being captured is not of `Copy` type.

This would result in:

**Base:** Captures `w` (because `w.0` is of type `String`).

**Copy:** Captures `w.0` (because `w.0` is of copy type).

**AddedDrop:** Precisely as the previous case (can change what is captured).

**AddedCopy:** Adding `Copy` would change what is captured to be more minimal.

## Future compatibility observations


If we adopted some variant of the [fields in traits RFC](https://github.com/nikomatsakis/fields-in-traits-rfc/), which disallowed moves from said fields, we would presumably want to take the same basic approach as we take with structs that implement `Drop`.

If we adopted some way to move out of structs that implement `Drop`, perhaps explicitly "forgetting" their destructor, then this would want to be considered in the closure capturing rules too.

## Axes for consideration

### Runtime performance (closure size, primarily)

Measurements so far have not shown significant effect on closure size and this issue is not expected to have much impact.

### Need for surprising workarounds

The goal for Disjoint Capture in Closures was to eliminate many of the surprising workarounds that are needed in Rust 2018 to get code to compile. Because Rust 2018 programs capture entire variables, it is frequently necessary to introduce "dummy variables" and capture those dummary variables:

```rust
let bar = &foo.bar;
|| ... bar ...
```

In Rust 2021, this is no longer necessary: the closure could simply access `&foo.bar` and (generally, anyway) the closure will capture precisely the path which was used. If we retain the current semantics, this case proves to be an exception. The user is required to add a workaround of adding some kind of dummy statement (e.g., `let w = w` or `drop(w)`) to the closure which serves no purpose but to "hint" to the closure how much to capture. These statements have proven to be non-obvious for users who don't deeply understand the closure capture semantics.

On the other hand, some of the proposed fixes carry surprises of their own! For example, if we truncate all moves at structs that implement `Drop`, then the "fields of copy type" case becomes an error, and requires a workaround similar to those required in Rust 2018 (first reading the field into a dummy variable), at least if you want a precise capture. This may not be a big deal, because capturing the entire variable is frequently "good enough". This is unclear.

**Also:** Rust 2021 is not always able to capture precise paths. The most frequent cause of truncation is indeed around move closures which (e.g.) truncate at `Box<T>`. Adding another case where moves truncate may not be terribly surprising to users, as it sort of fits the pattern of "sometimes this closure moves more than I expected and I need to help it". Having an error of the kind "the closure moves less than I need it to for things to compile" would be surprising.

### Overall rule complexity and "by hand" monomorphization idempotence

Some of the alternatives have quite complex rules, notably the one which truncates a `Drop` path only for non-copy paths. This also has the curious effect that monomorphizing a function "by hand" can change what values are captured by a closure (since knowing the precise types may mean that we can conclude that a type is `Copy`, which was not visible in the generic version of the function). Note that if we adopt "fields in traits", we have to start approximating whether a type implements `Drop`, in which case monomorphization could have the same effect.

Historically, we have found that most users don't even try to understand the full rules of the system. They develop an intuitive model for what is expected. This intuitive model is often smarter than the compiler itself, and hence they appreciate complex rules that more closely match that intuition more than they appreciate simpler rules.

### Code continues to compile across editions (potentially with runtime changes)

This is the only case where -- assuming no migrations are performed -- Rust 2021 rules would cause a compilation error that did not occur in Rust 2018 (there are cases where the runtime dynamic semantics would change without migrations, since destructors could run in a different place). If we adopted rules such as "drop truncation", then the code would continue to compile in Rust 2021, and we would eliminate this case.

### Rust 2021 rule complexity and non-linear behavior

The more complex rules have "non-linear" behavior when new impls, e.g. of Copy or Drop, are added. That is to say that closure captures change as a result of whether things have Drop or Copy. There is already plenty of precedent for these traits having intimate effects on borrow check behavior:

* Adding a Drop impl causes the borrow check to consider all references to be potentially used until the drop runs.
* Adding a Copy impl can cause non-move closures to capture "by ref" instead of "by value", which can trigger lifetime errors.
    * This could also have effected when destructors run, even in Rust 2018! (A value may or may not be moved into a closure.)

### Ultimately capture clauses will be the best way to get clarity about *exactly* what a closure captures

Closure captures are always going to be subject to rather complex rules. People rarely care when their destrutors run and, when they do, those destructors are almost always present in mutex guards or other variables whose lifetimes are rather carefully managed. For people who wish to be very precise, a feature like capture clauses is probably the best solution: there is an inherent tradeoff between "being smart" and having closures capture only what is needed and clarity around exactly what is captured. Even the old rule of just capturing variables (but inferring the mode) could be surprising and hard for people to reason about, and we are moving further in this direction (which is good). As long as the rules can be documented, we should aim to make them do the right thing most of the time.

## Niko's conclusion

We should adopt the "Truncate at Drop unless Copy" rule:

* Eliminating "workarounds" and dummy assignments is precisely the goal of this work. The rules are already complex enough that the minimal "simplification" of not considering Drop is not going to make them *simple*, and may indeed make them feel *more complex*, since it will cause captures that capture "too precise" to compile, instead of the current behavior, which errs on the side "capture truncated paths" when in doubt.
* Even in Rust 2018, adding `Drop` and `Copy` impls can create compilation errors and alter execution order around closures (e.g., it affects whether a non-move closure can capture by ref or not). This has not surfaced as a significant source of concern.
* In those cases where people care about precisely when Drop runs, they can easily document that with a `drop(w)` call.
    * In the future, I would favor capture clauses as well as the ultimate solution to clarity in cases where extreme clarity is desired.

