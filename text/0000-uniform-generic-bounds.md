- Feature Name: `uniform_generic_bounds`
- Start Date: 2018-09-26
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

Extends universal quantification (generics) in bounds with `for<..>` to
types and `const` values except for in `dyn` contexts.
As a consequence, closures can now be polymorphic on types and `const` values.
In other words, you can now write:

```rust
fn foo<F>(bar: F, baz: impl for<const N: usize> Fn([u8; N]))
where
    F: for<T> Fn(T) -> usize,
{
    bar(1u8) + bar(2u16);

    baz([1, 2]);
    baz([1, 2, 3])
}

foo(
    std::mem::size_of,
    |arr| println!("{:#?}", arr)
);
```

//TODO
Additionally, you may also now write `for<'a: 'b> `

# Motivation
[motivation]: #motivation

## Extending HRTBs to types improves understanding

## Use cases

### Generic closures

### Scrap your boilerplate


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Background
[background]: #background

[hrbt]: https://doc.rust-lang.org/nomicon/hrtb.html

First a bit of background.

Ever since the start with Rust 1.0, the language has had support for
what is sometimes referred to as *[higher-ranked trait bounds (HRTBs)][hrbt]*.
The feature is not commonly used directly by users,
but it is what allows you to write (1):

```rust
fn foo(bar: impl Fn(&u8) -> u8) {
    let x = 42;
    let y = bar(&x);
}
```

One might think that this is equivalent to (2):

```rust
fn foo<'outer>(bar: impl Fn(&'outer u8) -> u8) {
    let x = 42;
    let y = bar(&x);
}
```

However, if we try to compile (2), `rustc` won't be too happy with it and will
suggest to you that `&x` does not live long enough (3):

```rust
error[E0597]: `x` does not live long enough
 --> src/lib.rs:3:18
  |
3 |     let y = bar(&x);
  |                  ^ borrowed value does not live long enough
4 | }
  | - borrowed value only lives until here
  |
note: borrowed value must be valid for the lifetime 'outer
      as defined on the function body at 1:8...
 --> src/lib.rs:1:8
  |
1 | fn foo<'outer>(bar: impl Fn(&'outer u8) -> u8) {
  |        ^^^^^^
```

As you can see from the error message in (3) `&x` does not live during `'outer`
and so if we let it live for that long we might get undefined behavior.
Forunately, the compiler did not allow that to happen.

However, the question now becomes: - *"what is `?` in `Fn(&'? u8) -> u8`?"*.
This is where HRTBs come in. When a Rust compiler sees a definition as in (1),
it will understand it as (4):

```rust
fn foo(bar: impl for<'inner> Fn(&'inner u8) -> u8) {
    let x = 42;
    let y = bar(&x);
}
```

What `for<'inner> ... &'a inner u8` means is that `bar` works for *any* choice
of lifetime `'inner`. In other words, it doesn't matter what lifetime you choose.
By allowing any lifetime it works just fine to pass `&x` to `bar` since `foo`
can pick the lifetime instead of the caller of `foo` picking it.

[lifetime elision]: https://doc.rust-lang.org/nomicon/lifetime-elision.html

If you think about it, the desugaring in (4) is exactly the same as for
*[lifetime elision]* in general. However, when elided lifetimes are expanded
for normal functions there is something to "hang" the lifetime on
(the parameters of the function). For example, if you write (5):

```rust
fn quux(x: &u8) -> (&u8, &u8) { .. }
```

The compiler will desugar (5) to (6):

```rust
fn quux<'a>(x: &'a u8) -> (&'a u8, &'a u8) { .. }
```

In the case of (4) however, we need to add `for<..>` to hang the `'inner` on.

[rank-1 polymorphism]: https://en.wikipedia.org/wiki/Parametric_polymorphism#Rank-1_(prenex)_polymorphism

[rank-2 polymorphism]: https://en.wikipedia.org/wiki/Parametric_polymorphism#Rank-n_(%22higher-rank%22)_polymorphism

You might be wondering where the phrase "higher rank" comes from.
To understand that, let's consider what type the compiler would
assign to `quux` in (5) and to `foo` (4).
For `quux` the type would be `for<'a> fn(&'a u8) -> (&'a u8, &'a u8)`.
This is an example of *[rank-1 polymorphism]* or *"prenex polymorphism"*.
Meanwhile, the type of `foo` would be `for<F: for<'a> Fn(&'a) -> u8> fn(F) -> ()`.
This is an example of *[rank-2 polymorphism]*. We can extend this arbitrarily
in which case we get *rank-n polymorphism*.

Rust not only has higher-ranked *trait bounds*.
The language also has higher-ranked *types* which are a different concept.
In this case, there is a single instance where those exist in the type system.
That single case consists of function pointers. When you write (7):

```rust
fn foo(bar: fn(&u8) -> u8) -> u8 {
    let x = bar(&42);
    bar(&x)
}
```

this is the same as (8):

```rust
fn foo(bar: for<'inner> fn(&'inner u8) -> u8) -> u8 {
    let x = bar(&42);
    bar(&x)
}
```

However, in this case, there is no bound to talk of.
This RFC proposes no extension to higher ranked types and only deals with bounds.

## Overview of the proposal

Henceforth in this RFC, to use a more approchable,
and less jargon-packed terminology,
we will refer to a higher-rank trait bound as a *"generic bound"*.
What we've seen so far and what Rust currently supports are
*lifetime-generic* bounds (LGBs).

This RFC aims to:

[type variables]: https://en.wikipedia.org/wiki/Type_variable
[binders]: https://en.wikipedia.org/wiki/Free_variables_and_bound_variables
[outlives requirements]: https://github.com/rust-lang/rfcs/blob/master/text/2093-infer-outlives.md#motivation
[const-generic]: https://github.com/rust-lang/rfcs/pull/2000
[implied bounds]: https://github.com/rust-lang/rfcs/pull/2089

1. allow LGBs to optionally contain explicit *[outlives requirements]*
   in the form `F: for<'b: 'a> Fn(Thing<'b>)`.

2. introduce *type-generic* bounds (TGBs) of the form `F: for<T> Fn(Vec<T>)`.
   The *[type variables]* introduced in these `for<T>` *[binders]* may
   optionally be bounded such as with `for<T: Debug>`.

3. introduce *[const-generic]* bounds (CGBs) of the form
   `F: for<const N: usize> Fn([u8; N])`.

## [Outlives requirements] in lifetime-generic bounds

Consider the following, currently valid, program:

```rust
struct Foo<'x, 'y, 'z>
where
    'y: 'x,
    'z: 'y,
{
    field: &'x &'y &'z u8
}

fn foo<'a, F>(bar: F)
where
    F: for<'b, 'c> Fn(Foo<'a, 'b, 'c>)
{
}
```

In the definition of `Foo` we have specified the
outlives requirements `'y: 'x` and `'z: 'y`.
Thus, for `Foo<'x, 'y, 'z>` to be a valid (well-formed) type, these requirements
must hold for any lifetimes we substitute for `'x`, `'y`, and `'z`.

However, in the `where` *constraint* `F: for<'b, 'c> Fn(Foo<'a, 'b, 'c>)`
we have specified no such requirement that `'b: 'a` and `'c: 'b` but yet the
program above compiles. How is this possible - should it not result in an error?
It turns out that the compiler has inferred these requirements for us because
they are *[implied bounds]* from the definition of `Foo`. If it did not,
the type system would instead be unsound.

If we today try to annotate `for<'b, 'c>` with the inferred requirements,
that is: write `for<'b: 'a, 'c: 'b>`, the compiler is unhappy and emits an error:

```
error: lifetime bounds cannot be used in this context
```

Here, the aim of the RFC is to change this so that `for<'b: 'a, 'c: 'b>`
becomes legal and so the error message will no longer be emitted when using
`for<..>` this way. When you write a requirement such as `'b: 'a`, the lifetime
`'a` must as always be in scope. The construct `for<..>` is no different in this
respect. However, in this RFC, we provide no means to encode the reverse
requirement that `'a: 'b` when writing `for<'b> ...`.

The lifted restriction does not only apply in `fn` contexts but generally
anywhere `for<..>` can occur. This includes `where` clauses, in bounds applied
to `dyn` and `impl`, and anywhere type parameter lists may occur which includes
`impl`, `fn`, `struct`, `enum`, `union`, and `type`.

## Type-generic bounds

Suppose that you want to abstract over a function that is able to `Debug`
different types and present it in *some* fashion.
How would we go about doing that?
We could try to write something like:

```rust
fn debug_a_few_things_1<T, F>(debug: F)
where
    T: fmt::Debug,
    F: Fn(T)
{
    debug(42u8);
}
```

The compiler doesn't like this because it is the caller of `debug_a_few_things`
that gets to pick what the type `T` is, the function itself.
Therefore, our program is rejected with the following message:

```rust
error[E0308]: mismatched types
 --> src/lib.rs:7:11
  |
7 |     debug(42u8);
  |           ^^^^ expected type parameter, found u8
  |
  = note: expected type `T`
             found type `u8`
```

Let's now try to remove `T` and write the following instead:

```rust
fn debug_a_few_things_2<F>(debug: F)
where
    F: Fn(T)
{
    ...

    debug(42u8);
    debug("hello");

    ...
}
```

Here, we have seeminly invented `T` out of nowhere.
Even worse, we have no way to say that `T: fmt::Debug` anymore.
The compiler isn't too happy and tells us that it:

```rust
error[E0412]: cannot find type `T` in this scope
 --> src/lib.rs:3:11
  |
3 |     F: Fn(T)
```

What we could do in this situation is write:

```rust
trait DebugFn {
    fn debug<T: fmt::Debug>(message: T);
}

// The implementations...

fn debug_a_few_things_3<F>(debug: impl DebugFn)
where
    F: DebugFn
{
    ...

    debug.debug(42u8);
    debug.debug("hello");

    ...
}
```

However, in this case, we had to invent a new trait and we can't make use of
closures or other `fn` definitions. The solution is not particularly ergonomic.

So how do we solve this instead? The same way we solved the issues with
lifetimes that we discussed in the [background]. The trick here is to ensure
that `F` works with *any* type `T` that `debug_a_few_things_2` wants to use
within certain bounds. In current Rust, there is no way to formulate such a
condition. Thus, this RFC proposes such a mechanism. To do so, we write:

```rust
fn debug_a_few_things_4<F>(debug: F)
where
    F: for<T: fmt::Debug> Fn(T)
    // ------------------
    // We are saying that `F` may be passed any `T` sayisfying `fmt::Debug`.
{
    ...

    debug(42u8);
    debug("hello");

    if we_feel_inclined() {
        debug(true);
    }

    ...
}
```

What is a function that we can substitute for `F`? A polymorphic function!

```rust
fn send_messages_to_mars<T>(message: T)
where
    T: fmt::Debug
{
    let serialized = format!("Message to the red planet: {:#?}", message);
    locate_satelite()
        .and_then(|satelite| {
            satelite.send_message(serialized);
        })
        .unwrap();
}

fn main() {
    debug_a_few_things_4(send_messages_to_mars); // OK!
}
```

A polymorphic function with *weaker* requirements than `send_messages_to_mars`
can also be passed since it will accept all the types that `debug_a_few_things_4`
might send its way:

```rust
fn drop<T>(_: T) {}

fn main() {
    debug_a_few_things_4(drop); // OK!
}
```

However, a function that has *stronger* requirements can't be passed to
`debug_a_few_things_2` because it will not accept all possible types needed.
For example, we can't write:

```rust
fn debug_display<T: fmt::Debug + fmt::Display>(_: T) {}

fn main() {
    debug_a_few_things_4(debug_display);
    //                   -------------
    // ERROR! debug_display requires Display.
}
```

The distinction between weaker and stronger here has to do with what is
more / less polymorphic. TODO covariance / contravariance, positive / negative
position.

TODO notes about rank-3

### In `impl Trait`

The `impl Trait` shorthand in argument position also works here.
We could have instead written:

```rust
fn debug_a_few_things_4(debug: impl for<T: fmt::Debug> Fn(T)) {
    ...
}
```

It is also now possible to return a polymorphic object with the `impl Trait`
notation in return position. For example, while it is not particularly useful
in this case, we can return a family of identity functions like so:

```rust
fn id<T>(x: T) -> T { x }

fn identity() -> impl for<T> Fn(T) -> T { id }
```

### `for<T>` not permitted in `dyn Trait`

While we do currently permit constructs such as `Box<dyn for<'a> Fn(&'a u8)>`,
this RFC does not extend `for<T>` to `dyn Trait`. This means that a phrase
such as `Box<dyn for<T: fmt::Debug> Fn(T)>` won't be a legal type.
To understand why, consider a `trait` definition such as:

```rust
trait ObjectUnsafe {
    fn generic_method<T: fmt::Debug>(&self, arg: T);
}
```

If we then try to use `Box<dyn ObjectUnsafe>` as a type as in:

```rust
fn object_unsafe_safe(_: Box<dyn ObjectUnsafe>) {}
```

the compiler will refuse to compile the program and emit:

```rust
error[E0038]: the trait `ObjectUnsafe` cannot be made into an object
 --> src/lib.rs:L:C
  |
L | fn object_unsafe_safe(_: Box<dyn ObjectUnsafe>) {}
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `ObjectUnsafe` cannot be made into an object
  |
  = note: method `generic_method` has generic type parameters
```

As we can see, the compiler will not allow any trait with type parameters to
be used in `dyn Trait`. The reason why is because when the compiler attempts
to generate a vtable, it would have to contain generic function pointers,
which is impossible or it would have to possibly contain an unbounded number
of function pointers for each type the type parameter could be instantiatied at.

## Const-generic bounds

TODO

## Generic closures

TODO

### Inferred

TODO

### Explicit

TODO






Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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

- `dyn<'a>`?
- `impl<'a, T, ..>`?
- rank-3?
- `where` for reverse bounds?
- `impl Fn(impl Debug)`? meaning? where goes the binder?
