Until now I've been talking of covariant / shrinkable lifetimes, or lack thereof (non-covariant / unshrinkable lifetimes).

But it turns out that, as surprising as it may initially seem, there are some specific types for which the lifetime parameter, rather than shrink, can actually _grow_.

# Contravariance: when lifetimes can grow

The only time a lifetime is allowed to grow is when it occurs in **function argument position**:

 1. ```rs
    type MyCb = fn(String);
    ```

    which we have to make generic (reminder: variance is a property of generic types exclusively):

 1. ```rs
    type MyGenericCb<Arg> = fn(Arg);
    // or:
    type MyLtGenericCb<'lt> = fn(&'lt str);
    ```

In this case, we'll say that `MyGenericCb<Arg>` is contravariant in `Arg` ⇔ Lifetimes occurring in `Arg` are allowed to grow ⇔ `'lt` is allowed to grow in `MyGenericCb<&'lt str> = fn(&'lt str) = MyLtGenericCb<'lt>` ⇔ `MyLtGenericCb` is contravariant.

### Growing lifetimes???

To get an intuition as to why/how can this case of growing lifetimes be fine, consider:

```rs
type Egg<'expiry> = &'expiry str; // or smth else covariant.
struct Basket<'expiry>(Vec<Egg<'expiry>>);

impl<'expiry> Basket<'expiry> {
    fn stuff(
        egg: Egg<'expiry>,
    )
    {
        let mut basket: Self = Basket::<'expiry>(vec![egg]);
        /* things with basket: Self */
        drop(basket);
    }
}
```

Now, imagine wanting to work with a `Basket<'next_week>`, but only having an `Egg<'next_month>` to construct it:

```rs
fn is_this_fine<'next_week, 'next_month : 'next_week>(
    egg: Egg<'next_month>,
)
{
    let stuff: fn(Egg<'next_week>) = <Basket<'next_week>>::stuff;
    stuff(egg) // <- is this fine?
}
```

We have two dual but equivalent points of view that make this right:

 1. `Egg<'expiry>` is covariant in `'expiry`, so we can shrink the lifetime inside `egg` when feeding it: "we can shrink the lifetimes of covariant arguments right before they are fed to the function";

 1. at that point "we can directly make the function itself take arguments with bigger lifetimes directly"

    That is:

    1. Given some `'expiry` lifetime in scope (_e.g._, at the `impl` level):

        ```rs
        fn stuff(
            egg: Egg<'expiry>,
        )
        ```

    1. We could always shim-wrap it:

        ```rs
        fn cooler_stuff<'actual_expiry>(
            egg: Egg<'actual_expiry>,
        )
        where
            //             ≥
            'actual_expiry : 'expiry,
        {
            Self::stuff(
                // since `Egg<'actual_expiry> ➘ Egg<'expiry>`.
                egg // : Egg<'expiry>
            )
        }
        ```

    1. So at that point we may as well let the language do that (let `stuff` subtype `cooler_stuff`):

    ```rs
    // until_next_month ⊇   until_next_week
    //    'next_month   :        'next_week
    fn(Egg<'next_week>) ➘ fn(Egg<'next_month>)
    ```

That is:

```rs
fn(Egg<'short>) ➘ fn(Egg<'long>)
```
