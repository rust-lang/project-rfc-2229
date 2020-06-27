# Refactor `closure_captures`


## Goal
Be able to express memory captured by a closure more precisely.

For example

- `|| p.x`, here we want to express the field `x` is being captured instead of all of `p`.
- `|| tuple.0`, here we want to express that the 0th entry in `tuple` in being captured instead of the complete tuple.

## Background

Consider the following code:

```rust
struct Point {
    x: i32,
    y: i32,
    name: String,
}

// hir_id_1 associated with definition of p.
let p = Point { x: 10, y: 20, name: format!("p") };
// hir_id_2 associated with definition of mirror_p.
let mut mirror_p = Point { x: -10, y: 20, name: format!("mp") };

// hir_id_3 associated with definition of m_x.
let m_x = &mirror_p.x;

// hir_id_4 associated with the expression p.x
let c1 = || println!("{}", p.x);
// hir_id_5 associated with the expression m_x
let c2 = || println!("{}", m_x);

let c3 = || {
    // hir_id_12 associated with the definition of p3.
    let p3 = Point{ x: 1, y: 2, name: format!("p3") };
    // hir_id_6 associated with the expression p3.x
    println!("{}", p3.x);
}

let c4 = || {
    // hir_id_8 associated with the expression mirror_p.x
    c_nest = || { mirror_p.x -= 20 };
    // hir_id_7 associated with the expression p.x
    drop(p.name);
}

// hir_id_11 associated with definition of p.
let p2 = Point { x: 10, y: 20, name: format!("p") };
let mut c5 = || {
    // hir_id_9 associated with the expression mirror_p.x
    p2.x += 20;
}

let c6 = move || {
    // hir_id_10 associated with the expression mirror_p.x
    p2.x += 20;
}
```

### Root variable

In the above code `p` is the root variable to `p.x` and `p.name`, `m_x` is a root variable on its own and `p3` is the root variable to `p3.x`.

### Current state

#### [`upvars_mentioned`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/arena/struct.Arena.html#structfield.upvars_mentioned)
Provides list of root variables that are completely or partially captured by the closure (or any other nested clousre), and the span pointing to their use within the capture.

More precisely maps root variables outside of the closure (that are mentioned) within the closure to access span within the closure.

```
upvars_mentioned[c1] =  { hir_id_1 : Span(hir_id_4) }
upvars_mentioned[c2] =  { hir_id_3 : Span(hir_id_5) }
upvars_mentioned[c3] =  {}
upvars_mentioned[c_nest] =  { hir_id_2: Span(hird_id_8) }
upvars_mentioned[c4] =  { hir_id_2: Span(hird_id_8), hir_id_1: Span(hir_id_7) }
upvars_mentioned[c5] =  { hir_id_11 : Span(hir_id_9) }
upvars_mentioned[c6] =  { hir_id_11 : Span(hir_id_10) }
```

#### [`TypeckTables`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TypeckTables.html)

Stores type and various related infromation about a given scope.

#### [`TypeckTables::closure_captures`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TypeckTables.html#structfield.closure_captures)

Provides list of root variables that are completely or partially captured by the closure, and maps them to an `UpvarId`.
`UpvarId` contains the closure definition id and the hir id of the root variable.

```
closure_captures[c1] =  { hir_id_1 : UpvarId(def_id_c1, hir_id_1) }
closure_captures[c2] =  { hir_id_3 : UpvarId(def_id_c2, hir_id_3) }
closure_captures[c3] =  {}
closure_captures[c_nest] =  { hir_id_2: UpvarId(def_id_c_nest, hir_id_2) }
closure_captures[c4] =  { hir_id_2: UpvarId(def_id_c_4, hir_id_2), hir_id_1: UpvarId(def_id_c_4, hir_id_1) }
closure_captures[c5] =  { hir_id_11 : UpvarId(def_id_c5, hir_id_11) }
closure_captures[c6] =  { hir_id_11 : UpvarId(def_id_c6, hir_id_11) }
```

#### [`TypeckTables::upvar_capture_map`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TypeckTables.html#structfield.upvar_capture_map)

For given scope, maps all the `UpvarId` to capture kind, i.e. by value, or mutabale/immutable reference.

Assume that `c1 ... c6` are within the same scope, then for that scope:
```
upvar_capture_map = {
    UpvarId(def_id_c1, hir_id_1) : ByRef(ImmBorrow),
    UpvarId(def_id_c2, hir_id_3) : ByRef(ImmBorrow),
    UpvarId(def_id_c_nest, hir_id_2) : ByRef(ImmBorrow),
    UpvarId(def_id_c_4, hir_id_2) : ByRef(MutBorrow),
    UpvarId(def_id_c_4, hir_id_1) : ByValue,
    UpvarId(def_id_c5, hir_id_11) : ByRef(MutBorrow),
    UpvarId(def_id_c6, hir_id_11) : ByValue,
}
```

#### [`Place`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_typeck/mem_categorization/struct.Place.html)

Represents how memory is accessed.

Eg: `p3.x`, we can express that the base root variable is `p3` and then we get the value by accessing Field(x).

If the [`base`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_typeck/mem_categorization/struct.Place.html#structfield.base)
field is `PlaceBase::Upvar` we know that the access is based on a captured variable.

### Generating this information

To generate `upvars_mentioned`, `closure_captures`, `upvar_capture_map` we need to iterate the CFG, to see how variables are being captures and used.

[`rustc_hir::instravisit`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/intravisit/index.html)
provides a visitor/walker that can traverse the tree in execution order.
We can use it and override `visit_*` methods, to have a walker that meets our needs.


#### [`upvars_mentioned`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/arena/struct.Arena.html#structfield.upvars_mentioned)
Generated in [`librustc_passes`](https://doc.rust-lang.org/nightly/nightly-rustc/src/rustc_passes/upvars.rs.html#82).

Here we implement the instravisit visitor that can generate a list of local root variables by looking at `PatKind::Binding` (**TODO: explain**).

Once that is done, we walk the body again to look for
- [`Path`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/hir/struct.Path.html) that represents a variable use (`Res::Local`).
    Path isn't something like `a.b` but just represents some variable use.
    If the variable isn't a local variable we add it to the list of upvars and map it to the Path span.
- `Expr` that represents a Closure, i.e. handle the case of nested closures. We can query `upvars_mentioned` for the nested closure and include all the upvars returned, as part of captures for the current (enclosing) closure.


#### [`TypeckTables::closure_captures`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TypeckTables.html#structfield.closure_captures)

Generated in [`librustc_typeck`](https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/check/upvar.rs#L115).

This map for a given closure is very similar to upvars_mentioned. Instead of storing span we store the pair of `(closure_def_id, root_var_hir_id)`.

We iterate over `upvars_mentioned` for a given `closure_def_id` to get captured root variable HirId and generate an UpvarId for them and build the map.

#### [`TypeckTables::upvar_capture_map`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TypeckTables.html#structfield.upvar_capture_map)

Generated in [`librustc_typeck`](https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/check/upvar.rs#L115).

We implement another `intravist::Vistor` to analyse closures within a scope.
When we see a closure within the body of the scope, we walk the body of the closure, when we are done walking the closure's body we start the analysis on the closure.
(We need to walk first, to handle nested closures.)


To begin the analysis we use hints like `mut` for c5, `move` in c6, to guess the initial borrow kind of each capture.
Once the guesses are set, we create an `ExprUseVistior` that visits each expression within the closure body.
ExprUseVisitor calls into MemCategorization which returns the access information about the expression as a `Place`.

`ExprUseVisitor`(https://github.com/rust-lang/rust/blob/master/src/librustc_typeck/expr_use_visitor.rs) invokes methods on the delegate when it finds a Place being consumed or borrowed. The delegate has the job of deciding what to do with that information. In this case, the delegate is the upvar analysis, which "upgrades" the kind of borrow needed depending on the accesses it sees (all reads == shared borrow, writes == mutable borrow, moves == take ownership).
In our case the `Delegate` trait is implemented
by [`InferBorrowKind`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_typeck/check/upvar/struct.InferBorrowKind.html).

It's possible that while processing the expressions in ExprUseVisitor on the closure body we see another closure (i.e. nested within the enclosing closure).
Since we walked the body of the closure before we started the analysis, we have already proccessed all nested closures.
We will iterate over the captured variables of this nested closure and update borrow kind for all of them for the enclosing closure.
Last bit done here: [`expr_use_visitor::walk_captures`](https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/expr_use_visitor.rs#L309)

```rust
fn foo() {
    let a;

    let c1 = || {
        let c2 = || {
           drop(a);
        };
    };
}
```

The flow here would look like:
 1. Intravisit on foo, see a closure expr c1
 2. Intravisit on c1, sees a closure expr c2
 4. Intravisit the body c2. Done visiting body of c2.
 5. Analyses the closure c2 using ExprUseVisitor and generate `upvar_capture_map`.
 6. c2 intravisit finishes. Done visiting the body of c1.
 7. Analyses the closure c2 using ExprUseVisitor and generate `upvar_capture_map`.
 8. Encounter the closure expression for c2. Adjust capture infromation for `a` based on `c2`
 8. c1 intravisit finishes
 9. foo intravisit finishes

 More nicely (here `capture_kinds` refer to `upvar_capture_map` for a particular closure):
```
| visit body of foo
| | visit body of c1
| | | visit closure c2
| | | | visit body of c2 (does nothing)
| | | | analyze body of c2, producing capture_kinds[c2]
| | analyze body of c1, reading capture_kinds[c2], producing capture_kinds[c1]
```

#### [`Place`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_typeck/mem_categorization/struct.Place.html)

Generated in [`librustc_typeck`](https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/mem_categorization.rs).

Several methods exist that can analyse different kinds of expression and pattern to express how memory is being accessed.

### Use cases

Legend:

- `U`: `upvars_mentioned`
- `C`: `closure_captures`
- `B`: `upvar_capture_map`
- `U->C`: Currently uses `upvars_mentioned`, but needs to start using `closure_captures`. This is most likely because we have done type checking either entierly or at least for the related scope.
- `var_hir_id`: `HirId` of root variable of a capture
- `CaptureKind`: If we have `var_hir_id` and access to any of the tables (i.e. access to `closure_def_id`), we can access the `CaptureKind` for the variable from `upvar_capture_map` using `UpvarId(closure_def_id, var_hir_id)`.
- `AccessSpan`: Span in `upvars_mentioned` and points to an access of the captured variable within the closure. We can assume that we have `var_hir_id` to access this. Since we have `var_hir_id`, we also have access to definition span using using the [hir API](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#method.span).
- `Field`: When we build the MIR, we desugar the closure as a structure where captures are represented by fields within the structure. We refer to these using indecies. `Field(var_hir_id)` expresses we need the `var_hir_id` of the ith capture, provided some `i`.
- `Index`: As explained above when we build the MIR, we start representing the closure as a structure where some of the fields represent the captured variables. To build this desugared structure, we need to map `var_hir_id` to an index. For `closure_captures[c4]`, we have `hir_id_2` at index 0, and `hir_id_1` at index 1. (Order in which the captures apprear within the closure)


| Module                 | U/C/B | Purpose                                    |  Uses    |
|------------------------|-------|--------------------------------------------|----------|
| [typeck/closure.rs]    | U | Initialize Substs. **TODO: #4** | `var_hir_id`
| [typeck/upvar.rs (1)]  | U | Initialize `closure_captures` and `upvar_capture_map`| `var_hir_id`
| [typeck/upvar.rs (2)]  | U->C, B | Ensure initial and final types of captured variables unify| `var_hir_id`, `CapturKind`
| [expr_use_visitor]     | U->C, B | Handle captures in case of nested closures | `var_hir_id`, `CapturKind`
| [mem_categorization]   | U | Check if the expr uses a captured variable| `var_hir_id`
| [passes/liveness.rs]   | U->C | Liveness checks, reports unused upvars | `var_hir_id`, `AccessSpan`
| [mir_build (1)]        | U->C | hir::Expr -> mir::Expr | `var_hir_id`
| [mir_build (2)]        | C | Desugar a capture and represent it as field within a struct | `var_hir_id`, `Index`
| [mir_build (3)]        | C | Build [upvars_mutbls] | `var_hir_id` | `var_hir_id`, `CaptureKind`
| [librustc_middle/mir]  | U->C | Debug print | `var_hir_id`
| [borrow_check (1)]     | U->C | Debug message about the ith capture | `Field(var_hir_id)`
| [borrow_check (2)]     | C, B | Generate borrow check upvar| `var_hir_id`, `CaptureKind`
| [borrow_check (3)]     | U->C | Span within the closure for the ith capture | `Field(AccessSpan)`
| [interpret]            | C | Figure out which variable is being captured | `var_hir_id`, `Field`
| [region_errors]        | U->C | Report FnMut error | `AccessSpan`
| [pretty.rs]            | U->C? | Pretty print the names of the captured variables | `var_hir_id`
| [trait_selection]      | U->C? | Search for type that maches target return type | `var_hir_id`, `AccessSpan`


`TypeckTables` also provides an API ([`upvar_capture`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TypeckTables.html#method.upvar_capture)) that takes an `UpvarId` and returns the `CaptureKind`.

We can assume that if `upvars_mentioned` or `closure_captures` wasn't called in the neighbourhood,
then we somehow have access to the `UpvarId`. Most likely we are within the `typeck` module or have access to an
[`UpvarRegion`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_infer/infer/enum.RegionVariableOrigin.html#variant.UpvarRegion)
created in [`typeck/upvar.rs`](https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/check/upvar.rs#L130) or have `var_hir_id` and the `closure_def_id`.

## Proposed Solution

We can replace `closure_captures` and `upvar_capture_map`

**@nikomatsakis** [said](https://rust-lang.zulipchat.com/#narrow/stream/189812-t-compiler.2Fwg-rfc-2229/topic/Typecheck.20tables.20using.20Places/near/202033086):

`FxIndexMap<hir::Place, CaptureInfo>` where you have
```rust
struct CaptureInfo {
    /// the id of some use that caused us to pick the capture-mode below
    example_use_at: HirId,

    /// captured mode we selected
    capture_mode: Mode,
}
```

### Satisfiability

| Module                 | ✅ / 😭  | How?                                               |
|------------------------|----------|----------------------------------------------------|
| [typeck/closure.rs]    |
| [typeck/upvar.rs (1)]  |
| [typeck/upvar.rs (2)]  |
| [expr_use_visitor]     |
| [mem_categorization]   |
| [passes/liveness.rs]   |
| [librustc_middle/mir]  |
| [mir_build (1)]        |
| [mir_build (2)]        |
| [mir_build (3)]        |
| [borrow_check (1)]     |
| [borrow_check (2)]     |
| [borrow_check (3)]     |
| [interpret]            |
| [region_errors]        |
| [pretty.rs]            |
| [trait_selection]      |


[typeck/closure.rs]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/check/closure.rs#L93
[typeck/upvar.rs (1)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/check/upvar.rs#L114
[typeck/upvar.rs (2)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/check/upvar.rs#L221
[expr_use_visitor]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/expr_use_visitor.rs#L539
[mem_categorization]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_typeck/mem_categorization.rs#L469
[passes/liveness.rs]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_passes/liveness.rs
[librustc_middle/mir]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_middle/mir/mod.rs#L2594
[borrow_check (1)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir/borrow_check/diagnostics/mod.rs#L383
[borrow_check (2)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir/borrow_check/diagnostics/mod.rs#L827
[borrow_check (3)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir/borrow_check/mod.rs#L143
[region_errors]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir/borrow_check/diagnostics/region_errors.rs#L392
[mir_build (1)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir_build/hair/cx/expr.rs#L391
[mir_build (2)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir_build/hair/cx/expr.rs#L872
[mir_build (3)]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir_build/build/mod.rs#L821
[pretty.rs]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_middle/ty/print/pretty.rs#L667
[upvar_mutbls]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_build/build/struct.Builder.html#structfield.upvar_mutbls
[interpret]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_mir/interpret/validity.rs#L233
[trait_selection]: https://github.com/rust-lang/rust/blob/9bdd2db3a60176012f4dc240eea02d615cc60061/src/librustc_trait_selection/traits/error_reporting/suggestions.rs#L1458
