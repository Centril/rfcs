- Feature Name: `cfg_attr`
- Start Date: 2018-09-26
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

Permits conditional compilation with `cfg(attr(my_attribute))`
based on whether `my_attribute` is in scope or not.
For example, a user may write: `cfg(attr(optimize))`.

# Motivation
[motivation]: #motivation

## The problem

[rfc_2412]: https://github.com/rust-lang/rfcs/pull/2412

Consider a function:

```rust
fn foo(arg: Bar) -> Baz {
    // some logic...
}
```

You had written this function using Rust version `1.X`.
Then some months passed and version `1.{X + 3}` of Rust was released.
The new version included the attribute [`#[optimize(size)]`][rfc_2412]
which you wanted to take advantage of. However, you are now faced with a
dilemma - *"should I release a new major version using this new attribute
for improvements, or should I keep backwards compatibility due to the
*minimum supported rust version (MSRV)* and avoid a new major release?"*.

[rfc_2523]: https://github.com/rust-lang/rfcs/pull/2523
[rfc_2523_motivation]: https://github.com/Centril/rfcs/blob/rfc/cfg-path-version/text/0000-cfg-path-version.md#motivation

As noted in [RFC 2523][rfc_2523_motivation]:

> [stability_stagnation]: https://blog.rust-lang.org/2014/10/30/Stability.html
> [what_is_rust2018]: https://blog.rust-lang.org/2018/07/27/what-is-rust-2018.html
> 
> A core tenet of Rust's story is
> [*"stability without stagnation"*][stability_stagnation].
> We have made great strides sticking to this story while continuously
> improving the language and the community. This is especially the case with
> the coming [Rust 2018 edition][what_is_rust2018].

If we are to stray true to this goal, we should not be content with the status
quo and try to solve it so that crate authors may eat their cake and keep it too.

## An optimal solution

While [RFC 2523][rfc_2523] aims to solve the problem for new standard library
additions and language features in general the `cfg(version(1.31))` mechanism
is less than optimal for attributes specifically.

In particular, using `version(..)` is an indirect approach that is a proxy
for compilation that is conditional upon whether the attribute exists or not.

Testing with `version(..)` is also only useful as a proxy for built-in
attributes and can't be used to compile conditionally based on the
existence of  ecosystem-provided attributes such as `#[serde(..)]`.
By using an approach that works for all sorts of attributes,
it is possible to not only avoid major breakage due rasing the MSRV
but also public `[dependencies]` in your `Cargo.toml`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Using the example previously noted in the [motivation], we can now write (1):

```rust
#[cfg_attr(attr(optimize), optimize(size))]
fn foo(arg: Bar) -> Baz {
    // some logic...
}
```

The effect of `attr(optimize)` is to apply `optimize(size)` to `foo`
conditionally based upon whether the `optimize` attribute exists in scope or not.
If the attribute does not exist, because the compiler does not support it yet,
it will be as if you had instead written (2):

```rust
fn foo(arg: Bar) -> Baz {
    // some logic...
}
```

[rfc_2383_expect]: https://github.com/rust-lang/rfcs/blob/master/text/2383-lint-reasons.md#expect-lint-attribute

[RFC 2383][rfc_2383_expect] introduced the `#[expect(..)]` attribute.
This is yet another case in which `attr(..)` would be useful (3):

```rust
#[cfg_attr(
    attr(expect),
    expect(unused_mut, reason = "Everything is mut until I get this figured out")
)]
fn foo() -> usize {
    let mut a = Vec::new();
    a.len()
}
```

As with `optimize(..)`, `expect(..)` will only apply if the attribute
is actually is available and thus you may avoid having to raise MSRV
to make use of it.

Another example where this would have been historically useful is `must_use`.
Had we had support for `attr(..)` before `must_use` was available we could
have written (4):

```rust
#[cfg_attr(attr(must_use), must_use)]
fn double(x: i32) -> i32 {
    2 * x
}

fn main() {
    double(4);
    // warning: unused return value of `double` which must be used
    // ^--- This warning only happens if `must_use` exists.
}
```

We could have also used `attr` in the past to conditionally use a different
global allocator (5):

```rust
#[cfg(accessible(::std::alloc::System))]
#[cfg_attr(attr(global_allocator), global_allocator)]
static GLOBAL: std::alloc::System = std::alloc::System;

fn main() {
    let mut v = Vec::new();
    // This will allocate memory using the system allocator.
    // ^--- But only on Rust 1.28 and beyond!
    v.push(1);
}
```

Usage of `attr` is naturally not limited to `cfg_attr`. 
In some cases, such as with `#[used]` we might want to leave out an item
completely if the attribute does not exist. For example (6):

```rust
#[cfg(attr(used), used)] // will be present in the object file if the compiler supports it.
#[link_section = ".vector_table.exceptions"]
pub static EXCEPTIONS: [extern "C" fn(); 14] = [/* .. */];
```

If the compiler used does not support `#[used]` then `EXCEPTIONS` will not exist.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A rust compiler must provide a dynamic conditional compilation flag `attr(..)`
supported in `#[cfg($flag)]`, `#[cfg_attr($flag, $tts)]`, and `core::cfg!(..)`.

The grammar of the flag is:

```
attr_flag : "attr" "(" path ")" ;
```

Thus, it is for example valid to write `#[cfg(attr(foo::bar::my_attr))]`.

During conditional compilation, when checking whether a flag is active or not,
`attr($path)` will be considered active if and only if `$path` exists in the
attribute macro sub-namespace.

# Drawbacks
[drawbacks]: #drawbacks

## What if we never add more attributes?

One of the main drawbacks is if we stabilize `cfg(attr(..))` at a
time after which it is unlikely that we'll ever add more built-in attributes.
In such a scenario, the usefulness of `attr(..)` will be questionable and the
attribute could instead be considered technical debt. However, we believe
it is likely that more attributes will be added to the langauge because new
problems will arise. Furthermore, since `attr(..)` is useful for ecosystem
provided attributes, it would be unlikely that the flag would ever be
completely useless.

## Perhaps `version(..)` is enough?

Depending on how one feels about the suitability of `version(..)` as a
substitute `attr(..)` one could consider `attr(..)` to be redundant.
If `accessible(..)` is able to check the attribute macro sub-namespace,
one could similarly consider `attr(..)`.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Other approaches

## `version(1.30)`

As previously discussed in the [motivation] and in the [drawbacks] we noted
that `version(..)` is a less direct approach that is a proxy for the existence
of the attribute and that `attr(..)` is thus more appropriate.
However, a benefit of using `version(..)` could be that we limit complexity.

## `accessible(must_use)`

Another alternative that is perhaps more plausible is to just reuse
`accessible(..)` for the attribute macro sub-namespace as well.
You could then write `accessible(must_use)` and `accessible(::path::to_attribute)`.
However, thinking about `must_use` in terms of being accessible is probably
less natural than thinking about attributes separately.

## `rust_feature(must_use)`

[rfc_2523_feature]: https://github.com/Centril/rfcs/blob/rfc/cfg-path-version/text/0000-cfg-path-version.md#cfgrust_feature

As noted in [RFC 2523][rfc_2523_feature] another alternative would be to
conditionally compile based on whether a feature gate is active or not.

However, allowing code to detect the availability of specified feature gates
by name would require committing to stable names for these features,
and would require that those names refer to a fixed set of functionality.
This would require additional curation. However, as attribute names already
have to be standardized, `attr(..)` would not suffer the same problems wherefore
it may be the better solution.

## The bikeshed

As always, no RFC would be complete without a bikeshed about concrete syntax.
In the case of `attr(..)` there are other flag names which one could use instead:

### `has_attr(..)`

[rfc_2523_has_path]: https://github.com/Centril/rfcs/blob/rfc/cfg-path-version/text/0000-cfg-path-version.md#has_path

In this case, the reasoning is similar to that for `accessible(..)`
which opted not to use `has_path(..)` as the name.
This was discussed in [RFC 2523][rfc_2523_has_path].

One additional aspect which was left out of [RFC 2523][rfc_2523]
is that `has_attr(..)` would not be particularly internally
consistent with the naming of other attributes in Rust.
For example, we write `feature = ".."` and not `has_feature = ".."`.

### `attribute(..)`

One potential drawback of `attr(..)` is that `cfg(attr(..))` might look too
similar to `cfg_attr(..)` and could thus be confused with it; however, we
do not think that such confusion is likely.

# Prior art
[prior-art]: #prior-art

[clang_has_attribute]: https://clang.llvm.org/docs/LanguageExtensions.html#has-attribute

The main prior art is due to the `clang` compiler which provides
[__has_attribute][clang_has_attribute]. With it, you may write:

```cpp
#ifndef __has_attribute         // Optional of course.
  #define __has_attribute(x) 0  // Compatibility with non-clang compilers.
#endif

...
#if __has_attribute(always_inline)
#define ALWAYS_INLINE __attribute__((always_inline))
#else
#define ALWAYS_INLINE
#endif
...
```

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None as of yet.
