# Intersection lifetime

Imagine wanting to write:

```rs
//! Pseudo-code

// this is what `#[async_trait]` wants to write, for instance.
fn async_fn<'a, 'b>(
    a: Arc<Mutex<&'a str>>,
    b: Arc<Mutex<&'b str>>,
) -> BoxFuture<'a ^ 'b, i32>
//                👆
//           pseudo-code for `intersection('a, 'b)`
```

  - See [the `async fn` unsugaring section](./async-fn.md) to know why.

Alas, we can't write that. Indeed, `'a ^ 'b` does not exist in current Rust (to keep things "simpler" I guess). So we have to resort to quantifications and set theory to express this property.

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

Back to Rust, this becomes:

```rs
'a ^ 'b = max { 'i where 'a : 'i, 'b : 'i }
```

And two point of views are possible:

  - either we realize we don't necessarily _need the max_ exactly, that something Big Enough™ would suffice. For instance, in our `dyn` example, we can realize that a subset of the actual intersection can suffice in practice (the subset where we actually use the `dyn`).

    In this case, we can focus on this freely introduced `I` and all will be good.

  - or we consider that a "free" extra generic parameter will be "min-maxed", as needed, by Rust's borrow checker, which is constantly trying to make our code pass/compile.

    With this more magical but simpler point of view, we just introduce `I` and let Rust auto-maximize it as needed into the actual full intersection of `A` and `B`: `A ∩ B`.

```rs
'a ^ 'b ≈ 'i where 'a : 'i, 'b : 'i
```

In both cases we end up with the following recipe for something like an intersection "lifetime"/region of regions `'r0`, …, `'rN`:

#### Recipe for `'i = 'r0 ^ … ^ 'rN`

 1. `<'i>` (let there be `I`)

    That is, introduce `'i` as a new/added "free" generic region parameter.

 2. Add

    ```rs
    where
        //  ⊇
        'r0 : 'i,
        'r1 : 'i,
        …
        'rN : 'i,
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

  - (in the specific case of an `async fn` or `Future`, it may be more sensible to rename `'i` as `'fut`, or `'async_fn`, or something along those lines)
