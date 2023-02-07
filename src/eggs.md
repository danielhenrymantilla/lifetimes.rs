# Some eggsamples of variance

Say there is a typical half-a-dozen 🥚s in your fridge, with an expiry date of, say, next week. You go and eat one of them, using some eggcelent recipe, but which is not the point here (although you can probably guess what it _boils_ down to).

The point is, there are now only 5 🥚s left, and since there may be guests coming over to taste that delicious _omelette du fromage_ you are infamous for, you do need there to be 6 in the fridge in case these guests have the indecency of actually showing up to the invitation.

So you go buy that one 🥚. Since it is more recent, this one has an expiry date of, say, next month. Alas, there is no room for yet another egg basket, since each of these 🥚s takes already way too much volume in the fridge (these are ostrich 🥚s, after all). That's why you have to unwrap that new 🥚 if you want it to fit in your fridge.

So now you have that philosophical question: is it fine to put that ostrich 🥚 which must be eaten within the next month within a basket of 🥚s that must be eaten within the next week? ~~"I fully own these 🥚s, so covariance is fine!"— you may hear yourself chant. Or not.~~ Anyhow, you can guess the answer: yes, it's fine!

The one that wouldn't be fine would be if you decided to do it the other way around: use the _new_ one-month-expiring-labelled basket for the _old_ one-week-expiring 🥚s.

  - To clarify, since things might have gotten a bit too heggtic, the question here is about when it is fine to view an 🥚 with a certain expiry date as an 🥚 with a _different_ expiry date. In general, the intuition says that you can "always" be eggerly conservative / "pessimisitic" with an expiry date, reducing the span of edibility. Such span can shrink, and nobody will be eating spoiled 🥚s.

But there are cases where you can't reduce spans of edibility just because you fancy it.

Consider a variant (we have ostrich 🥚s laying around for some reason, we may as well use them!): imagine that the old one-week-expiry-labelled basket is empty, and that you have just acquired 5 fresh 1-month-edible 🥚s. The old, 1-week-expiring-labelled basket has gotten stuck in the frige, so you decide to use it for your new eggs, _with the intention of eating them within the month anyways_. You don't mind, because you _know_ your 🥚s will be edible for a month.

That doesn't sound too bad, right? So you leave the kitchen proud of your big brain play (which is true, relatively speaking, since ostriches have quite tiny brains: they're actually smaller than their eyes!).

The problem is that your roommate, Regginald, who had gone to work with one of the previous 1-week-expiring 🥚s, comes back from work, with the 🥚 unscathed (a show about ostrich wildlife had just made them realize that ostrich 🥚s weren't the most sustainable or ethical diet). So they **put back the 1-week-expiring 🥚 in the 1-week-expiring-labelled basket**, which is a perfectly sensible and _sound_ thing to do.

And that's how, two weeks later, you end up with a horrible episode of food poisoning.

Back to Rust, the idea here is that if you had been the only one with mutable access to that ill-labelled basket of 🥚s, then that operation of _temporarily shrinking the expiry date, to have it increase back afterwards_ would have been fine.

But since Regginald had mutable access to the ill-labelled basket, they _legitimately and soundly_ put a short-living 🥚 inside it. And you, oblivious to this fact, enlargened "back" the lifetime of not only the original things, but also of the now present short-living 🥚. And food poisoning ensued.

In Rust the mechanism to "change the way we view something" in a way that auto-reverts would be a borrow:

 1. You bought a `Basket<'long>`;
 1. By exposing something in the fridge, you are, in Rust parlance, offering access to the basket with a shortened expiry date (`Basket<'short>`), through a borrow:
      - either immutable (`&Basket<'short>`);
      - or mutable (`&mut Basket<'short>`).
 1. An innocent safe-Rust-API user works with that `&[mut] Basket<'short>`.
      - In the `&` case, this is fine.
 1. In the `&mut` case, they may/could insert an `Egg<'short>` inside it.
 1. When you recover things out of the `Fridge` / when that initial borrow ends, the `Basket<'long>` collection of `Egg<'long>`s actually has an `Egg<'short>` impostor among them 🤮

```rs
#![deny(unsafe_code)]

use dbg as eat;

type Basket<'expiry> = Vec<Egg<'expiry>>;
type Egg<'expiry> = &'expiry str;

#[allow(unsafe_code)] // The only unsafe operation happens here
fn shrink_lifetime<'r, 'short, 'long : 'short> (
    basket_in_fridge: &'r mut Basket<'long>,
) -> &'r mut Basket<'short>
{
    // Assuming the following to be sound:
    unsafe {
        ::core::mem::transmute(basket_in_fridge)
    }
}

/// Notice, here, how the only `unsafe` operation has been that of
fn open_fridge<'short>(
    fridge_user: impl FnOnce(&mut Basket<'short>),
) -> Basket<'static>
{
    // Our long-living eggs.
    let mut basket: Basket<'static> = vec!["long-living 🥚"];

    // The basket we should have put in the fridge:
    let basket_in_fridge: &mut Basket<'static> = &mut basket;

    // But we end up using, instead, the already present shortly-expiring
    // basket, with our new eggs.
    let basket_in_fridge: &mut Basket<'short> =
        shrink_lifetime(basket_in_fridge)
    ;

    // We go out of the kitchen for a few days
    fridge_user(basket_in_fridge);

    // We get back "our" long-living eggs.
    basket
}

fn food_poisoning() {
    // The short-lived egg Regginald went and came back with.
    let short_lived_egg = String::from("short-living 🥚");

    let basket = 'regginald: {
        // It can soundly be put back in the basket since it is a
        // shortly-expiring basket!
        open_fridge(|basket: &mut _| {
            // Look ma, no unsafe!
            basket.push(&short_lived_egg);
        })
    };

    // Say that two weeks pass by, so that the `short_lived_egg` has expired:
    drop(short_lived_egg);

    // And now we go back to the basket with "our" eggs and eat them.
    for egg in basket {
        eat!(egg); // Uh-oh
    }
}
```

___

#### Conclusion

I'd eggspect, or at least hope, this eggcelent aneggdote to have given you the following key _intuitions_:

 1. Don't say about an expired egg that it hasn't expired yet:

    > **Increasing/enlarging expiry-dates ❌ is almost always Not Right™**

 1. But you can always say that a good egg is about to expire even when it is not:

    > **Usually, we can shrink/shorten expiry-dates 👍 willy-nilly and it will Be Fine™**

 1. But beware of mutable borrows! Since borrows are temporary, whenever they end, the borrowee can be reüsed again with its original lifetime. So any lifetime shrinkage happening within the borrow will, in a way, be _reverted_ the moment the borrow ends. And reverting a lifetime shrinkage boils down to having enlarged a lifetime ⇒ `1.` ⇒ Bad™.

    > **Shrinking expiry-dates behind temporary mutable access —such as mutable borrows ⚠️— is Easily Problematic™**
