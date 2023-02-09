# `async fn` unsugaring / lifetime

Consider, for instance:

```rs
trait Trait {
    async fn async_fn<'a, 'b>(
        a: Arc<Mutex<&'a str>>,
        b: Arc<Mutex<&'b str>>,
    ) -> i32
    {
        dbg!(a.lock());
        dbg!(b.lock());
        42
    }
}
```

As of `1.66.1`, in Rust, this cannot be written directly, since
```rs
async fn f(…) -> Ret
```
is sugar for
```rs
fn f(…) -> impl Future<Output = Ret>
```
and `-> impl Trait` cannot be used in traits yet.

In the case of `Future`, which is a `dyn`-able trait, and which is `Pin<Box>`-transitive (_i.e._, given `T : ?Sized`, if `T : Future` then `Pin<Box<T>> : Future<Output = T::Output>`), a workaround for this limitation is thus to write the function as `-> Pin<Box<dyn Future…>>`.

And since the process is:

  - [x] tedious,
  - [x] quite automatable,
  - [x] and yet a bit subtle at times,

we end up with —you guessed it— macros to do this for us, such as [`#[async_trait]`][0]

[0]: https://docs.rs/async-trait/0.1.64/async_trait/index.html

But how does [`#[async_trait]`][0] do it?

  - Answering this question is more important than just intellectual curiosity. Indeed, what if:

      - we didn't have access to it;
      - we had to deal with a distinct trait or use case not covered by it;
      - or ran into [`#[async_trait]` bugs/limitations](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=529ab9fcde6ea24f149a494560ba8ff5)

To anticipate any of these cases, it's actually very sensible to learn to do what [`#[async_trait]`][0] does.

 1. So we start off

    ```rs
    trait Trait {
        async fn async_fn<'a, 'b>(
            a: Arc<Mutex<&'a str>>,
            b: Arc<Mutex<&'b str>>,
        ) -> i32
        {
            …
        }
    }
    ```

 1. We replace `async fn … -> Ret` with `fn … -> impl Future<Output = Ret>`:

    ```rs
    trait Trait {
        fn async_fn<'a, 'b>(
            a: Arc<Mutex<&'a str>>,
            b: Arc<Mutex<&'b str>>,
        ) -> impl Future<Output = i32>
                + InfectedBy<'a> + InfectedBy<'b>
        {
            async move {
                let _captures = (&a, &b);
                …
            }
        }
    }
    ```

      - See [`InfectedBy<'_>`][infected-by]

    [infected-by]: ./return-position-impl-trait.md

 1. We `Pin<Box>`-wrap it:

    ```rs
    trait Trait {
        fn async_fn<'a, 'b>(
            a: Arc<Mutex<&'a str>>,
            b: Arc<Mutex<&'b str>>,
        ) -> Pin<Box<
                impl Future<Output = i32>
                   + InfectedBy<'a> + InfectedBy<'b>
            >>
        {
            Box::pin(async move {
                let _captures = (&a, &b);
                …
            })
        }
    }
    ```

 1. We let it `dyn`:

    ```diff
    ) -> Pin<Box<
    -       impl Future<Output = i32>
    +        dyn Future<Output = i32>
               + InfectedBy<'a> + InfectedBy<'b>
        >>
        {
            Box::pin(async move {
                let _captures = (&a, &b);
                …
            })
        }
    }
    ```

      - We won't bother with `Send`-ness, here.

 1. We can't really use [`+ InfectedBy<'_>`][infected-by] with `dyn` (and don't really need to, as we'll see below), so we get rid of that too:

    ```diff
          dyn Future<Output = i32>
    -       + InfectedBy<'a> + InfectedBy<'b>
    +       + '???
    ```

At this point we've ended up with the following return type:

```rs
dyn '??? + Future<Output = i32>
```

And now the million dogecoin question is to know which lifetime we put here:

  - `dyn 'a + Future…` ?
      - May cover a region outside that of `'b`, so our future may dangle in there: ❌
  - `dyn 'b + Future…` ?
      - symmetrical situation: ❌
  - `dyn 'a + 'b + Future…` ?
      - Papering over the fact Rust doesn't let us write `dyn 'a + 'b + …` for some reason, this cannot be right, since this expresses a `'usability` that includes `'a` _and_  `'b`, and we've seen that each of these is already problematic, so a bigger usability will be just as problematic, if not more!

I've talked about all this in more detail over the [section about `-> impl Trait`](./return-position-impl-trait.md), which also happens to mention the answer.

The gist of the answer is that:

> An `impl InfectedBy<'a> + InfectedBy<'b>` is only usable within the **intersection of `'a` and `'b`**.
>
> ```rs
> InfectedBy<'a, 'b> : 'a ^ 'b
> ```

  - Where `'a ^ 'b` is pseudo-code syntax to express [the intersection](./intersection-lifetime.md) of `'a` and `'b`.

In other words, our `'???` / the "`'usability`" of such an entity is `'a ^ 'b`.

So we end up needing to write:

```rs
//! Pseudo-code

fn async_fn<'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> BoxFuture<'a ^ 'b, i32>
```

___

This, as the non-existing-in-Rust `'a ^ 'b` syntax suggests, is not real Rust code. See the [intersection lifetime section](./intersection-lifetime.md) to know more about how the above ends up becoming:

```rs
//! Real-code!

fn async_fn<'fut, 'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> BoxFuture<'fut, i32>
where
    // ⊇
    'a : 'fut, 'b : 'fut,
 // 'a ^ 'b ⊇ 'fut
```
