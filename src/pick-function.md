# Case study: the `pick()` function

In these series we'll be dealing with kind of the lifetime-equivalent of the trolley problem: `fn pick(&str, &str) -> &str` and friends.

The body will always be along the lines of:

```rust
fn pick(left: &'_?? str, right: &'_?? str)
  -> &'_?? str
{
    if ::rand::random() {
        left
    } else {
        right
    }
}
```

And the whole question will be about what should we put instead of each `'_??`.

#### Let's try elision?

For starters, let's try the typical works-way-more-often-than-it-deserves do-no-overcomplicate approach of using [elided lifetimes] everywhere:

```rs
fn pick(left: &'_ str, right: &'_ str)
  -> &'_ str
```

❌ this fails because ~~I wanted to make your life miserable~~ the [lifetime elision rules] do not handle this situation: the inputs are so symmetric there is no clear "borrowee" argument.

  - <b>👉 [lifetime elision rules] 👈</b>

[lifetime elision rules]: ./lifetime-elision-rules.md
[elided lifetimes]: ./elided-lifetimes.md

#### Let's try repeating

Now we could also try using distinct lifetime parameters for all three, but that will "never" work: we return a borrowing thing, so it has to be borrowing from at least one of the arguments, thus, **in practice, a borrowing return type almost always has to involve a lifetime parameter that appears among one of the input types**.

Given how symmetric our code is, the sensible thing to do now would be to repeat all the lifetimes:

```rs
fn pick<'ret>(s1: &'ret str, s2: &'ret str)
  -> &'ret str
```

And this does:
  - compile ✅
  - not result in unnecesarily restrictive API ✅

Yay! 🥳

  - Aside: beginners often forget about the latter. Massaging a signature so that it becomes compatible with a given implementation often means we are making that function _less compatible_ with certain call-site situations. While it can often be an acceptable loss, or a necessary one, other times that initial `cargo check` passing can hide that the API has become near-uncallable.

    That's why it is advisable to do this massaging of a function signature dance with an example usage / unit test next to it, so as to see the "two sides" of the caller/callee situation.

### Congratulations, puzzle solved. Now to the next puzzle!

Consider:

```rs
use ::std::sync::{Arc, Mutex};

#[derive(Clone)]
struct Person<'name> {
    name: Arc<Mutex<&'name str>>,
}

fn pick(p1: Person<'_??>, p2: Person<'_??>)
  -> &'_?? str
{
    if ::rand::random() {
        *p1.name.lock().unwrap()
    } else {
        *p2.name.lock().unwrap()
    }
}
```

and with the following test case:

```rs
#[test]
fn test()
{
    let local = String::from("non-static");

    let p1: Person<'static> = Person {
        name: Arc::new(Mutex::new("static")),
    };
    let p2: Person<'_> = Person {
        name: Arc::new(Mutex::new(&local)),
    };
    let choice = pick(p1, p2);
    dbg!(choice);
}
```

Now, if you try the `'ret`-everywhere approach, the test function will not compile!

```rust ,ignore
error[E0597]: `local` does not live long enough
  --> src/lib.rs:49:35
   |
45 |     let p1: Person<'static> = Person {
   |             --------------- type annotation requires that `local` be borrowed for `'static`
...
49 |         name: Arc::new(Mutex::new(&local)),
   |                                   ^^^^^^ borrowed value does not live long enough
...
53 | }
   | - `local` dropped here while still borrowed
```

From the error message, we can guess that Rust is now convinced that our `p2: Person<'_>` is actually a `Person<'static>` (at which point it complains about `&local` not being able to be borrowed for `'static` / `'forever` due to the impending doom that looms over the `local` variable).

But why is that? Well, remember how the input of our `pick()` function was _repeating_ the same `'ret` lifetime parameter for both function arguments. Well, that's exactly what repeating a lifetime parameter means: lifetime equality! If `p1` involves a `'static` lifetime, then it means that `p2`'s to-be-inferred lifetime has to be `'static`. Hence the error.

  - For the skeptical ones who may still think the lifetime of the return value may also be playing a role here (which it is not), and for the otherwise just curious people, here is an interesting thing: since having both function args involving the same lifetime parameter was enough to cause this restriction, we can actually get rid of the return type altogether (and thus the function body as well), and still have the same problem! 😄

    ```rs
    fn pick<'ret>(_: Person<'ret>, _: Person<'ret>)
    // -> no return type whatsoever
    {
        // no meaningful body either 🙃
    }
    ```

Now, at this point we may wonder about the initial `&str` case, and see if it also suffers from this problem (had we been overly optimistic?)

```rs
fn pick<'ret>(s1: &'ret str, s2: &'ret str)
  -> &'ret str
{
    "…"
}

#[test]
fn test()
{
    let local = String::from("non-static");

    let s1: &'static str = "static";
    let s2: &'_ str = &local;
    let choice = pick(s1, s2);
    dbg!(choice);
}
```

Aaand… it does compile!

So, what has changed? Why/how does `&'_ str` work but `Person<'_>` not?

And the answer is that the `'_` in `&'_ str` is "allowed to shrink", which is not the case for `Person<'_>`.

VARIANCE yadda yadda

Indeed, the main change here is that `Person<'_>`, contrary to `&'_ str`, does not allow its lifetime `'_` to "shrink"; which is something the `'ret`-everywhere approach was (probably unknowingly) relying on for the signature not to be restrictive. As

```rs
#[test]
fn test()
{
    let local1 = String::from("Jane");
    let local2 = String::from("Dove");

    let p1: Person<'_> = Person {
        name: Arc::new(Mutex::new(&local1)),
    };
    let p2: Person<'_> = Person {
        name: Arc::new(Mutex::new(&local2)),
    };
    let choice = pick(p1.clone(), p2.clone());
    dbg!(choice);
    if ::rand::random() {
        drop(p1); drop(local1);
        drop(p2); drop(local2);
    } else {
        drop(p2); drop(local2);
        drop(p1); drop(local1);
    }
}
```
