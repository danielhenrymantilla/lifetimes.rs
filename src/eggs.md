# Some eggsamples of variance

In this case, let's go with eggs (🥚): say there is a typical half-a-dozen of them in your fridge, with an expiry date of, say, next week. You go and eat one of them, using some eggcelent recipe, but which is not the point here (although you can probably guess what it _boils_ down to).

The point is, there are now only 5 🥚s left, and since there may be guests coming over to taste that delicious _omelette du fromage_ you are infamous for, you do need there to be 6 in the fridge in case these guests have the indecency of actually showing up to the invitation.

So you go buy that one 🥚. Since it is more recent, this one has an expiry date of, say, next month. Alas, there is no room for yet another egg basket, since each of these 🥚s takes already way too much volume in the fridge (these are ostrich 🥚s, after all). That's why you have to unwrap that new 🥚 if you want it to fit in your fridge.

So now you have that philosophical question: is it fine to put that ostrich 🥚 which must be eaten within the next month within a basket of 🥚s that must be eaten within the next week? ~~"I fully own these 🥚s, so covariance is fine!"— you may hear yourself chant. Or not.~~ Anyhow, you can guess the answer: yes, it's fine!

The one that wouldn't be fine would be if you decided to do it the other way around: use the _new_ one-month-expiring-labelled basket for the _old_ one-week-expiring 🥚s.

  - To clarify, since things might have gotten a bit too heggtic, the question here is about when it is fine to view an 🥚 with a certain expiry date as an 🥚 with a _different_ expiry date. In general, the intuition says that you can "always" be eggerly conservative / "pessimisitic" with an expiry date, reducing the span of edibility. Such span can shrink, and nobody will be eating spoiled 🥚s.

But there are cases where you can't reduce spans of edibility just because you fancy it.

Consider a variant (we have ostrich 🥚s laying around for some reason, we may as well use them!): imagine that the old one-week-expiry-labelled basket is empty, and that have just acquired 5 fresh 1-month-edible 🥚s. The old, 1-week-expiring-labelled basket has gotten stuck in the frige, so you decide to use it for your new eggs, _with the intention of eating them within the month anyways_. You don't mind, because you _know_ your 🥚s will be edible for a month.

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
type Basket<'expiry> = Vec<Egg<'expiry>>;
type Egg<'expiry> = &'expiry str;

fn example() {
    let short_lived_egg = String::from("🥚");
    let reginald = |basket: &mut _| {
        basket.push(&short_lived_egg);
    };

    // you
    let basket_back = with_fridge(reginald);
    let last_egg = basket_back.pop().unwrap();
    drop(short_lived_egg); // `last_egg` now passed its expiry date.
    eat(last_egg); // Uh-oh
}

fn with_fridge<'short>(
    fridge_user: impl FnOnce(&mut Basket<'short>),
) -> Basket<'static>
{
    let mut basket: Basket<'static> = vec!["example"];
    let exposed_basket: &mut Basket<'static> = &mut basket;
    // Let's assume the following were sound:
    let exposed_basket: &mut Basket<'short> = unsafe {
        &mut *exposed_basket
    };
    fridge_user(exposed_basket);
    basket
}

fn eat<'not_expired_yet>(egg: Egg<'not_expired_yet>) {
   dbg!(egg);
}
```

So the rule of thumb of this whole aneggdote is that:
  - Usually, we can shrink/reduce expiry-dates willy-nilly and it will Be Fine™;
  - But there are also many cases involving a mutable-potentially-not-last
