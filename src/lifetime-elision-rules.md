# Lifetime elision

  - Prerequisite: [elided lifetimes](./elided-lifetimes.md).

The lifetime elision rules are just there as **convenience sugar** for what could otherwise be written in a more verbose manner. As with any sugar that does not want to become overly terse and footgunny (cough, C++, cough), it only makes sense to have it **in clear unambiguous cases**.

## Lifetime elision in function _bodies_

Inside function _bodies_, that is, in a place where _type inference is allowed_, any elided lifetimes, implicit or not, will adjust to become whatever suits the code / whatever type-inference (with borrow-checking) dictates.

## Lifetime elision in function signatures

By "function signatures" the following rules will apply to:

  - A function (item) signature:
      - ```rs
        fn some_func(…) -> …
        ```

  - A function pointer type:
      - ```rs
        type MyCb = fn(…) -> …;
        //          ^^^^^^^^^^
        ```

  - A `Fn{,Mut,Once}` trait bound:
      - ```rs
        : Fn{,Mut,Once}(…) -> …
        ```
      - ```rs
        impl Fn{,Mut,Once}(…) -> …
        ```
      - ```rs
        dyn Fn{,Mut,Once}(…) -> …
        ```

So, for all these, it turns out there are only two categories of "clear unambiguous cases":

  - ### When lifetimes play no meaningful role

    That is, when the function returns nothing that borrows or depends on inputs. That is, **when the return type has no "lifetime" parameters**:

    ```rust
    fn eq(s1: &'_ str, s2: &'_ str)
      -> bool // <- no "lifetimes"!
    ```

    Since there isn't really any room for lifetimes/borrows subtleties here, the unsugaring will be a maximally flexible one. Given that **repeating a lifetime parameter name is a restriction** (wherein two lifetime-generic types will need to be using equal "lifetimes" / matching regions), we get to be maximaly flexible / lenient / loose by _not_ doing that: by introducing and using _distinct lifetime_ parameters for each lifetime placeholder:

    ```rust
    fn eq<'s1, 's2>(s1: &'s1 str, s2: &'s2 str)
      -> bool
    ```

  - #### With borrowing _getters_

    With thus a clear unambiguous receiver / thing being borrowed:

      - ```rust
        fn get(from_thing: &'_ Thing) // same for `BorrowingThing<'_>`.
          -> &'_ SomeField
        // replacing `&'_` with `&'_ mut` works too, of course.
        ```

         1. In case of **single "lifetime" placeholder among all the inputs**;
         1. ⇒ the borrow of `SomeField` necessarily stems from it;
         1. ⇒ the output is to be "connected" to that input borrow by **repeating the lifetime parameter**.

            ```rs
            fn get<'ret> (from_thing: &'ret Thing)
              -> &'ret SomeField
            ```

      - ```rust
        fn get(&'_ self, key: &'_ str)
          -> &'_ Value
        ```

         1. **A `&[mut] self` _method receiver_**, despite the multiple lifetime placeholders among all the inputs, is "favored" and by default deemed to be the "thing being borrowed".
         1. ⇒ the borrow of the `Value` is to be "connected" to the `self` receiver; and `key`, on the other hand, is in a "I'm not borrowed" kind of situation.
         1. ⇒ `key` will thus be using its own distinct "dummy" lifetime, whereas `self` and the returned `Value` will be using a single/repeated lifetime parameter.

            ```rs
            fn get<'ret, '_key> (
                self: &'ret Self,
                key: &'_key str,
            ) -> &'ret Value
            ```

Regarding `fn`-pointers and `Fn…` traits, the same rules apply, it's just that the named "generic" lifetime parameters for these things have to be introduced as "nested generics" / higher-rank lifetime parameters, using the `for<>` syntax.

  - In the `fn`-pointer case, we have:
    ```rs
    let eq: fn(s1: &'_ str, s2: &'_ str) -> bool;
    // stands for:
    let eq: for<'s1, 's2> fn(s1: &'s1 str, s2: &'s2 str) -> bool;
    ```

  - In the `Fn…` case, we have:
    ```rs
    impl/dyn Fn…(&'_ str, &'_ str) -> bool>
    // stands for
    impl/dyn for<'s1, 's2> Fn…(&'s1 str, &'s2 str) -> bool
    ```

So the rule of thumb is that "lifetime holes in ouput connect with an unambiguous lifetime hole in input".

  - Technically, lifetime holes can connect with named lifetimes, but mixing up named lifetimes with elided lifetimes for them to "connect" is asking for trouble/confusing code: it ought to be linted against, and I hope it will end up denied in a future edition:

    ```rs
    fn example<'a>(
        a: impl 'a + AsRef<str>,
        _debug_info: &'static str,
    ) -> impl '_ + AsRef<str>
    {
        a // Error, `a` does not live for `'static`
    }
    ```

## Lifetime elision in `dyn Traits`

Yep, yet another set of rules _distinct_ from that of elision in function signatures, and this time, for little gain 😔

  - tip: if you're the one writing the Rust code, avoid using these to reduce the cognitive burden of having to remember this extra set of rules!

This is a special form of _implicit_ elision: `dyn Traits`[^traits]. Indeed, behind a `dyn` necessarily lurks a "lifetime"/region: that of owned-usability.

```rs
dyn Traits // + 'usability?
```

  - ⚠️ this is different than the explicitly elided lifetime, `'_`.

    That is, if you see a `dyn '_ + Trait` it will be the rules for [lifetime elision in function signatures](#lifetime-elision-in-function-signatures) which shall govern what `'_` means:

    ```rs
    fn a(it: Box<dyn Send>) // :impl Fn(Box<dyn 'static + Send>)

    fn b(it: Box<dyn '_ + Send>) // :impl for<'u> Fn(Box<dyn 'u + Send>)
    ```

[^traits]: When I write `Traits`, it is expected to cover situations such as `Trait1 + Trait2 + … + TraitN`

### Rules of thumb for elision behind `dyn`:

 1. Inside of function bodies: [type inference](#lifetime-elision-in-function-bodies).

 1. `&'r [mut] dyn Traits = &'r [mut] (dyn 'r + Traits)`

      - More generally, given `TyPath<dyn Traits>` where `TyPath<T : 'bound>`, we'll have `TyPath<dyn Traits> = TyPath<dyn 'bound + Traits>`.

 1. otherwise `dyn Traits = dyn 'static + Traits`

## Lifetime elision in `impl Traits`

### Return-position `-> impl Traits` (RPIT)

Similarly to `dyn`, here, `-> impl Traits ≠ -> impl '_ + Traits`.

But contrary to other properties/aspects of type erasure (where `dyn` and `-> impl` can be quite similar), when dealing with lifetimes / captured generics, `impl Traits` happens to involve different semantics than those of `dyn Traits` ⚠️

See [the dedicated section](return-position-impl-trait.md) for more info.


### Argument-position `impl Traits` (APIT)

  - (dubbed "universal" / caller-chosen)

**Intuition**: `impl Traits` will "be the same as `impl '_ + Traits`", that is, introducing a new named generic lifetime parameter `<'usability>`, and then `impl 'usability + Traits`. This aligns with the idea of it being a "universal" `impl` type.

<details><summary>Rationale / actual semantics</summary>

These are kind of analogous to the following:

For each `impl Traits` occurrence,
 1. introducing a new generic _type_ parameter: `<T>`
 1. bounded by the `Traits`: `where T : Traits`
 1. and replacing the `impl Traits` with that parameter: `T`.

```rs
fn example(a: impl Send, b: impl Send)
// is the "same" as:
fn example<A, B>(a: A, b: B)
where
    A : Send,
    B : Send,
```

  - the only difference will be that in the latter signature, callers will be able to use turbofish to specify the actual choices of `A` or `B`, whereas in the former case this will be left for type inference.

___

</details>
