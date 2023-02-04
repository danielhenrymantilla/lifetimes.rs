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

For starters, let's try the typical works-way-more-often-than-it-deserves do-no-overcomplicate approach of using elided lifetimes everywhere:

```rs
fn pick(left: &'_ str, right: &'_ str)
  -> &'_ str
```

❌ this fails because ~~I wanted to make your life miserable~~ the [lifetime elision rules] do not handle this situation: the inputs are so symmetric there is no clear "borrowee" argument.

  - <b>👉 [lifetime elision rules] 👈</b>

[lifetime elision rules]: ./lifetime-elision-rules.md

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

But why is that? Well, remember how the input of our `pick()` function was _repeating_ the same `'ret` lifetime parameter for both function arguments. Well, that's exactly what repeating a lifetime parameter means: lifetime equality! If `p1` involves a `'static` lifetime, then it means that `p2`'s to-be-inferred lifetime has to be `'static`. Hence th error.

  - For the skeptical ones who may still think the lifetime of the return value may also be playing a role here (which it is not), and for the otherwise just curious people, here is an interesting thing: since having both function args involving the same lifetime parameter was enough to cause this restriction, we can actually get rid of the return type altogether (and thus the function body as well), and have the same problem! 😄

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

## When "lifetimes"/regions shrink

At this point things can get a bit overly abstract around this area; some people may start chanting some strange surely satanic incantation, which the most soulless text-to-speech machines out there interpret as this random dictionary word: variance.

Instead, let's go with food and my beloved expiry dates. In this case, let's go with eggs: there is a typical half-a-dozen of them in your fridge, with an expiry date of, say, next week. You go and eat one of them, using some eggcelent recipe, but which is not the point here (although you can probably guess what it _boils_ down to). The point is, there are now only 5 eggs left, and since there may be guests coming over to taste that delicious _omelette du fromage_ you are infamous for, you do need there to be 6 in the fridge in case these guests have the indecency of actually showing up to the invitation. So you go buy that one egg. Since it is more recent, this one has an expiry date of, say, next month. Alas, you have to unwrap the egg if you want it to fit in your fridge, since each of these eggs takes otherwise way too much volume in the fridge (these are ostrich eggs, after all).

So now you have that philosophical question: is it fine to put that ostrich egg which must be eaten within the next month within a basket of eggs that must be eaten within the next week? ~~"I fully own these eggs, so covariance is fine!"— you may hear yourself chant. Or not.~~ Anyhow, you get the point. The one that wouldn't be fine would be if you had eaten not 1, but 5 of these eggs (you can see how with such a start the story cannot end well), and were then considering putting the remaining egg in your new basket with the expiry date of a month.

  - To clarify, since things might have gotten a bit too heggtic, the idea is about when it is fine to view an egg with a certain expiry date as an egg with a _different_ expirty date. In general, the intuition says that you can always be eggerly conservative / "pessimisitic" with an expiry date, reducing the span of edibility. Such span can shrink, and nobody will be eating spoiled eggs.

Now let's consider a variant (we have ostrich eggs laying around for some reason, we may as well use them!): imagine that now all you have are 5 fresh 1-month-edible eggs, but you decide to unwrap them nonetheless into that old 1-week-expiry-date empty basket. You don't mind, because you _know_ your eggs will be edible for a month, so you can and will ignore this 1-week expiry date, and probably still be eating eggs from that basket two weeks from now. That doesn't sound too bad, right? So you leave the kitchen proud of your big brain play (which is true, relatively speaking, since ostriches have quite tiny brains–actually smaller than their eyes!).

The problem is that your roommate, Regginald, who had gone to work with one of the previous 1-week-expiring eggs, comes back from work, with the egg unscathed (a show about ostrich wildlife had just made them realize that ostrich eggs weren't a sustainable lifestyle). So they **put back the 1-week-expiring egg in the 1-week-expiring-labelled basket**, which is a perfectly sensible and _sound_ thing to do.

And that's how, two weeks later, you end up with a horrible episode of food poisoning.

Back to Rust, the idea here is that if you had been the only one with mutable access to that ill-labelled basket of eggs, then that operation of _temporarily shrinking the expiry date, to have it increase back afterwards_ would have been fine.

But since Regginald had mutable access to the ill-labelled basket, they _legitimately and soundly_ put a short-living egg inside it. And you, oblivious to this fact, enlargen the lifetime "back" of the original things, but also of the now present short-living egg.

In Rust the mechanism to "change the way we view something" in a way that auto-reverts would be a borrow:

 1. You bought a `Basket<'long>`;
 1. Through the fridge, you are offering access to the basket with a shortened expiry date (`Basket<'short>`), through a borrow:
      - either immutable (`&Basket<'short>`);
      - or mutable (`&mut Basket<'short>`).
 1. An innocent safe-Rust-API user works with that `&[mut] Basket<'short>`.
      - In the `&` case, this is fine.
 1. In the `&mut` case, they may/could insert an `Egg<'short>` inside it.
 1. When you recover things out of the `Fridge` / when that initial borrow ends, the `Basket<'long>` collection of `Egg<'long>`s you had actually has an `Egg<'short>` among them 💥

 ```rs
 type Egg<'expiry> = &'expiry str;
 type Basket<'expiry> = Vec<Egg<'expiry>>;

 fn eat<'not_expired_yet>(egg: Egg<'not_expired_yet>) {
    dbg!(egg);
 }

fn example<'short>(
    regginald: impl FnOnce(&mut Basket<'short>),
) -> Basket<'static>
{
    let mut basket: Basket<'static> = vec!["example"];
    let view: &mut Basket<'static> = &mut basket;
    regginald(unsafe { &mut *view });
    basket
}

let short_lived_egg = String::from("🥚");
let mut basket = example(|basket| {
    basket.push(&short_lived_egg)
});
let last_egg = basket.pop().unwrap();
drop(short_lived_egg); // `last_egg` now dangles.
eat(last_egg); // Uh-oh
```

So the rule of thumb of this whole aneggdote is that:
  - Usually, we can shrink/reduce expiry-dates willy-nilly and it will Be Fine™;
  - But there are also many cases involving a mutable-potentially-not-last

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
