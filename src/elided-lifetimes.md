# "Lifetime placeholders", or elided lifetimes

A lifetime can be elided:
  - either implicitly, as is the case for `&str` or `&mut i32` (**behind a `&` always lurks a lifetime parameter**);
  - or explicitly, using the special `'_` syntax.

Across this book, I personally (overly) use the latter syntax, so as to show, clearly, all the elided lifetime parameters.

The result of an elided lifetime is thus **a lifetime "hole", or _lifetime placeholder_**, which will be the key thing to look at when thinking about **[lifetime elision rules]**. For instance, all these types involve lifetime holes:

  - `&i32`, `&'_ i32`, `&mut i32`, `&'_ mut i32`,
  - `GenericTy<'_>`[^elided_lifetimes_in_paths]
      - _e.g._, `Cow<'_, str>`, `Formatter<'_>`, `Context<'_>`, `BoxFuture<'_, R>`, `BorrowedFd<'_>`
  - `dyn '_ + Trait`, `dyn GenericTrait<'_>`, and ditto for `impl …`,
      - _e.g._, `Pin<Box<dyn '_ + Send + Future<Output = R>>>`

[^elided_lifetimes_in_paths]: Historically, it has been possible in Rust to use a lifetime-generic type without the `<'_>`. But this can be very confusing, which is why since the edition `2018_idioms`, there is a `elided_lifetimes_in_paths` lint which can be enabled (_e.g._, set to `warn`), to catch these things. **It is highly advisable to always set up this lint in any Rust project**.

See the [lifetime elision rules] for more info.

[lifetime elision rules]: ./lifetime-elision-rules.md
