# Language Changes

This document lists changes that will be made to language along with RFC-2229.

For this document let RS20 be the pre-RFC compiler and RS21 be the post-RFC compiler.

## `_` pattern in closures

Consider the following code 

```rust 
let x;
let c = || {
      let _ = x;
}; 
```

RS20 will result in the capture of x within the closure, but `_` is just an ignore/discard and that means x can't/won't be used within the closure and should not result in a capture.

For RS21 we propose to ignore such captures allowing the following pieces of code to be accepted by RS21

```rust
let x;
let c = || {
      let _ = x;
};

let f : fn() = c; 

f();
```

The above code works now because `x` isn't captured by `c` anymore, making `c` a non-capturing closure, which can be coerced to a `FnPtr`.

```rust
fn main() {
    let mut x = String::from("1");
    
    let c = || {
        let _ = x;
    };
    
    
    let mx = &mut x;
    *mx = String::from("2");
    
    c();
    
    println!("{}", x);
}
```
RS20 would immutably borrow x within the closure (even thought it's discarded), preventing a mutable borrow to exist outside the closure. 
The above code works with RS21 because `x` isn't captured within the closure allowing the mutable borrow outside the closure to exist.
