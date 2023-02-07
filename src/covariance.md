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

This is called a _subtyping relation_, and you may observe that it is quite similar to an implicit coercion:

```rs
fn implicit_coercion(r: &String)
  -> &str
{
    r /* as _ */
}
```

  - See [the dedicated article](./subtyping-vs-coercions.md) to know the difference between the two.

### Is shrinking lifetimes really "silly"?

<details open><summary>Click to skip this section</summary>

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

Imagine trying to call `pick()` on a `&String::from("…")` and a `"static str"`: with this signature, the borrows are constrained to have equal lifetimes! Covariance to the rescue 🙂

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

</details>

### Other examples of "types with shrinkables lifetimes"
