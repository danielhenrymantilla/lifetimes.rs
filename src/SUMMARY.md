# Summary

[Introduction](README.md)

- [What is a "lifetime"?]() <!-- what-is-a-lifetime.md) -->

# Core notions

- [Case study: the `pick()` function](pick-function.md)


- [Chapter 1]() <!-- ./chapter_1.md) -->
- [Bar]() <!-- ./bar.md) -->

# Reference

- [Lifetime elision rules](lifetime-elision-rules.md)

- [Variance](variance-intro.md)

    <!-- - [Some eggsamples of variance](eggs.md) -->

    - [Covariance: when lifetimes can shrink](covariance.md)

    - [Contravariance: when lifetimes can grow](contravariance.md)

    - [Variance rules: a recap](variance-rules.md)

- [Lifetime semantics of `-> impl Trait`](return-position-impl-trait.md)

- [Intersection lifetime](intersection-lifetime.md)

- [`async fn` unsugaring / lifetime](async-fn.md)

___


# Niche stuff

- [<code>impl\<#\[may_dangle\] …\> Drop</code>]()

___

- [Appendix](appendix.md)

    - [Elided lifetimes / what is `'_`](elided-lifetimes.md)

    - [`Trait + 'lifetime`](usability.md)

    - [Subtyping _vs._ Coercions](subtyping-vs-coercions.md)
___

[Closing thoughts]()
