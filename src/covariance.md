# Covariance: when "lifetimes" can shrink

So, from the previous chapter we've seen that sometimes, there is a seemingly silly operation which can be done: that of shrinking or shortening expiry-dates / regions / "lifetimes".

For instance, let's consider the simplest lifetime-infected type: a borrow (to, say, some primitive type):

```rs
&'borrow i32
// but also:
&'borrow mut i32
```

When you got your hands on such a thing with a `'long` borrow, if you have to feed it to some API that wants a `'short` borrow instead, you can!

```rs
use ::core::convert::identity;

fn demo<'short, 'long : 'short>(
    r: &'long mut i32,
) -> &'short mut i32
{
    identity::<&'short mut i32>(r) // ✅ OK!
}
```

This is called a _subtyping relation_, and [you may observe that it is quite similar to an implicit coercion](./subtyping-vs-coercions.md).

### Is shrinking lifetimes really "silly"?

<details><summary>Click to see this section</summary>

I guess the main example would be when trying to put together references with distinct lifetimes into a single collection such as a `Vec`:

```rs
let local = String::from("…");
let names: Vec<&str> = vec![local.as_str(), "static str"];
```

Or, similarly, when type-unifying the output of two branches:

```rs
let storage: PathBuf;
let file_to_read: &Path =
    match ::std::env::args_os().nth(1) {
        | Some(s) => {
            storage = s.into();
            &storage
        },
        // no `.to_owned()` whatsoever needed in this branch 💪
        | None => "default-file.toml".as_ref(),
    }
;
```

Finally, remember the [`pick()` function section](pick-function.md):

```rs
fn pick<'ret>(
    left: &'ret str,
    right: &'ret str,
) -> &'ret str
{
    if … { left } else { right }
}
```

Imagine trying to call `pick()` on a `&String::from("…")` and a `"static str"`: with this signature, the borrows are constrained to have equal lifetimes! Lifetime shrinkage to the rescue 🙂

Even in the more pedantic case of doing:

```rs
fn pick<'ret, 'left, 'right>(
    left: &'left str,
    right: &'right str,
) -> &'ret str
where
    'left : 'ret,
    'right : 'ret,
{
    // Here, we have `'left` and `'right` lifetimes in  branch;
    // by covariance they get to shrink-unify down to some `'ret` intersection.
    if … { left } else { right }
}
```

notice how the `left` and `right` expressions in each branch of the `if`, respectively, _by allowing the lifetime within each to shrink_, managed to unify under a single shrunk lifetime, `'ret` (cough [intersection](./intersection-lifetime.md) cough).

</details>

## Covariance, a definition

### Covariant lifetime parameter

So, this property of certain **generic** lifetime **parameters** being allowed to **shrink** is called **covariance** (in that lifetime parameter). While most _generic_ types will happen to be _covariant_, there is an important exception / counter-example which we need to be aware of:

  - [x] mutable…
  - [x] …borrows!

When both properties are "ticked"/met, then any lifetime _in the borrowee_ is not (allowed) to be shrunk. We say of such a lifetime that it is **non-covariant**, or for short[^contra], **invariant**.

  - ⚠️ notice how we're talking of the lifetimes that _may_ occur within _the borrowee_: the lifetime of the borrow itself is not part of the invariant stuff.

For instance, let's look at the type of that second argument of `{Display,Debug}::fmt` function:

```rs
&'_ mut Formatter<'_>
```

Naming the lifetimes, we end up with:

```rs
// mutable borrow of
//  vvvvvvvvvvvv
    &'borrow mut Formatter<'fmt>
//               ^^^^^^^^^^^^^^^
//              invariant borrowee



//  can shrink
//      =
//  covariant
//   vvvvvvv
    &'borrow mut Formatter<'fmt>
//                         ^^^^
//                     cannot shrink (nor grow)
//                           =
//                       invariant
```

A more typical albeit potentially visually confusing example is when the borrowee is itself a borrow:

```rs
&'borrow mut (&'fmt str)
```

In both cases, we have a `&mut`-able `'borrow`, with thus a _covariant_/shrinkable lifetime, and behind such borrow, a thus _invariant_ borrowee, resulting in `'fmt` not being allowed to shrink (nor grow).

  - #### Reminder that shared mutability exists

    Note that the _unique borrow_, `&mut`, is not the only form of _mutable borrow_. Consider, for instance:

    ```rs
    // Covariant in `'borrow`, invariant in `'fmt`.
    &'borrow Cell<&'fmt str>
    ```

      - To gain a better intuition of this, I highly recommend that you open your mind to the following "statement":

        ```rs
        use ::core::cell::Cell as Mut;
        ```

          - (this is a legitimate thing to do since `Cell` does indeed let you mutate "its interior")

        Thay way we end up with:

        ```rs
        &'borrow Mut<&'fmt str>
        ```

    Finally, for technical reasons[^composition], it turns out that the harmless _owned_ shared mutability wrappers have to be declared invariant as well, despite the lack of borrows:

    ```rs
    // Invariant too!
    Cell<&'_ str>
    ```

    In practice, you're more likely to run into this when dealing with things like a <code>**Mut** ex</code>, a `RwLock<_>`, or any form of `channel` (at least the `Sender<_>` end):

    ```rs
    //! Invariant
    Mut ex<&'_ str>
    RwLock<&'_ str>
    RefCell<&'_ str>
    channel::Sender<&'_ str>
    ```

### Covariant type parameter and composition rules

It turns out that saying:

  - given `&'borrow /* Borrowee */`,
  - then, each lifetime occurring in `Borrowee` (if any),
  - is covariant;

is kind of a mouthful to say/write/express. We thus define this conventional term, when this occurs: we'll talk of `&'borrow T` being covariant in the `T` _type parameter_.

So a not-so-formal definition of type-parameter-covariance would be:

  - #### Determining whether a `<T>`-generic type is covariant

    A practical rule (of thumb) for determining whether a `<T>`-generic type is covariant (in `T`) is to replace `T` by, say. `&'s str`, and seeing the covariance (in `'s`) of the resulting type, since that will then be the covariance of the `<T>`-generic type.

    Examples:

      - <details><summary>What is the covariance of <code>type Example&lt;T&gt; = (T, i32);</code>?</summary>

         1. We replace `T` with `&'s str`: what is the covariance of `(&'s str, i32)`?

         1. We somehow know that `type ExampleT<'s> = (&'s str, i32)` is covariant (in `'s`).

         1. "Thus", `Example<T>` is covariant (in `T`).

        </details>

      - <details><summary>What is the covariance of <code>type Example&lt;T&gt; = Arc&lt;Cell&lt;T&gt;&gt;;</code>?</summary>

         1. We replace `T` with `&'s str`: what is the covariance of `Arc<Cell<&'s str>>`?

         1. We know that "lifetimes behind a `Cell` are invariant", so we know that:

            `type ExampleT<'s> = Arc<Cell<&'s str>>` is invariant (in `'s`).

         1. "Thus", `Example<T>` is invariant (in `T`).

        </details>
  - #### Composition rules: when you know that a `<T>`-generic type is covariant:

    For instance, consider `type Vec<T>` (or, more generally, any `type F<T>`).

    Now define some type depending on a lifetime parameter:

    ```rs
    type T<'lt> = (i32, &'lt str, bool); // FYI, this is covariant.
    ```

    Then the resulting type, `F<T<'lt>>`, _i.e._, `Vec<(i32, &'lt str, bool)>`, will have the _same variance_ as `T<'lt>` alone: **`<T>`-covariant types** let the `<'lt>`-variance inside `T` **pass through**.

[^contra]: when _also_ non-contravariant.

[^composition]: if `Cell<T>`, owned, were covariant in `T`, since `&C` is covariant in `C`, it would mean that `&Cell<T>`. Which can't be. So `Cell<T>` can't be covariant in `T`.
