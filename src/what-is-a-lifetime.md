# What is a "lifetime"?

  - ⚠️ It does not necessarily match the life-time of a value or variable:

    ```rs
    {                                 // 0
      let mut s = String::from("42"); // 1
      let r = &s;                     // 2
      // stuff…                       // 3
      println!("{r}");                // 4
      mem::take(&mut s);              // 5
      drop(s);                        // 6
    }                                 // 7
    ```

      - Life-time of `s`: `1..6`
      - Life-time of `r`: `2..7`
        - "lifetime"/duration of the borrow of `s` held in `r`: `2..5`
        - `r` is "inert" / "zombie" during `5..7`: NLL!

    Note that the "lifetime"/duration of a borrow can ~~outlive~~ span beyond the life-time of the variable holding it:

    ```rs
    {                                              // 0
        let s: &'static str = "non moriturum sum"; // 1
    }                                              // 2
    ```

      - Life-time of `s`: `1..2`
      - "lifetime"/duration of the borrow of that `str`: `1..∞`

For the rest of this first section, I will thus be using:
  - "lifetime", to talk of what Rust denotes with `'tick`;
  - life-time, to talk about when some value / instance is alive (and pay thus special interest to it dying / being moved).

#### <span style="text-decoration: line-through;">life</span>times

So, what Rust calls "lifetimes" for historical reasons, is a quite inaccurate term to talk of "just times", we could say; that is (pick your point of view):
  - durations
  - regions/spans of code/time.

##### Bonus point of view: "lifetimes" as end dates (and thus, "expiry dates")

Interestingly enough, we can notice that the start point of these "time intervals" (_e.g._, the `1` in `1..2` or the `2` in `2..5`) is not really important: "lifetimes", ultimately, are used to know when some value ceases to be usable / when it may involve dangling pointers, since when a value is born, it is always valid / it doesn't dangle yet.

That is, when Rust looks at these lifetimes, it will mostly be thinking about `..2` _vs._ `..5` _vs._ _specific usage points_ (_e.g._, "what happens at line `3`").

So, we can really look at "lifetime" constraints as constraints having to deal with "end of region" instants, with usages of a "lifetime"-infected / end-date-infected having to occur before that end. In other words, **"lifetimes" as expiry dates**.

```rs
{                                   // 0
    let mut s = String::from("42"); // 1
    let r = &s;                     // 2
    // stuff…                       // 3
    println!("{r}");                // 4
    drop(s);                        // 5
}                                   // 6
```

 1. `s` expires at `5`.
 1. so the borrow of `s` held in `r` must have expired by then.
      - ⇒ Maximal expiry date of that borrow is thus `5`.
 1. Usage of `r` occurs before it: OK ✅

Notice how, even though "lifetimes" do not have to match life-times of variables or values, in practice these "expiry dates" constraints will stem from borrowing an eventually-destroyed object.

## Whence do lifetimes come

An actual lifetime either:

  - is `'static`, the `..∞` / right-open-ended time interval / never-ending duration (I personally like to name it `'forever`);

      - aside: similarly to "lifetime", "`'static`" itself is quite poorly/confusingly named, since it ultimately is an orthogonal concern to that of `static` storage.

  - or originates from a borrow: `&value` or `&mut value`.

In the latter case, it will thus be _upper-bounded_ by the life-time of the borrowed `value`:

> a non-`'static` "lifetime" is necessarily upper-bounded by some life-time.

  - Hence the name, I suppose?

___

Now, to better illustrate why talking of regions can be more useful than tunnel-visionning on the life-time upperbounds, let's consider increasingly complex `fn pick(&str, &str) -> &str` signatures.
