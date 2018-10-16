- Feature Name: `pat_guard_pat`
- Start Date: 2018-10-15
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

[RFC 2294]: https://github.com/rust-lang/rfcs/blob/master/text/2294-if-let-guard.md

Subsuming [RFC 2294], the pattern language of Rust is extended such that
pattern guards, i.e. `Some(x) if condition(x)`, are turned into true patterns.

# Motivation
[motivation]: #motivation

## Smoothing out the language

[RFC 2535]: https://github.com/rust-lang/rfcs/pull/2535
[noted]: https://github.com/rust-lang/rfcs/blob/master/text/2535-or-patterns.md#reducing-complexity-with-uniformity

This RFC does not aim to introduce new syntax or even a new feature.
Instead, the goal is to smooth out the language, make it more compositional,
and thus reduce surprises for users.

As was [noted] in the now accepted [RFC 2535], which took the or-patterns
(e.g. `Ok(x) | Err(x)`) that already existed at the top level in the pattern
part of `match` arms, by reducing the number of special and corner cases in
the language, especially with respect to syntax, we can make the language
easier to understand, and thus reduce the overall complexity of Rust.
The removed complexity then goes back into our budget which we can use for
lofty goals such as providing and improving mechanisms for type safety.

The reduction in complexity is not just for users (which is arguably more
important). This RFC entails a reduction in technical grammatical complexity
in terms of BNF. We can see this by understanding that `match` expressions
have the form:

```rust
match $expr_scrutinee {
    $pat_0 (if $expr_if_0)? => $expr_0,
    ...
    $pat_n (if $expr_if_n)? => $expr_n,
}
```

Thus, `if $expr_if_i` is here a special construct and not part of the pattern
syntactically. With this RFC applied, `match` expressions instead have the form:

```rust
match $expr_scrutinee {
    $pat_0 => $expr_0,
    ...
    $pat_n => $expr_n,
}
```

By achieving this simplification, parsing Rust becomes simpler for tools such
as `nom` but also for `macro_rules!` macros.

## More local reasoning

A benefit of turning `pat if expr` into a pattern is that we may not write
things such as `Foo(Bar(x if condition(x))) => ...` instead of writing
`Foo(Bar(x)) if condition(x) => ...`.
While in this case the difference is not great, we have achieved a small
improvement in local reasoning by moving the guard directly to where the
binding `x` is introduced.

## Use cases

This RFC is not *just* about language unification.

[pcwalton_935]: https://github.com/rust-lang/rfcs/issues/935#issuecomment-77198835

In [#935 (comment)][pcwalton_935], [@pcwalton](https://github.com/pcwalton)
noted a need for pattern guards in `if let`, that is, writing:

```rust
if let Foo(x if x > 42) = bar() {
    ...
}
```

[eRFC 2497]: https://github.com/rust-lang/rfcs/pull/2497

While we could have written that instead as ([eRFC 2497]):

```rust
if let Foo(x) = bar() && x > 42 {
    ...
}
```

the first way of writing this reads better if the pattern `Foo(x)` is more complex.

Turning to use cases of the more real-world variety, [RFC 2294] introduced
`if let` guards into `match` expressions. We can improve upon the example
there and instead write:

```rust
#[inline]
fn csi_dispatch(&mut self, parms: &[i64], ims: &[u8], ignore: bool, x: char) {
    match (x, parms) {
        ('C', &[n]) => self.screen.move_x( n as _),
        ('D', &[n]) => self.screen.move_x(-n as _),
        _  if let Some(e) = erasure(x, parms) => self.screen.erase(e, false),
        ('m', &[] | &[0]) => *self.screen.def_attr_mut() = Attr {
            fg_code: 0, fg_rgb: [0xFF; 3],
            bg_code: 0, bg_rgb: [0x00; 3],
            flags: AttrFlags::empty()
        },
        ('m', &[n if n / 10 == 3]) if let Some(rgb) = color_for_code(n % 10, 0xFF)
          => self.screen.def_attr_mut().fg_rgb = rgb,
        _ => log_debug!(
            "Unknown CSI sequence: {:?}, {:?}, {:?}, {:?}",
            parms, ims, ignore, x
        ),
    }
}
```

As compared to RFC 2294, we have deduplicated the `_ => log_debug!(..)` fall back
arm and made the code use more short circuiting logic because we avoid computing
`color_for_code(...)` when `n / 10 == 3` doesn't hold.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The basic idea

Your employer sells lots of tasty cakes. You have been tasked with checking
if the customer wanted a lemon or a chocolate cake and if they do,
the customer will get a rebate. However, this only applies if the
customer wanted the chocolate to have a cacao above a certain threshold.

Given this description, you start typing up the types of cakes supported (1):

```rust
enum Cake {
    Lemon(...),
    Chocolate(u8),
    ...
}
```

Having already defined `enough_cacao` and the rebate logic,
you then move on to define the matching logic (2):

```rust
match cake {
    Cake::Lemon | Cake::Chocolate(percent if enough_cacao(precent))
      => special_rebate(cake),
    _ => no_rebate(cake),
}
```

and now you are done.

Notice here that we have written `percent if enough_cacao(percent)`.
This is because this RFC turns `pattern if condition` into a pattern itself
where before it was only allowed as a pattern guard in `match` arms.

Notice also that even though we have an or-pattern `p | q` here,
and where `q` introduces a binding `percent`, this is *not* an error.
We are allowed to get away with this since the expression to the right
of `=>` does not reference `percent`. If however the expression to the
right of `=>` does reference `percent`, an error is raised.

Before this RFC, we would have had to write (2) as (3):

```rust
match cake {
    Cake::Lemon
      => special_rebate(cake),
    Cake::Chocolate(percent) if enough_cacao(precent)
      => special_rebate(cake),
    _ => no_rebate(cake),
}
```

In (3), we repeat the call to `special_rebate(cake)` twice,
thus making the job of the reader and the optimizer more difficult,
whereas in example (2) we only had to write the call once.

You can also use `if let` guards per [RFC 2294] inside patterns.
For example, to check if the customer also wanted sorbet with the lemon cake,
we may write:

```rust
match cake {
    Cake::Lemon
        if let Some(Desert::Sorbet) = ask_for_anything_else()
    | Cake::Chocolate(percent if enough_cacao(precent))
      => special_rebate(cake),
    _ => no_rebate(cake),
}
```

Some time later, our requirements change such that deserts may have a flavour.
We now want to give rebates only if it was our special mix of sorbet or ice-cream.

```rust
match cake {
    Cake::Lemon if let Some(
        Desert::Sorbet(flavor) | Desert::Iceream(flavor) if flavor.is_special_mix()
        // We can use `flavor` here because it is initialized
        // in both sides of the or-pattern.
      ) = ask_for_anything_else()
    | Cake::Chocolate(percent if enough_cacao(precent))
      => special_rebate(cake),
    _ => no_rebate(cake),
}
```

The equivalent of this snippet without this RFC would have been:

```rust
match cake {
    Cake::Lemon => match ask_for_anything_else() {
        Some(Desert::Sorbet(flavor) | Desert::Iceream(flavor))
            if flavor.is_special_mix() => special_rebate(cake),
        _ => no_rebate(cake),
    },
    Cake::Chocolate(percent) if enough_cacao(precent)
      => special_rebate(cake),
    _ => no_rebate(cake),
}
```

## Further notes

Now that the basic gist of the RFC has been explained, there are some further
notes to keep in mind.

1. a pattern of form `$pat if $expr` or `$pat if let ...` is always refutable.
   This means that you need the `_ => ...` match arm to ensure that the match
   is exhaustive which all matches in Rust must be.

2. a pattern of form `p | q if ...` associates as `(p | q) if ...`.

   This must be the case because if we were to interpret it as `p | (q if ...)`
   then that would be a breakage as compared to the current rules.
   To get the other behaviour you can use parenthesis explicitly, i.e.
   `p | (q if ...)`.

   However, if the `if` guard is on the left hand side of the disjunction,
   i.e. `p if ... | q`, then we interpret this as `(p if ...) | q`
   and you don't need to use parenthesis to disambiguate.

3. these pattern guard patterns are permitted *anywhere* a pattern is.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Grammar



TODO

## Static semantics

TODO

## Dynamic semantics

TODO

# Drawbacks
[drawbacks]: #drawbacks

TODO

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future work
[future-work]: #future-work

TODO
