# Variance rules: a recap

### Definition

Variance is a property of generic types:

  - Variance for a lifetime-generic type (_e.g._, `Formatter<'_>`) will determine whether **lifetimes** can:
      - shrink (covariant),
      - grow (contravariant),
      - or neither (invariant).

  - Variance for a type-generic type (`Vec<T>`) will determine:
      - **intuitively**,
        if that type parameter (`T`) were replaced with a lifetime-generic type itself (`type T<'r> = &'r str`), how the variance of that inner `T<'r>` type will propagate and affect the resulting variance of the composed type (`Vec<T<'r>>`).

        |                            ×                           |      `T<'lt>`<br>Co-variant      |    `T<'lt>`<br>Contra-variant    |    `T<'lt>`<br>In-variant    |
        |:------------------------------------------------------:|:--------------------------------:|:--------------------------------:|:----------------------------:|
        |       `F<T>` **cov**ariant<br>=<br>"passthrough"       |   **Cov**ariant<br>`F<T<'lt>>`   | **Contra**variant<br>`F<T<'lt>>` | **Inv**ariant<br>`F<T<'lt>>` |
        | `F<T>` **contra**variant<br><br>=<br>"flips it around" | **Contra**variant<br>`F<T<'lt>>` |   **Cov**ariant<br>`F<T<'lt>>`   | **Inv**ariant<br>`F<T<'lt>>` |
        |                  `F<T>` **inv**ariant                  |   **Inv**ariant<br>`F<T<'lt>>`   |   **Inv**ariant<br>`F<T<'lt>>`   | **Inv**ariant<br>`F<T<'lt>>` |

        Tip: if you see covariance as "being positive" (`+`), and contravariance as "being negative" (`-`), and invariance as "being zero" (`0`), these composition rules are the same as the sign multiplication rules!

        | × | +1 | -1 | 0 |
        |:-:|:-:|:-:|:-:|
        | +1 | +1 | -1 | 0 |
        | -1 | -1 | +1 | 0 |
        | 0 | 0 | 0 | 0 |

      - **formally**, how the subtyping relation among two choices for the type parameter results in some subtyping relation for the generic type.

        That is, if you have `T ➘ U` (`T` a subtype of `U`), and `type F<X>;`

          - if `F` is covariant, then `F<T> ➘ F<U>`;
          - if `F` is contravariant, then`F<U> ➘ F<T>` (reversed!);

        You can have `T ➘ U` by:

          - **common case**: having a variant `type T<'lt>`, and picking the lifetimes accordingly (this is the **intuitively** section just above);

          - <details><summary><b>niche case</b>: having a higher-rank <code>fn</code> pointer type</summary>

            and certain concrete choices of the inner generic lifetime:

            ```rs
            //! Pseudo-code:
            //! using `fn<'any>(…)` instead of `for<'any> fn(…)`

            // From higher-order lifetime to a fixed choice:
            fn<'any>(&'any str) ➘ fn(&'fixed str)

            // From a covariant lifetime to a higher-order one:
            fn(fn(&'static str)) ➘ fn<'any>(fn(&'any str))
            ```

            Interestingly enough, it does mean that if we pick:

              - ```rs
                type T = for<'any> fn(fn(&'any str));
                type U = fn(fn(&'static str));
                ```

            Then:
              - `T ≠ U`;
              - `T : 'static` and `U : 'static`;
              - `T ➘ U` by "fixing the higher-order lifetime"

                ⚠️ A `: 'static` bound is not sufficient to prevent subtyping shenanigans form happening! ⚠️
              - `U ➘ T` by "higher-ordering" a covariant lifetime by "induction from `'static`".
              - [Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=85bfd651072e498a469e0d40f4a0ad8c)

### Main variance examples to keep in mind

  - Mutable borrows and shared **mutability** wrappers are **invariant**.
      - `Mutex<T>`, `Cell<T>`, `&'cov Cell<T>`, `&'cov mut T` are invariant (in `T`);
  - otherwise, generally, **owned** stuff or **immutably-borrowed** stuff can be **covariant**.
      - `T`, `&'_ T`, `Box<T>`, `Arc<T>`, are covariant.
  - <code><b>fn</b>(CbArg)</code> is **contravariant** in `CbArg`.
      - But `fn(…) -> Ret` is **covariant** in `Ret`!
  - `impl Traits<'lt>` and `dyn Traits<'lt>` are **invariant**;
      - this includes the `Fn{,Mut,Once}` traits.
  - `+ 'usability` is **covariant**:
      - Bonus: there is also a no-overhead-upcasting/reünsizing _coercion_ possible from `&mut (dyn 'big + Traits)` to `&mut (dyn 'short + Traits)`, which to the untrained eye could appear as if `&mut (dyn 'lt + Traits)` were covariant in `'lt`, [which it is not](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=7778dcd83cbfa56c8d7d89f3bd04bb3d).

### Variance of "product types" / `struct`ural records / tuples

The rule of thumb is: combine them _restrictively_ / _non-covariance_ and _non-contravariance_ are infectious.

By that I mean that you could think of variance as of marker traits:

  - `trait Covariant {}`
  - `trait Contravariant {}`

```rs
struct Example<'lt> {
    x: &'lt str,            //  Covariant + !Contravariant
    y: Mutex<&'lt bool>,    // !Covariant + !Contravariant
}                           // !Covariant + !Contravariant = Invariant
```

For instance, in this `Example`, we have a non-contravariant field alongside an invariant, _i.e._, neither-covariant-nor-contravariant field.

Thus, the resulting `Example` can't be:
  - contravariant, due to either field;
  - covariant, due to the second field.

So we have a neither-covariant-nor-contravariant type, _i.e._, an invariant one.

####
