# Update closure pretty printing

Currently the compiler prints the capture information (name and type) of the root variable being captured.  For example, current pretty print:

```rust
struct Point {
    x: i32,
    y: i32,
}

let p = Point { x: 10, y: 20 };

// pretty print: [closure@src/main.rs:8:14: 10:6 p:_]
let c1 = || {
    println!("{}", p.x);
};

// pretty print: [closure@src/main.rs:11:14: 13:6 p:_]
let c2 = || {
    println!("{}", p.y);
};
```
(`p:_`, means p is captured and its type hasn't been inferred yet)

If `-Zverbose` is set the closure pretty print will also include the closure signature and closure kind (eg: i32 for FnOnce).


## Motivation for change
- Information about closure kind, signature and capture types isn't really diagnostically useful to the end user.
- Once [RFC 2229](https://github.com/rust-lang/rfcs/blob/master/text/2229-capture-disjoint-fields.md) is implemented, the current closure pretty print won't be completely accurate. In the above example, after RFC-2229 `c1` only captures `p.x` and c2 only captures `p.y`.
- Closures don't print information about generics they inherit from their parent scope.

## Proposed changes
- Don't print closure information (kind, signature, capture types) unless `-Zverbose` is specified.
- When the compiler is invoked with `-Zverbose`  only print the type of the capture and not the name of or path to the variable.

Applying these two changes to example above we get
```rust
// pretty print: [closure@src/main.rs:8:14: 10:6] 
let c1 = || {
    println!("{}", p.x);
};

// compile-flags: -Zverbose 
// pretty print: [closure@src/main.rs:8:14: 10:6 closure_kind_ty=<kind>, closure_sig_as_fn_ptr_ty=<signature>, upvar_tys=(&i32)]
let c1 = || {
    println!("{}", p.x);
};
```
(Note this example is missing changes explained below)

- If type information captures isn't available or can't be verified as available then the information won't be printed out.
```rust
// compile-flags: -Zverbose 
// pretty print: [closure@src/main.rs:8:14: 10:6 substs=(unavailable)]
let c1 = || {
    println!("{}", p.x);
};

let c :() = c1; // Error here prevents us from doing capture analysis
```
(Note this example is missing changes explained below)

- In case of -Zverbose, print the path (including generics) to the closure instead of the `[closure]` notation, similar to how types defined within the closure are printed. 

Note: If the path is non-ambigous it will be truncated automaitcally, this behaviour can be disbaled using `-Z trim-diagnostic-paths=off`. Check https://github.com/rust-lang/rust/pull/73996 
for more details.

```rust
mod mod1 {
    pub fn f<T: std::fmt::Display>(t: T) 
    {
        let x = 20;
    
        // pretty print: [closure@rc/main.rs:line:col line:col]
        let c = || println!("{} {}", t, x);
        c(); // *
    }
}

fn main() {
    f(format!("S"));
}
```

- If `-Zverbose` is set then pretty print for the closure will be `f<T>::{{closure}}#0 closure_kind_ty=i8, closure_sig_as_fn_ptr_ty=<sig>, closure_signature=(&T, &i32)`. 
- If `-Z trim-diagnostic-paths=off` is _also_ set then pretty print for the closure will be `mod1::f<T>::{{closure}}#0 closure_kind_ty=i8, closure_sig_as_fn_ptr_ty=<sig>, closure_signature=(&T, &i32)`. 
- If  `-Zverbose` is set and the line marked with `*` is changed to `let c1 :() = c`, that is it prevents us to start capture analysis, the pretty print will be `f<T>::{{closure}}#0 substs=(unavailable)`.
