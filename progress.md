# RFC 2229

This document lists the changes that were made to implement the RFC.
### Terms

- **Old compiler**: Pre-RFC 2229 compiler
- **New compiler**: Post-RFC 2229 compiler

### Naming convention within the compiler

- `upvars_mentioned`: These are [root variables] that are referred from a closure.
- `closure_captures`: Precise paths will be captured by the compiler.

PRs to ensure consistent naming convetion with compiler:
- https://github.com/rust-lang/rust/pull/72544
- https://github.com/rust-lang/rust/pull/72591

### Representing capture information

Closure get desugared earlier (HIR) in the compiler passes compared to most of the other code. 

To express information about an expression MIR uses [`mir::Place`] that stores information about
the root variable for the expression, how we derive the value (called projection) from that root variable and 
associated type information.

Eg: If we have the expression `foo.x.y[20]`, we can represent it like
```
-- Base(foo)
   └─-- Field(x)
        └─-- Field(y)
              └─-- Index(20)
```


HIR has a similar construct, also called Place, which could be used to represent capture information.
It was modified to meet our requirements, as follows:

### Decoupling information

The information about the expression was tightly coupled with the expression id. 
Eg:`bar(foo.x, foo.x)`, each `foo.x` has it's own unique HirId.
This adds overhead when keep track of information and in general working with it.

The structure was split into two, 
1. `hir::Place` represents the information about the compiler, called.
2. `hir::PlaceWithHirId` stores the association between `hir::Place` and the expression id as before.

PR for refactor: https://github.com/rust-lang/rust/pull/73489

### Precise Projections

`hir::Place` only supported `Deref` and `Other` as ways of expression how a "place" is memory is being read.
Taking notes from `mir::Place`, we added the following to support our analysis

#### `Field(u32, VariantIdx)`
This stores information about access from a [ADT] (struct, union of field). 

Fields within an ADT can be thought of to be indexed starting at 0. Similarly variants within an
enum are indexed starting at 0. Structures and unions can be thought of having only one variant

The `u32` in the Field projection represents index of the field being used,
and the `VariantIdx` is only for enum types and represents which variant being.

This is equivalent to `Field` and `Downcast` [`mir::ProjectionKind`] (called `PlaceElem`) 
<details>
<summary>Examples (click to expand)</summary>

```rust
#[derive(Copy, Clone, Debug, Eq, PartialEq)]
pub struct Point {
    x: i32,
    y: i32,
}

#[derive(Clone, Debug, Eq, PartialEq)]
pub enum OpKind {
    Sum,
    DoSomething(u32, Point),
    Mult(i32, i32, i32),
    Other,
}

let op = OpKind::DoSomething(10, Point{ x: 5, y: 10} );

// Place for num 
//  - PlaceBase::Local(op), 
//  - projections: [ProjectionKind::Field(Field: 0, Variant: 1)]
// Place for p
//  - PlaceBase::Local(op), 
//  - projections: [ProjectionKind::Field(Field: 1, Variant: 1)]
if let OpKind::DoSomething(num, p) = op {
    
    // Place for p.y
    //  - PlaceBase::Local(p), 
    //  - projections: [ProjectionKind::Field(Field: 1, Variant: 0)]
    // Since p is a structure, it has only one variant and that has index 0. 
    println!("{} {}", p.y, num);
}

let op2 = OpKind::Mult(5, 10, 20);

match op2 {
    // Place for x
    //  - PlaceBase::Local(op2), 
    //  - projections: [ProjectionKind::Field(Field: 0, Variant: 2)]
    // Place for z
    //  - PlaceBase::Local(op2), 
    //  - projections: [ProjectionKind::Field(Field: 2, Variant: 2)]
    OpKind::Mult(x, .., z) => {
        println!("x={}, z={}", x, z);
    }
    _ => ()
}
```

</details>

It is possible that the code we are processing is not valid,
eg: `Opkind::Mult(...) = 5`, here 5 is not an ADT and this will fail.

When we are generating Places in HIR we might not want to stop compilation right away, rather 
make sure that error has been properly reported for this line/column.

For this we use [`delay_span_bug`], instead of panicing instantly it ensures that by the end
of the current compiler pass a bug was reported for the provided [span] (line/col), else the compiler
panics preventing it from going to the next pass.

`rustc_hir` provides pattern utility APIs to make it easy to work with patterns. 
[`EnumerateAndAdjust`] is useful for working with patterns that support the double dots.
Sample use: [EnumerateAndAdjustUsecase]


#### `Index`

Used for present accessing an index in an arrays.

Since arrays are borrowed completely (outside the context of closures), storing the actual index
doesn't provide us any benefit at the moment.

So arr[1] and arr[2] are considered non disjoint. 
https://doc.rust-lang.org/nomicon/borrow-splitting.html

This is equivalent to `Index` [`mir::ProjectionKind`]


#### `Subsclice`

Same as `Index`, but used for subslices.
This is equivalent to `Subsclice` [`mir::ProjectionKind`]



PR for adding precise projections: https://github.com/rust-lang/rust/pull/74140


### Using Places as Closure Captures

- [Zulip Stream for using Places]

#### What didn't work
<details>
<summary> Talk about why the vec design didn't work </summary>
</details>

#### Capture Analysis

// TODO: insert link
To understand how the old compiler did capture analysis, checkout: [closure_captures.md]

- Talk about using bridge
- Ignoring projections
- Using place.ty() directly and how it works in both (ignore and not ignore) cases 
- Intended use at this step:
    - Keep closure_captures it allows us to answer questions like if a root variable is part of a captured 
    
    ath quickly


### Capture Rules

- Precise paths only matter for Structs and Tuples.
- For arrays non-overlapping parts are not considered disjoint. (even outside the context of closures).


### ClosureSubsts

This helps represent the closure in a desugared form.
It consists of:
- The closure type: Fn/FnMut/FnOnce. This is done using type variables so, eg: Fn is assigned u8 etc.
- The closure signature
- A tuple containing the list of types of captured variables.

Before capture analysis the old compiler could create a tuple of inference variables of a known arity, when the ClosureSubsts are created.
This is because we knew the root variables that were going to be captured and this was same as the number of paths captured.
These inference variables were then replaced with their types after capture analysis.

However in the new compiler, number of captures will be atleast the number of root variables captured but we won't know the 
exact number until we do the capture analysis. To deal with the case, instead of create a tuple of inference variables in ClosureSubsts,
we create a single inference variable that then gets replaces with a tuple of closure capture types.

We need to make sure that we handle this change properly, the type is no more `ty::Tuple` until the analysis happens.
We rely on `infcx.shallow_resolve(substs.tupled_upvars)` to check if inference variable has been resolved, i.e. the type isn't `ty::Infer`.
Depending on the how this is used we will deal with at the site.
- trait_selection:
- pretty.rs: We don't have infcx context so we won't be printing the type information of the captures.
- typeck/coercion: We just need to know if closure captures any variables or not, so that we can decide if we want to coerce it to a FnPtr. We can just check upvars_mentioned to know if there are captured root variables. 

[root variables]: https://github.com/rust-lang/project-rfc-2229/blob/master/closure_captures.md#root-variable
[`mir::Place`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/struct.Place.html
[ADT]: https://rustc-dev-guide.rust-lang.org/ty.html?highlight=adt#adts-representation
[`delay_span_bug`]: https://rustc-dev-guide.rust-lang.org/ty.html?highlight=delay_span_bug#type-errors
[span]: https://rustc-dev-guide.rust-lang.org/diagnostics.html?highlight=span#span
[`mir::ProjectionKind`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/type.PlaceElem.html
[`EnumerateAndAdjust`]: https://doc.rust-lang.org/nightly/nightly-rustc/src/rustc_hir/pat_util.rs.html#33
[EnumerateAndAdjustUsecase]: https://github.com/rust-lang/rust/blob/master/src/librustc_typeck/check/pat.rs#L1006
[Zulip Stream for using Places]: https://rust-lang.zulipchat.com/#narrow/stream/189812-t-compiler.2Fwg-rfc-2229/topic/Typecheck.20tables.20using.20Places

