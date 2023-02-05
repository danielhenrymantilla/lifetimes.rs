# Intersection lifetime

Imagine we want to write:

```rs
//! Pseudo-code

// this is what `#[async_trait]` wants to write, for instance.
fn async_fn<'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> BoxFuture<'a ^ 'b, i32>
```

<details><summary>Why</summary>

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

As of `1.66.1`, in Rust, this cannot be written directly, since `async fn foo(…) -> Ret` is sugar for `fn foo(…) -> impl Future<Output = Ret>`, and `-> impl Trait` cannot be used in traits yet.

In the case of `Future`, which is a `dyn`-able trait, and which is `Pin<Box>`-transitive (_i.e._, given `T : ?Sized`, if `T : Future` then `Pin<Box<T>> : Future<Output = T::Output>`), a workaround for this limitation is thus to write the function as `-> Pin<Box<dyn Future…>>`.

And since the process is:

  - [x] tedious,
  - [x] quite automatable,
  - [x] and yet a bit subtle at times,

we end up with —you guessed it— macros to do this for us, such as [`#[async_trait]`](https://docs.rs/async-trait/0.1.64/async_trait/index.html).

But how does `#[async_trait]` do it?

  - Answering this question is more important than just intellectual curiosity. Indeed, what if:

      - we didn't have access to it;
      - we had to deal with a distinct trait or use case not covered by it;
      - or ran into [`async_trait` bugs/limitations](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=529ab9fcde6ea24f149a494560ba8ff5)

To anticipate any of these cases, it's actually very sensible to learn to do what `async_trait` does.

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

      - See [`InfectedBy<'_>`](./return-position-impl-trait.md)

 1. We let it `dyn`:

    ```diff
    ) -> Pin<Box<
    -       impl Future<Output = i32>
    +       dyn  Future<Output = i32>
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

      - We'll do `?Send` here, that is, we won't bother with `Send`-ness.

 1. We can't really use `+ InfectedBy<'_>` with `dyn` (and don't really need to, as we'll see below), so we get rid of that too:

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
      - _vice versa_: ❌
  - `dyn 'a + 'b + Future…` ?
      - Papering over the fact Rust doesn't let us write `dyn 'a + 'b + …` for some reason, this would not be right, since this is expresses a `'usability` that includes `'a` _and_  `'b`, and we've seen that each of this is already problematic, so a bigger usability is _a fortiori_ problematic.

I've talked about all this in more detail over the [section about `-> impl Trait`](./return-position-impl-trait.md), which also happens to mention the answer.

The gist of the answer is that:

> An `impl InfectedBy<'a> + InfectedBy<'b>` is only usable within the **intersection of `'a` and `'b`**.
>
> ```rs
> InfectedBy<'a, 'b> : ('a ^ 'b)
> ```

  - In other words, the "`'usability`" of such an entity is `('a ^ 'b)`

___

So we end up needing to write:

```rs
//! Pseudo-code

fn async_fn<'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> BoxFuture<'a ^ 'b, i32>
```

</details>

Alas, we can't write that. Indeed, `'a ^ 'b` (that is, the intersection of `'a` and `'b`) is not a thing expressible in Rust (to keep things simpler I guess). So we have to resort to quantifications and set theory to express this property.

Behold, what peak "simpler" looks like:

Let `A` and `B` be two sets, and `C`, their intersection: `C = A ∩ B`

Then, by definition, `A ⊇ C` and `B ⊇ C` (`C` is a subset of `A` and `C` is a subset of `B`).

In fact, one can define the intersection with that property:

 1. Consider some set `I` which is a subset of both `A` and `B`:

    `I where A ⊇ I and B ⊇ I`

      - visually, you can observe that `C ⊇ I`.

        <img width="245" alt="venn diagram" src="https://user-images.githubusercontent.com/9920355/216795708-dbf79c91-50ea-4ff8-920d-47aec22bcd20.png">

 1. Now imagine the maximum of all possible `I` / "try to make `I` bigger until you can't anymore": it's `C`!

```
A ∩ B = C = max { I where A ⊇ I and B ⊇ I }
```

Back to Rust, two point of views are possible:

  - either we realize we don't necessarily _need the max_ exactly, that something Big Enough™ would suffice. For instance, in our `dyn` example, we can realize that a subset of the actual intersection can suffice in practice (the subset where we actually use the `dyn`).

    In this case, we can focus on this freely introduced `I` and all will be good.

  - or we consider that a "free" extra generic parameter will be "min-maxed", as needed, by Rust's borrow checker, which is constantly trying to make our code pass/compile.

    With this more magical but simpler point of view, we just introduce `I` and let Rust auto-maximize it as needed into the actual full intersection of `A` and `B`: `A ∩ B`.

In both cases we end up with the following recipe for something like an intersection "lifetime"/region of regions `'r0`, …, `'rN`:

```rs
// we want:
let 'i = 'r0 ^ … ^ 'rN
```

 1. `<'i>` (let there be `I`)

    That is, introduce `'i` as a new/added "free" generic region parameter.

 2. Add

    ```rs
    where
        'r0 : 'i, // R0 ⊇ I
        'r1 : 'i, // R1 ⊇ I
        …
        'rN : 'i, // RN ⊇ I
    ```

    as `where` clauses so as to guarantee that `I` is one of the `{ I where R0 ⊇ I and … and RN ⊇ I }`.

 3. Use `'i` as if it were the desired intersection lifetime.

 4. Profit™

#### Example

We had:

```rs
fn async_fn<'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> BoxFuture<'a ^ 'b, i32>
```

We apply the recipe:

 1. ```rs
    //          👇
    fn async_fn<'i, 'a, 'b>(
    ```

 2. ```rs
    where
        'a : 'i,
        'b : 'i,
    ```

 3. ```diff
    - ) -> BoxFuture<'a ^ 'b, i32>
    + ) -> BoxFuture<'i, i32>
    ```

That is:

```rs
//          👇 (1)
fn async_fn<'i, 'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
//             👇 (3)
) -> BoxFuture<'i, i32>
where
    'a : 'i, // 👈 (2)
    'b : 'i, // 👈
```
