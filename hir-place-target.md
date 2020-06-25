We are going to be refactoring [HIR places] to capture sufficient detail for us to use them to describe the places that a closure captures.

The expected target structure looks something like this:

```rust
/// A reference to a particular place that appears in the source code
struct PlaceWithHirId<'tcx> {
    /// the place being referenced
    place: Place<'tcx>,

    /// hir-id of source expression or pattern
    hir_id: HirId,

    // Maybe no Span, since they're being removed from the Hir
}

/// An expression that refers to some place in memory, similar
/// to MIR places
struct Place<'tcx> {
    /// start of the place expression, typically a local variable
    base: PlaceBase,

    /// Type of the Base
    base_ty: Ty<'tcx>,

    /// projections select parts of the base expression; e.g.,
    /// in the place expression `a.b.c`, `b` and `c` are projections
    projections: Vec<Projection<'tcx>>,
}

/// *Projections* select parts of the base expression; e.g.,
/// in the place expression `a.b.c`, `b` and `c` are projections
struct Projection<'tcx> {
    /// Type before the projection is applied.
    before_ty: Ty<'tcx>,

    /// type of the projection.
    kind: ProjectionKind,

    /// Type after the projection is applied.
    after_ty: Ty<'tcx>,
}

/// Kinds of projections
enum ProjectionKind {
    /// `*B`, where `B` is the base expression
    Deref,

    /// `B.F` where `B` is the base expression and `F` is
    /// the field. The field is identified by which variant
    /// it appears in along with a field index. The variant
    /// is used for enums.
    Field(Field, VariantIdx),

    /// Some index like `B[x]`, where `B` is the base
    /// expression. We don't preserve the index `x` because
    /// we won't need it.
    Index,

    /// A subslice covering a range of values like `B[x..y]`.
    Subslice,
}
```
