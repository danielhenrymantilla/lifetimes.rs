# Subtyping vs. Coercions

  - Example of subtyping:

    ```rs
    fn demo<'short>()
    {
        let s: &'static str = "…";
        let s: &'short str = s;
    }
    ```

  - Examples of coercions:

    ```rs
    fn implicit_coercions(r: &'_ mut Box<[u8; 4]>)
    {
        // `&mut` to `*mut` (& friends)
        let _: *mut _ = r /* as _ */;

        // `&mut` to `&`:
        let r: &Box<[u8; 4]> = r /* as _ */;

        // `&impl Deref` to `&Deref::Output`
        let r: &[u8; 4] = r /* as _ */;

        // Unsized coercions:
        let _: &[u8] = r /* as _ */;
        let _: &dyn ::core::any::Any = r /* as _ */;
    }
    ```

There are two differences between subtyping and implicit coercions:

  - Coercions are allowed to perform (basic) _runtime operations_ to perform the conversion. For instance, the coercion from `&[u8; 4]` (thin data pointer to the beginning of 4 bytes) to `&[u8]` (attaching a runtime `4: usize` "field" next to that thin data pointer to end up with a wide pointer).

    Whereas subtypining relations cannot, so as to have the following property:

  - Subtyping relations can be nested/chained within arbitrarily deep complex types (provided that they be [covariant](./covariance.md)), whereas coercions can't:

    ```rs
    fn ok<'short, 'long : 'short>(v: Vec<&'long mut i32>)
      -> Vec<&'short mut i32>
    {
        v // ✅ OK
    }

    fn nope(v: Vec<&mut i32>)
      -> Vec<&i32>
    {
        v // ❌ Error
    }
    ```

The latter point, I'd say, is really the critical difference from a theoretical perspective, since, for instance, given some `'short` lifetime, consider the following "generic type" / type constructor:

```rs
<T> => &'short T
```

It is a rather simple type, which happens to be covariant in `T`.

### Covariance (using `&'long X <: &'short X`)

Let's try with `type T = &'long String;`, and trying to reach the `type T = &'short String;`, since the former subtypes the latter:

```rs
&'short (&'long String) <: &'short (&'short String)
// by covariance (in T) of `&'short T`
// since &'long String <: &'short String
// since 'long ⊇ 'short
```

✅

### Coercion? (using the `&'long mut X -> &'long X` coercion)

Now let's try with `type T = &'long mut String`, and trying to go to `type T = &'long String;`, since we know the former can be coerced to the latter:

```rs
fn test<'short, 'long : 'short>(
    r: &'short (&'long mut String),
) -> &'short (&'long String)
{
    r /* as _ */
}
```

This errors! ❌

<details><summary>Error message</summary>

```rs
error[E0308]: mismatched types
 --> src/lib.rs:5:5
  |
3 | ) -> &'short (&'long String)
  |      ----------------------- expected `&'short &'long String` because of return type
4 | {
5 |     r /* as _ */
  |     ^ types differ in mutability
  |
  = note: expected reference `&'short &'long String`
             found reference `&'short &'long mut String`
```

</details>

Not only does it error, but there is actually no way to write this soundly.

Indeed, such operation is unsound, _i.e._, it makes it possible, using it and non-`unsafe` Rust, to trigger UB/memory-corruption:

 1. Given:
    ```rs
    fn unsound<'short, 'long : 'short>(
        r: &'short (&'long mut String),
    ) -> &'short (&'long String)
    {
        unsafe { ::core::mem::transmute(r) }
    }
    ```

 1. One can write:

    ```rs
    /// Here is an example of what `unsound` lets us do:
    fn exploit_helper<'long>(
        s: &'long mut String,
    ) -> (
            &'long String,
            &'long mut String,
        )
    {
        // `s` cannot be used while `short` (or data
        // derived from it) is being used.
        let short: &'_ &'long mut String = &s;
        // the `'short` lifetime in `changed` is still
        // derived from `short`, so `s` still can't be used
        let changed: &'_ &'long String = unsound(short);
        // But we can now dereference `changed`,
        // since `&'long String : Copy`,
        // and the resulting `&'long String` is no longer
        // tied to `short`.
        let r: &'long String = *changed;
        // That is, we can freely use `s` even if `r` is used too!
        (r, s)
    }
    ```

    Which means we have a (shared) reference _aliasing_ with an _exclusive_ one! UB

 1. For instance:

    ```rs
    let s = &mut String::from("Hey!");
    let (r, s) = exploit_helper(s);
    // Have `r` point to the heap-allocated utf-8 bytes the `String` owns.
    let r: &str = &*r;
    // And now let's deallocate those bytes:
    *s = String::new();
    // read deallocated memory:
    dbg!(r);
    ```

      - [Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=bc7bce1298bf9f48214e7462a2f91151)

    ```rs
    error: Undefined Behavior: trying to retag from <3360> for SharedReadOnly permission at alloc1685[0x0], but that tag does not exist in the borrow stack for this location
      --> src/main.rs:19:5
       |
    19 |     (r2, r)
       |     ^^^^^^^
       |     |
       |     trying to retag from <3360> for SharedReadOnly permission at alloc1685[0x0], but that tag does not exist in the borrow stack for this location
       |     this error occurs as part of retag at alloc1685[0x0..0x18]
       |
       = help: this indicates a potential bug in the program: it performed an invalid operation, but the Stacked Borrows rules it violated are still experimental
       = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md for further information
    help: <3360> was created by a SharedReadOnly retag at offsets [0x0..0x18]
      --> src/main.rs:19:6
       |
    19 |     (r2, r)
       |      ^^
    help: <3360> was later invalidated at offsets [0x0..0x18] by a Unique retag
      --> src/main.rs:19:10
       |
    19 |     (r2, r)
       |          ^
       = note: BACKTRACE (of the first span):
       = note: inside `exploit_helper` at src/main.rs:19:5: 19:12
    note: inside `main`
      --> src/main.rs:25:18
       |
    25 |     let (r, s) = exploit_helper(s);
       |                  ^^^^^^^^^^^^^^^^^
    ```
