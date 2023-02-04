# Rust lifetimes: from `'static` to ecstatic.






___

# Generic ~~lifetime~~ region( parameter)s

  - Idea: since single/specific "lifetimes"/regions are unnameable, functions wanting to handle these are forced to handle them all / to be generic over that "lifetime"/region.

Generic region parameters are the main tool used by Rust to have borrow-checking (solving these "interval inequalities problems") be:

  - a global property of a Rust codebase (the soundness property: non-`unsafe` code that passes borrow-checking cannot exhibit, at runtime, a danling pointer);
  - that can nonetheless be checked, and reasoned about, _locally_:
      - Rust can check the "lifetimes" and borrows of a function simply by matching the function _body_/implementation against the function _signature_: for a given `callee` function signature, the call-sites of such function do not matter w.r.t. borrow-checking it (this is the whole principle of abstraction, and is how _libraries_ can even exist).
      - within some `caller` scope that calls `callee`, Rust can borrow-check that call exclusively by looking at `callee`'s function signature (and `caller`s variables): the body of `callee` does not matter! (this is what lets these functions be way more future-proof w.r.t. changes in `lib::callee`). function needs not to know how it is called
      /  while keeping functions untangled from one another.

That is, imagine doing:

```rs
//! Imaginary Rust pseudo-code

let s = String::from("…");
// let's imagine this defines a global `'borrow` "symbol":
#[new_lifetime(let 'borrow_of_s)]
let r: &'borrow_of_s String = &s;
factored_out_code(r)
// where:
fn factored_out_code(r: &'borrow_of_s String) {
    …
}
```

  - which would be a way to factor out code into a helper function while still borrow-checking that _specific_ `'borrow_of_s` held by `r`.

Real Rust, instead, will decide that any factored out code that can handle a non-`'static` region necessarily has to involve a _generic region parameter_:

```rs
let s = String::from("…");
// let's imagine this defines a global `'borrow` "symbol":
let r: &'_ String = &s;
factored_out_code(r)
// where:
fn factored_out_code<'some_borrow>(r: &'some_borrow String) {
    …
}
```

If we look at the point of view of the `factored_out_code` callee, it is thus written in a way where it needs to be able to handle any kind of choice –_by the caller!_– of that `'some_borrow` parameter:
  - whilst it could be called with `'some_borrow = 'borrow_of_s`,
  - it could also be called with `'some_borrow = 'static`,
  - _etc._ for any of the infinite range of possible choices of `'some_borrow`.
    ```rs
    /// Let's rename the region parameter:
    fn factored_out_code<'any>(r: &'any String) {
    ```
    - spoiler: this "any" / capable-of-handling-an-infinite range-of-possible-choices _universal quantification_ is expressible in rust using `for<'regions>` syntax:
      ```rs
      factored_out_code: for<'any> fn(&'any String)
                       : impl for<'any> Fn(&'any String)
      ```

### ⚠️ A universal quantification, granted, but limited nonetheless

Let's go over that nuance added above:

> any kind of choice –_by the caller!_– of that `'…` parameter

Indeed, the choice of `'some_borrow` is nonetheless made by the caller / unknown to the callee. Indeed:
  - in the same fashion that **the callee cannot see, across the function boundary, a region from the caller** (_e.g._, it cannot see the `'borrow_of_s` region)
      - (which is what requires the callee to be generic/universal over regions in order to hope to cover/encompass the region picked of the caller)
  - **the caller cannot see across the function boundary a region from the callee**
      - (which is what would require _the caller_ to be generic/universal over regions in order to hope to cover/encompass the region used by the callee).

**Function boundaries are opaque to any specific region choice made across it, and the only way for such a choice to be used across them is for the other side of the boundary to be _generic_ / "universal".**

This is why the following snippet fails:

```rs
// generic function taking a non-generic callback: ❌
fn callee<'some_borrow>(caller_cb: fn(&'some_borrow str)) {
    let local = String::from("…");
    let r: &'some_borrow str = &local; // Error! ❌
    caller_cb(r);
}
```

  - There is an easy way to see the problem considering an adversarial caller: what if they picked `'some_borrow = 'static`?

Here, the region/duration of the borrow of the `local` is completely an implementation detail of the callee (thus opaque to the caller), and the only one who could be able to "pick" that region / make that choice of region is the callee, not the caller.

  - #### Tip: "visualizing" the span of a generic region parameter

    In practice, a generic region parameter (introduced at `impl<…>` or `fn fname<…>`), represents a region which _necessarily spans beyond the `return`s point / the closing `}` of a function_:

    ```rs
    fn callee<'some_borrow>() {
        let local = String::from("…");
        let r: &'some_borrow str = &local; // Error! ❌

    // <- `local` dies here, so its borrow must have expired by now.

    } // ----------- END OF FUNCTION ------------------

    // <- `'some_borrow` at least spans over this point, by virtue of
    //    being a generic lifetime parameter.
    ```

The only way for the caller to be able to take a callee-chosen borrow is for them to be the generic party, here:

```rs
// non-generic function taking a generic-callback: ✅
fn callee(caller_cb: fn<'some_borrow>(&'some_borrow str)) {
    let local = String::from("…");
    let r: &'_ str = &local;
    caller_cb(r);
}
```

Granted, `fn<'some_borrow>(&'some_borrow str)` is not real Rust code, but pseudo-code (which hopefully illustrates the point). The semantics, however, do exist, they just require a slightly more convoluted syntax. The real Rust syntax for this notion is actually:

```rs
// non-generic function taking a generic-callback: ✅
fn callee(caller_cb: for<'some_borrow> fn(&'some_borrow str)) {
```

  - Caller usage:
    ```rs
    'caller /* scope */ : {
        callee(caller_cb);
        // where:
        fn caller_cb<'some_borrow>(s: &'some_borrow str) {
            …
        }
    }
    ```

    with closure syntax:

    ```rs
    #![feature(closure_lifetime_binders)]

    'caller /* scope */ : {
        callee(for<'some_borrow> |s: &'some_borrow str| {
            …
        })

        // or, with "`for<>` promotion of simply-elided lifetimes":
        callee(|s: &'_ str| {
            …
        })
    }
    ```

    <details><summary>about <code>for&lt;&gt;</code> promotion of simply-elided lifetimes</summary>

    This is a very hacky area of Rust with a lot of _ad-hoc_ heuristics:

    ```rs
    'caller /* scope */ : {
        // an elided lifetime —explicitly, `'_`, or implicitly, `&`—
        // in closure-arg position (before type inference!),
        // gets "higher-order promoted" into a `for<>`/generic parameter:
        callee(|s: &'_ str| { // ✅
            …
        })
        /* this does not work in return-type position, though */

        // Also, when the signature of `callee` explicitly requires a
        // `impl for<'any> Fn(&'any str)` kind of parameter (or a subtrait
        // thereof), we can, *exceptionally*, delegate to type inference,
        // but only *provided that the literal closure expression be inlined
        // as an argument of such a function* (this does work even for
        // `for<>` lifetimes in return type position)
        callee(|s /* : <type inferred> */| { // ✅
            …
        });

        // Otherwise the promotion does not kick in.
        // For instance, the following fails:
        let caller_cb = |s| {
            // no matter how much we help type-inference here…
            let _: &str = s;
            …
        };
        // the fact that:
        //   - neither the closure arg has a visible lifetime placeholder;
        //   - nor it is inlined in argument position of a function expecting
        //     an `impl for<'any> Fn…` kind of signature
        // means higher-order promotion does not kick in, and we get an error:
        callee(caller_cb); // ❌
    }
    ```

    </details>
