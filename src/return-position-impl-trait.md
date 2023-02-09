# Lifetime semantics of `-> impl Trait`

### `-> impl Trait` automagically captures generic _type_ parameters in scope

```rs
fn foo<'lt, T>(lt: &'lt (), ty: T)
  -> impl Sized
```

is sugar (and the only stable-Rust syntax at the moment) for:

```rs
#![feature(type_alias_impl_trait)]

fn foo<'lt, T>(lt: &'lt (), ty: T)
  -> __ImplTrait<T>
// where the compiler has secretly generated:
pub type __ImplTrait<T> = impl Sized; // no `'lt` generic param!
```

  - with `__ImplTrait<T>` being a compiler-generated `struct`, of sorts, which is generic over `<T>` and thus fully "_infected by_ `T`".

That is, it has the semantics of:

```rs
//! Pseudo-code

fn foo<'lt, T>(lt: &'lt (), ty: T)
  -> impl Sized + InfectedBy<T> // no `+ InfectedBy<'lt>` !
```

> What the helium is `InfectedBy<T>`?

It's pseudo-code of mine to express the property that this `struct __ImplTrait<T>` would have: for any region `'r`, if the return type is usable within that region, then so must `T` itself be:

```rs
for<'r> (
    impl Sized + InfectedBy<T> : 'r
    ⇒
    T : 'r
)
```

Although the practical/intuitive aspect of it is rather the contra-position:

```rs
for<'r> (
    T : !'r // let's assume this is syntax to say that `T : 'r` does not hold.
    ⇒
    impl Sized + InfectedBy<T> : !'r
)
```

To illustrate, let's pick `T = &'t str`, for instance:
  - beyond `'t`, instances of type `T = &'t str` are not allowed to be used.
  - Then it means that beyond `'t`, the returned `-> impl Sized` won't be allowed to be used either (_no matter the function body!_).

These `-> impl Sized + InfectedBy<T>` semantics are what allow the returned instance to contain `ty: T` inside it.

But notice how, despite the `'lt` generic "lifetime" parameter in scope, there is no `+ InfectedBy<'lt>` in the unsugaring I've shown!

  - <details><summary>Example</summary>

    The following works fine:

    ```rs
    fn nothing_doër<'lt>(_: &'lt String)
      -> impl Fn()
    {
        || ()
    }

    let f = {
        let local = String::from("…");
        nothing_doër(&local)
    };
    f() // ✅ OK
    ```

    But the following does not:

    ```rs
    fn nothing_doër<T>(_: T)
      -> impl Fn()
    {
        || ()
    }

    let f = {
        let local = String::from("…");
        nothing_doër(&local)
    };
    f() // ❌ Error, `local` does not live long enough!
    ```

    😬

    ___

    </details>

### `-> impl Trait` does not implicitly capture generic _"lifetime"_ parameters in scope

This is not an oversight of mine (nor of the language), but rather a deliberate choice of the design of `-> impl Trait`. See the associated RFC for more info about it, and the rationale behind that choice.

**The fact there is no `+ InfectedBy<'lt>` means the returned instance won't be allowed to contain `lt: &'lt …` inside it**.

```rs
fn foo<'lt, T>(lt: &'lt (), ty: T)
  -> impl Sized
{
    lt // ❌ Error, return type captures lifetime `'lt`
       //    which does not appear in bounds.
}
```

<details><summary>Error message</summary>

```rs
error[E0700]: hidden type for `impl Sized` captures lifetime that does not appear in bounds
  --> src/lib.rs:11:5
   |
8  | fn foo<'lt, T>(lt: &'lt (), ty: T)
   |        --- hidden type `&'lt ()` captures the lifetime `'lt` as defined here
...
11 |     lt
   |     ^^
   |
help: to declare that `impl Sized` captures `'lt`, you can add an explicit `'lt` lifetime bound
   |
9  |   -> impl Sized + 'lt
   |                 +++++
```

  - By the way that suggestion is not right, since it is not fully generalizable. See below for an example.

</details>

The workaround, currently, is to **make the lifetime appear in bounds**. If your trait in question does not give you room for that, then you can still achieve that by virtue of defining a helper trait for this, which happens to map quite nicely to my previous `InfectedBy<…>` pseudo-code.

 1. Define a helper generic-over-a-lifetime trait:
    ```rs
    /// Dummy empty marker trait.
    pub trait InfectedBy<'__> {}
    impl<T : ?Sized> InfectedBy<'_> for T {}
    ```

 1. Add it (`+ InfectedBy<…>`) to the bounds of the returned `-> impl Trait`:

    ```rs
    //! Real rust code

    fn foo<'lt, T>(lt: &'lt (), ty: T)
      -> impl Sized + InfectedBy<'lt>
    {
        lt // ✅ OK
    }
    ```

FWIW, you can imagine that this procedure is easily automatable through a macro:

  - [`#[fix_hidden_lifetime_bug]`](https://docs.rs/fix_hidden_lifetime_bug)


### About `+ 'lt`, a seemingly shorter, but not always applicable, workaround

In general, people will reach for a simple `+ 'lt` kind of bound (rather than the more cumbersome `+ InfectedBy<'lt>`).

But if you have understood [the meaning of `+ 'usability` correctly](TODO), you should realize how this only works for simple cases:

```rs
/// ✅
fn simple_case<'lt>(lt: &'lt ())
  -> impl 'lt + Sized
{
    lt
}

/// ❌
fn does_not_work<'a, 'b>(a: &'a (), b: &'b ())
  -> impl 'a + 'b + Sized
{
    (a, b)
}
```

<details><summary>Error message</summary>

```rs
error: lifetime may not live long enough
  --> src/lib.rs:21:5
   |
18 | fn does_not_work<'a, 'b>(a: &'a (), b: &'b ())
   |                  --  -- lifetime `'b` defined here
   |                  |
   |                  lifetime `'a` defined here
...
21 |     (a, b)
   |     ^^^^^^ function was supposed to return data with lifetime `'b` but it is returning data with lifetime `'a`
   |
   = help: consider adding the following bound: `'a: 'b`

error: lifetime may not live long enough
  --> src/lib.rs:21:5
   |
18 | fn does_not_work<'a, 'b>(a: &'a (), b: &'b ())
   |                  --  -- lifetime `'b` defined here
   |                  |
   |                  lifetime `'a` defined here
...
21 |     (a, b)
   |     ^^^^^^ function was supposed to return data with lifetime `'a` but it is returning data with lifetime `'b`
   |
   = help: consider adding the following bound: `'b: 'a`

help: `'a` and `'b` must be the same: replace one with the other
```

</details>

Indeed, `+ 'usability` means that it has to be usable (at least) within the whole `'usability` region!

So if you write `+ 'a + 'b` it means that it will be usable within `'a` _and_[^and] that it will be usable within `'b` as well. That is, (an instance of (type)) `impl 'a + 'b` has to be usable within the _union_ of the regions `'a` and `'b`.

[^and]: that's exactly what `+` means in Rust: `+` adds capabilities, rather than restraining them, so there is no way `+ 'a + 'b` yields, _to the caller_, something more restrictive than just `+ 'a`.

But the `(a, b): (&'a (), &'b ())` value we are returning is meant to be unusable whenever:

  - we leave the `'a` region (because of its `a` component),
  - we leave the `'b` region (because of its `b` component).

Which means `(a, b)` is only usable within the _intersection_ of the `'a` and `'b` regions.

  - In expiry dates parlance, imagine a small bottle of milk, `a`, with an expiry date of one month, and another such bottle, `b`, but this time with an expiry date of one week. Say now that you mix the contents of both into a big bottle (this would be our `(a, b)` Rust item). Then the resulting mix should be drunk or thrown away within the week, lest the `b` part inside it go sour. That is, the resulting mix has an expiry date of `'b = min('a, 'b) = intersection('a, 'b)`.

    An `impl 'a + 'b` bottle of milk would, on the other hand, have meant that it would have been drinkable both before next week, and before next month, so effectively before next month (`'a = max('a, 'b) = union('a, 'b)`).

What we want to say is precisely that `(a, b)` is infected by both `'a` and `'b`, _i.e._, that it is `InfectedBy<'a> + InfectedBy<'b>`:

```rs
fn example<'a, 'b>(
    a: &'a str,
    b: &'b str,
) -> impl Fn(bool) + InfectedBy<'a> + InfectedBy<'b>
{
    move /* (a, b) */ |choice| {
        println!("{}", if choice { a } else { b })
    }
}
```

Granted, the previous function can be simplified:

```rs
fn example<'intersection_of_a_and_b>(
    a: &'intersection_of_a_and_b str,
    b: &'intersection_of_a_and_b str,
) -> impl 'intersection_of_a_and_b + Fn(bool)
```

  - since `'a` and `'b`, from the call-sites, would be able to **shrink down** to some `'intersection_of_a_and_b` region smaller than both, which would bring us back to a single-lifetime scenario, and thus to using the convenient `+ 'usability` syntax to get the returned existential to be infected by `'intersection_of_a_and_b`.

But in more complex APIs, lifetimes may not be able to shrink, and a proper solution needs to be used if you are to make it work.

For instance, no matter how hard you try, there is no simpler signature for the following function:

```rs
fn impl_<'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> impl Fn(bool) + InfectedBy<'a> + InfectedBy<'b>
{
    move /* (a, b) */ |choice| {
        println!("{}", if choice {
            *a.lock().unwrap()
        } else {
            *b.lock().unwrap()
        });
    }
}
```

### Comparison to `dyn Trait`

Note that `dyn Trait` is _actual_ type erasure, so if you manage to get your hands on the `'intersection` lifetime of all the captured lifetime and type parameters, then you can simply use `+ 'intersection` and Rust won't complain even when bigger unshrinkable lifetimes are part of the before-erasure type (in other words, type erasure is kind of able to shrink even otherwise unshrinkable lifetimes):

```rs
fn dyn_<'i, 'a : 'i, 'b : 'i>( // `intersection('a, 'b) ⊇ 'i`.
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> Box<dyn 'i + Fn(bool)>
{
    Box::new(move /* (a, b) */ |choice| {
        println!("{}", if choice {
            *a.lock().unwrap()
        } else {
            *b.lock().unwrap()
        });
    })
}
```

  - [Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=21498ea39cc6e85a074142c45c81e5d5)

The keen eye may have spotted the ["intersection lifetime" pattern][1]. Otherwise, that newly introduced `'i` "lifetime" parameter will probably seem weird or confusing, in which case you may want to check out [the dedicated section][1].

[1]: ./intersection-lifetime.md
