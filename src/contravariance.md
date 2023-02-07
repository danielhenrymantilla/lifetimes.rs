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

In this case, we'll say that `MyGenericCb` is `MyGenericCb<Arg>` is contravariant in `Arg` ⇔ Lifetimes occurring[^non_elided] in `Arg` are allowed to grow ⇔ `'lt` is allowed to grow in `type MyLtGenericCb<'lt> = fn(&'lt str)` ⇔ `MyLtGenericCb` is contravariant.

[^non_elided]: "non-elided ones": since we are dealing with function pointer types (`fn()`), remember that higher-rank lifetime parameters can occur.

### Growing lifetimes???

To get an intuition as to why/how can this case of growing lifetimes be fine, consider:

```rs
type Egg<'expiry> = &'expiry str; // or smth else covariant.
type Basket<'expiry> = Vec<Egg<'expiry>>;

impl<'expiry> Vec<Egg<'expiry>> {
    fn push(
        self: &'_ mut Basket<'expiry>,
        egg: Egg<'expiry>,
    )
```

Know, imagine having a `Basket<'next_week>`, and wanting to put an `Egg<'next_month>` inside it. We have two dual but equivalent points of view:

  - `Egg<'expiry>` is covariant in `'expiry`, so this is fine: "we can shrink the lifetimes of covariant arguments right before they are fed to the function";

  - at that point "we can directly make the function itself take arguments with bigger lifetimes directly"

That is:

 1. Given some `'expiry` lifetime in scope (_e.g._, at the `impl` level):

    ```rs
    fn push(
        egg: Egg<'expiry>,
    )
    ```

 1. We could always shim-wrap it:

    ```rs
    fn more_flexible_push<'actual_expiry>(
        egg: Egg<'actual_expiry>,
    )
    where
        //             ≥
        'actual_expiry : 'expiry,
    {
        Self::push(
            // since `Egg<'actual_expiry> <: Egg<'expiry>`.
            egg // : Egg<'expiry>
        )
    }
    ```

 1. So at that point we may as well let the language do that.
