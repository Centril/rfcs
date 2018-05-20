- Feature Name: `const_everywhere`
- Start Date: 2018-05-20
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

TODO

# Motivation
[motivation]: #motivation

TODO

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

[RFC 911]: https://github.com/rust-lang/rfcs/pull/911
[CTFE]: https://en.wikipedia.org/wiki/Compile_time_function_execution
[HOF]: https://en.wikipedia.org/wiki/Higher-order_function
[Robustness principle]: https://en.wikipedia.org/wiki/Robustness_principle
[RFC 1210]: https://github.com/rust-lang/rfcs/pull/1210
[inherent `impl`]: https://doc.rust-lang.org/reference/items/implementations.html#inherent-implementations

## Two restrictions modes: `const` and `?const`

As you might know, [RFC 911] introduced `const fn`s into the language.
As of yet, `const fn`s are not yet stable.

When given arguments that can be evaluated at compile time, `const fn`s can be
used in `const` items to perform [*Compile Time Function Execution (CTFE)*](CTFE).
However, `const fn`s are not limited to CTFE and can be executed at runtime as well.

We can think of normal functions, `fn`s, as having an *"impurity effect"*,
such that they can perform non-deterministic actions. Given this interpretation,
`const fn` is then a *negation* of this impurity effect.
That is, if we imagined that Rust had `io fn` instead of `fn`,
then `const fn` would be the same as `!io fn`.

Upon `const fn` and the `const` restriction in general,
this RFC extends the language with `?const fn` and the `?const` restriction.
_We read `?const` as **"may be const"**_ and it acts much like `?Sized`
(*"may be `Sized`"*) does but for the default impurity effect instead.

What `?const` means practically is that you can't assume that it is `const`,
but neither can you assume that it isn't `const` and use anything which is impure.
So you can only use other `?const` things or `const` things in a `?const` context.
However, you can't use a non-concrete `?const` thing inside a `const` context as
it isn't known whether or not it is `const` or not. **(TODO: CHECK THIS!)**

## Functions: `const fn` and `?const fn`

Let's take a look at a few examples of `fn`, `?const fn`, and `const fn`.

You can't perform side-effects in `?const fn` and `const fn`, but you can in `fn`.

```rust
fn foo() {
    println!("Hello world!"); // OK.
}

const fn foo() {
    println!("Hello world!"); // TYPE ERROR.
}

?const fn foo() {
    println!("Hello world!"); // TYPE ERROR.
}
```

You can assign the result of `const fn`, and `?const fn` - in some cases,
to `const` items.

```rust
fn foo() -> usize { 1 }
const ITEM: usize = foo(); // TYPE ERROR.

const fn foo() -> usize { 1 }
const ITEM: usize = foo(); // OK.

?const fn foo() -> usize { 1 }
const ITEM: usize = foo(); // OK. Nothing non-const involved, so this is fine.
```

## HoFs and pointers to `const fn` and `?const fn`

So far, we've seen no practical differences between `const` and `?const`.
They become apparent when we get to non-trivial examples.
One such example is when we introduce [*higher-order functions (HoFs)*][HOF].
Let's consider a simple HoF `twice_impure` which applies a function `fun`
to some argument `arg` twice:

```rust
fn twice<T>(arg: T, fun: fn(T) -> T) -> T {
    fun(fun(arg)) // OK.
}
```

So far, we've seen nothing new, as this is legal in today's Rust.

Let's try to introduce a version `const fn` of `twice` in today's Rust:

```rust
const fn pure_twice<T>(arg: T, fun: fn(T) -> T) -> T {
    fun(fun(arg)) // TYPE ERROR.
}
```

The compiler doesn't like this one bit and will error with the following message:

```rust
error[E0015]: calls in constant functions are limited to constant functions,
              tuple structs and tuple variants
```

And for good reason! Let's define a function `net_inc`:

```rust
fn net_inc(arg: usize) -> usize {
    arg + networked_function()
}
```

If the compiler permitted us to write:

```rust
const THREE: usize = pure_twice(1, net_inc);
```

we would have subverted the rules of CTFE by performing networked side-effect
at compile time thus making compilation behave non-deterministically.

If the compiler instead permitted:

```rust
let three = pure_twice(1, net_inc);
```

we would not subvert the rules of CTFE, but we would still perform side-effects
inside `const fn`. So the definition of `pure_twice` is no good!

To solve this and let us write `const fn` HoFs, this RFC introduces `const` and
`?const` function pointers. Given those type of function pointers, let's make
a version `twice` that is a `const fn` and one which is a `?const fn`:

```rust
const fn const_twice<T>(arg: T, fun: const fn(T) -> T) -> T {
    fun(fun(arg)) // OK. We know that `fun` respects the rules of `const`.
}

const fn const_twice_invalid<T>(arg: T, fun: ?const fn(T) -> T) -> T {
    fun(fun(arg)) // TYPE ERROR. `fun` could be a non-const function pointer.
}

?const fn maybe_const_twice_restrictive<T>(arg: T, fun: const fn(T) -> T) -> T {
    // This is overly restrictive, but `const` is allowed in `?const fn`.
    fun(fun(arg)) // OK.
}

?const fn maybe_const_twice<T>(arg: T, fun: ?const fn(T) -> T) -> T {
    fun(fun(arg)) // OK.
}
```

Let's now define three functions:

```rust
fn identity<T>(x: T) -> T { x }
const fn const_identity<T>(x: T) -> T { x }
?const fn maybe_const_identity<T>(x: T) -> T { x }
```

With these definitions in mind,
we take a look at how they interact with our HoFs above:

```rust
// We can coerce `const fn` and `?const fn` to `fn` in this case:
let a = twice(1, identity); // OK.
let b = twice(1, const_identity); // OK.
let c = twice(1, maybe_const_identity); // OK.

// We can't assign the result of `twice` to a `const` item:
const CA: u8 = twice(1, identity); // TYPE ERROR.
const CB: u8 = twice(1, const_identity); // TYPE ERROR.
const CC: u8 = twice(1, maybe_const_identity); // TYPE ERROR.


// `const_twice` won't accept an `fn` but `const fn` and `?const fn` is fine.
let d = const_twice(1, identity); // TYPE ERROR; identity is not const fn.
let e = const_twice(1, maybe_const_identity); // OK.
//  ^-- in this case, `maybe_const_identity` is concrete and all of its zero
//   -- dependencies can be 'const fn', wherefore the compiler accepts this.
let f = const_twice(1, const_identity); // OK.

// We can assign the OK results above to a `const` item.
const CD: u8 = const_twice(1, identity); // TYPE ERROR.
const CE: u8 = const_twice(1, const_identity); // OK.
const CF: u8 = const_twice(1, maybe_const_identity); // OK.


// Same as for `const_twice`:
let g = maybe_const_twice_restrictive(1, identity); // TYPE ERROR.
let h = maybe_const_twice_restrictive(1, const_identity); // OK.
let i = maybe_const_twice_restrictive(1, maybe_const_identity); // OK.

// Here too:
const CG: u8 = maybe_const_twice_restrictive(1, identity); // TYPE ERROR.
const CH: u8 = maybe_const_twice_restrictive(1, const_identity); // OK.
const CI: u8 = maybe_const_twice_restrictive(1, maybe_const_identity); // OK.


// `maybe_const_twice` will accept any of `fn`, `const fn`, and `?const fn`:
let j = maybe_const_twice(1, identity); // OK.
let k = maybe_const_twice(1, const_identity); // OK.
let l = maybe_const_twice(1, maybe_const_identity); // OK.

// `maybe_const_twice` will still accept `fn`, `const fn`, and `?const fn`.
const CJ: u8 = maybe_const_twice(1, identity); // TYPE ERROR.
//    ^-- However, the result of this function application is non-const!
const CK: u8 = maybe_const_twice(1, const_identity); // OK.
const CL: u8 = maybe_const_twice(1, maybe_const_identity); // OK.
```

Whew! This is a lot of cases; It is worth paying attention to cases
`d`, `g-i`, and `CJ`. We can also think of these cases in terms
of valid and invalid coercions between various types of function pointers:

```rust
struct A;

let p1: fn(A) -> A = identity; // OK.
let p2: fn(A) -> A = const_identity; // OK.
let p3: fn(A) -> A = maybe_const_identity; // OK.


let p4: ?const fn(A) -> A = identity; // TYPE ERROR.
let p5: ?const fn(A) -> A = const_identity; // OK.
let p6: ?const fn(A) -> A = maybe_const_identity; // OK.


let p7: const fn(A) -> A = identity; // TYPE ERROR.
let p8: const fn(A) -> A = const_identity; // OK.
let p9: const fn(A) -> A = maybe_const_identity; // OK.
```

Let's also consider how the HoFs themselves coerce:

```rust
// Coercions of an `fn` HoF:
let p10: fn(A, fn(A) -> A) -> A = twice; // OK.
let p11: fn(A, const fn(A) -> A) -> A = twice; // OK.
//             ^-- We can restrict in input / contravariant position or "negative polarity".
let p12: fn(A, ?const fn(A) -> A) -> A = twice; // OK.
//             ^-- Same here.
// We can't convert `twice` to any kind of `?const fn` or `const fn`
// since we can't restrict in output / variant position or "positive polarity".
let p13: ?const fn(A, fn(A) -> A) -> A = twice; // TYPE ERROR.
let p14: ?const fn(A, ?const fn(A) -> A) -> A = twice; // TYPE ERROR.
let p15: ?const fn(A, const fn(A) -> A) -> A = twice; // TYPE ERROR.
// Similarly, we can't coerce `twice` to `const fn(..) -> u8`.


// Coercions of a `const fn` HoF:
let p16: ?const fn(A, fn(A) -> A) -> A = const_twice; // TYPE ERROR.
//                    ^-- Can't loosen in input position.
let p17: ?const fn(A, ?const fn(A) -> A) -> A = const_twice; // TYPE ERROR.
//                    ^-- Same here, can't loosen!
let p18: ?const fn(A, const fn(A) -> A) -> A = const_twice; // OK.
//       ^-- We can loosen in output / variant position or "positive polarity"!
let p19: fn(A, fn(A) -> A) -> A = const_twice; // TYPE ERROR.
//             ^-- Can't loosen in input position.
let p20: fn(A, ?const fn(A) -> A) -> A = const_twice; // TYPE ERROR.
//             ^-- Same here, can't loosen!
let p21: fn(A, const fn(A) -> A) -> A = const_twice; // OK.
//       ^-- Same here; loosening to `fn` in outpout position is fine.


// Coercions of a `maybe_const_twice_restrictive`:
let p22: ?const fn(A, fn(A) -> A) -> A = maybe_const_twice_restrictive; // TYPE ERROR.
let p23: ?const fn(A, ?const fn(A) -> A) -> A = maybe_const_twice_restrictive; // TYPE ERROR.
let p24: fn(A, fn(A) -> A) -> A = maybe_const_twice_restrictive; // TYPE ERROR.
let p25: fn(A, ?const fn(A) -> A) -> A = maybe_const_twice_restrictive; // TYPE ERROR.
let p26: fn(A, const fn(A) -> A) -> A = maybe_const_twice_restrictive; // OK.


// Coercions of a `const?` HoF:
let p22: fn(A, fn(A) -> A) -> A = maybe_const_twice; // OK.
//             ^-- You can loosen in input position,
//       ^-- But doing so will also loosen in output position!
let p23: fn(A, ?const fn(A) -> A) -> A = maybe_const_twice; // OK.
let p24: fn(A, const fn(A) -> A) -> A = maybe_const_twice; // OK.
let p25: ?const fn(A, fn(A) -> A) -> A = maybe_const_twice; // TYPE ERROR.
let p26: const fn(A, fn(A) -> A) -> A = maybe_const_twice; // TYPE ERROR.
let p27: const fn(A, ?const fn(A) -> A) -> A = maybe_const_twice; // TYPE ERROR.
let p28: const fn(A, const fn(A) -> A) -> A = maybe_const_twice; // OK.
//       ^-- You can restrict in output position.
//                   ^-- But doing so will also restrict in input position.
```

### Help!? This is too much. I don't know what choice to make now...

We agree; the interactions specified above, and the presence of `fn`,
`?const fn`, and `const fn` makes it more difficult to decide which one to use.
However, we can take a page out of the [Robustness principle] which states:

> Be conservative in what you do, be liberal in what you accept from others.

In the case of whether to use `fn`, `?const fn`, and `const fn`, you should
be:
+ conservative in what you promise, so not `const fn`.
+ conservative in what you do, so not `fn`.
+ liberal in what you accept, so not `const fn`.

Put together, what you most likely want is `?const fn(A, ?const fn) -> A`, and
so the HoF should be `maybe_const_twice`. This type will be usable in both
CTFE and CTFE contexts, which helps you reuse code.

Of course, if you actually do need to perform side effects, or require that
something be `const`, such as in the example below, then you should use `fn`
and `const fn` respectively.

## Traits with `const fn` and `?const fn`

You can now put `const fn`s and `?const fn`s inside `trait` definitions.

When providing an implementation of such a trait, any `(?)const fn` in the given
trait must also be at least as restrictive in the implementation for the type.
This means that if the trait requires `?const fn`, you must provide a `?const fn`
or `const fn` in the implementation. If the trait requires `const fn`, you must
provide a `const fn` in the implementation.

Just as with normal `fn`s in traits, you can give provided definitions for
`const fn`s and `?const fn`s in the trait itself.

Whether provided or not, a `const fn` or `?const fn` in an implementation will
naturally be type checked under the rules of `const fn`s and `?const fn`s.

An example of using `const fn` and `?const fn` in a trait is:

```rust
trait Foo {
    const fn bar(baz: usize) -> usize;

    const fn quux() -> usize { 42 }

    ?const fn wibble() -> usize { 42 }
}

impl Foo for MyType {
    const fn bar(baz: usize) -> usize {
        let arr1 = [42; Self::quux()];   // OK.
        let arr2 = [42; Self::wibble()]; // TYPE ERROR.
        baz
    }
}
```

If the trait requires a `const fn` but the implementation provides an `fn` such
as in the example below, a Rust compiler will reject the implementation.

```rust
trait Foo {
    const fn bar() -> u8;
    const fn baz() -> u8;
    ?const fn quux() -> u8;
}

impl Foo for MyType {
    fn bar() -> u8 { 1 } // TYPE ERROR.
    ?const baz() -> u8 { 1 } // TYPE ERROR.
    fn quux() -> u8 { 1 } // TYPE ERROR.
}
```

## Restricting an `fn` as `const fn` and `?const fn` in implementations

Consider the `Default` trait:

```rust
pub trait Default {
    fn default() -> Self;
}
```

And and some type - for instance:

```rust
struct Foo(usize);
```

With this RFC, you may now write:

```rust
impl Default for Foo {
    const fn default() -> Self {
        Foo(0)
    }
}
```

Note that this `impl` has restricted `default` more than that required by the
`Default` trait.

We can of course also restrict `default` as a `?const fn` instead with:

```rust
impl Default for Foo {
    ?const fn default() -> Self {
        Foo(0)
    }
}
```

The restriction that `default` is a `const fn` or `?const fn` in this case allows
you to use `Foo::default()` within in a `const` item. In the case of `?const fn`,
you can do this because when `?const fn` is called with no non-`const` bounds
(see section on [const bounds][const-bounds]) or non-`const` `fn`s transitively,
then it is permitted in a `const` context.

```rust
const TOP_ITEM: Foo = Foo::default(); // OK.

trait Baz {
    const ASSOC_ITEM: Self;
}

impl Baz for Foo {
    const ASSOC_ITEM: Foo = Foo::default(); // OK.
}
```

## Syntactic sugar: `impl const` and `impl ?const`

In any [inherent `impl`], or an `impl` of a trait, you may now write:

```rust
impl const MyType {
    fn foo() -> usize { .. }

    fn bar(x: usize, y: usize) -> usize { .. }

    // ..
}

impl const MyTrait for MyType {
    fn baz() -> usize { .. }

    fn quux(x: usize, y: usize) -> usize { .. }

    // ..
}
```

and have the compiler desugar this for you into:

```rust
impl MyType {
    const fn foo() -> usize { .. }

    const fn bar(x: usize, y: usize) -> usize { .. }

    // ..
}

impl MyTrait for MyType {
    const fn baz() -> usize { .. }

    const fn quux(x: usize, y: usize) -> usize { .. }

    // ..
}
```

You can also write this way with `?const` so that:

```rust
impl ?const MyType {
    fn foo() -> usize { .. }

    fn bar(x: usize, y: usize) -> usize { .. }

    // ..
}

impl MyTrait for MyType {
    ?const fn baz() -> usize { .. }

    ?const fn quux(x: usize, y: usize) -> usize { .. }

    // ..
}
```

For the latter case of `impl const MyTrait for MyType`, when it type checks,
this always means that `MyType` may be used to substitute for `T` in a bound of
the form `T: const MyTrait`. This also applies to `impl ?const MyTrait for MyType`
and the bound form `T: ?const MyTrait`. But more on that later..

The compiler will of course now check that the `fn`s are `const`, and refuse to
compile your program if you lied to the compiler.

With respect to migrating existing code to this new model, assuming that you are 
comfortable with requiring a new compiler version for your library or application,
it is recommended that you simply start by adding `?const` before your type or
your trait in and before any trait bound **(TODO.. is `?const` inserted automatically?)**,
see if it still compiles and then continue this process until all `impl`s that
can be `?const` are. For those that can't, you can still add `?const fn` to some
`fn`s in the `impl`. The standard library will certainly follow this process in
trying to make the standard library as reusable as possible.

## Bounds: `T: const Trait` and `T: ?const Trait`
[const-bounds]: #bounds-T-const-Trait-and-T-?const-Trait

Similar to the function pointers we've discussed before, you may write
`T: const Trait` instead of `T: Trait` when defining the type variables
of your function, implementation, or in a `where` clause. As discussed in
the previous section, such a bound is satisfied by `impl const Trait for <type>`.
It is also satisfied by an implementation which only has `const fn`s in it.
**(TODO: DOUBLE CHECK SEMANTICS ABOVE!)**
Finally, the bound is also satisfied by an implementation of the form:
```rust
impl<T_0: Bound_0, .., T_N: Bound_N> ?const Trait<..> for Type<..> { .. }
```
where all `Bound_I`s are either `const` ones or transitively `?const` or `const`.

A bound of the form `T: ?const Trait` is meanwhile satisfied by any
implementation of the form:
+ `impl Trait for <type>`
+ `impl const Trait for <type>`
+ `impl ?const Trait for <type>`

However, as we just noted, if a `impl Trait for <type>` is provided,
then that can't be used in a `T: const Trait` bound.

Let's look at a larger example:

```rust
// T: ?const Default, U: const Default ⊢ (T, U): ?const Default:
impl<T, U> ?const Default for (T, U)
where
    T: ?const Default,
    U: const Default,
{
    fn default() -> Self {
        (T::default(), U::default())
    }
}

struct Foo;
struct Bar;
struct Baz;

impl Default for Foo {
    fn default() -> Self { Foo }
}

impl const Default for Bar {
    fn default() -> Self { Bar }
}

impl ?const Default for Baz {
    fn default() -> Self { Baz }
}

// We use these to verify that a type satisfies a bound:
fn take_default<X: Default>() {}
fn take_const_default<X: const Default>() {}
fn take_maybe_const_default<X: ?const Default>() {}

// Some interactions... This is not an exhaustive list!
fn main() {
    take_default::<Foo>(); // OK.
    take_default::<Bar>(); // OK.
    take_default::<Baz>(); // OK.
    take_default::<(Foo, Foo)>(); // TYPE ERROR.
    //                   ^-- Foo /: const Default.
    take_default::<(Foo, (Foo, Bar))>(); // Type ERROR.
    //                   ^--------- Same reason.
    take_default::<(Foo, Bar)>(); // OK.
    take_default::<(Foo, Baz)>(); // OK.
    take_default::<(Foo, (Bar, Baz))>(); // OK.
    // .. a lot of other combinations are OK as well,
    // so long as Foo is not in the second element of any tuple
    // with arbitrary nesting.

    take_const_default::<Foo>(); // TYPE ERROR.
    //                   ^-- Foo /: const Default.
    take_const_default::<(Foo, Bar)>(); // TYPE ERROR.
    //                    ^-- Same reason.
    take_const_default::<Bar>(); // OK.
    take_const_default::<Baz>(); // OK.
    take_const_default::<(Bar, Baz)>(); // OK.
    take_const_default::<(Bar, (Baz, Baz))>(); // OK.

    take_maybe_const_default::<Foo>(); // OK.
    take_maybe_const_default::<(Foo, Foo)>(); // TYPE ERROR.
    //                               ^-- Foo /: const Default.
    take_maybe_const_default::<(Foo, Bar)>(); // OK.
    take_maybe_const_default::<Bar>(); // OK.
    take_maybe_const_default::<Bar>(); // OK.
}
```

### Using the bounds

Naturally, if you have some constraint `T: const Trait`, then you may use
`Trait`'s methods in CTFE or a `const fn` context.

You may use `?const Trait` or `const Trait` in the usual places that you can
use bounds, these include using `(?)const Trait` in the requirements of an
associated type. For example:

```rust
trait Foo {
    // Normal associated types:
    type Bar: ?const Baz;
    type Quux: const Wibble;

    // Generic associated types (GATs):
    type C_GAT<T: const Wobble>;
    type MC_GAT<T: ?const Wobble>;
}

fn req_assoc_const<T>()
where
    T: const Iterator,
    T::Item: const PartialEq
{
    // ..
}
```

### Multiple bounds

It should be noted that `const` in `const Trait` applies to the trait and does
not apply to the whole bound. This means that the constraint `T: const Add + Mul`
has a normal implementation of `Mul`. If you want both traits to be required to
have `const` implementations, you should instead write `T: const Add + const Mul`.
Writing `T: const 'a` is also nonsensical and has no meaning.
**(TODO: PERMIT `T: const(Add + Mul)`?)**

### TODO / FOR REVIEWERS: Contravariance and the source of our troubles

OK; I have realized that this entire model proposed by the RFC is probably shit.

The essential reason is that the model in this RFC violates the fundamental rule
we want that:

```rust
Δ ⊢ T: const Trait
------------------ const_trait_subset_trait
Δ ⊢ T: Trait
```

Consider for example a minimal definition of `Hasher`:

```rust
pub trait Hash {
    fn hash<H: Hasher>(&self, state: &mut H);
}
```

If we want to define an implementation of `Hash` where `hash` can be `const fn`
in practice, then we must have that `H: const Hasher`. Otherwise we can't
actually use any of `Hasher`'s methods and so we can't define any useful implementation.
But if we write the desugared form:

```rust
impl Hash for Foo {
    const fn hash<H: const Hasher>(&self, state: &mut H) {
        // ..
    }
}
```
Then we have not truly restricted `hash`. We have imposed new requirements for
the user of `<Foo as Hash>::hash`, that is: `H: const Hasher`. This means that
we would not be able to use `impl Hash for Foo` for a bound `X: Hash`.
Thus, `const Hash` as a bound is not satisfied by a subset of types satisfying
`Hash`. So contravariance in type parameters of trait functions strikes and
we simply can't do this.

The most crucial trait we'll want to support `const` implementations for is `Iterator`.
It too won't work at all... Let's take a look at a minimal subset that illustrates
the problem with `Iterator`.

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Self::Item;

    fn for_each<F: FnMut(Self::Item)>(self, f: F);
}
```

This `for_each` method is enough. We won't be able to claim that `Foo: Iterator`
given the following definition of `Iterator for Foo`.

```rust
impl Iterator for Foo {
    ...

    fn for_each<F: const FnMut(Self::Item)>(self, f: F) { .. }
}
```

#### A new hope? `const fn` accepts `T: Trait` and we have no `?const`

There might be a different mechanism that is workable to preserve the rule:

```rust
Δ ⊢ T: const Trait
------------------ const_trait_subset_trait
Δ ⊢ T: Trait
```

Let's consider that we eliminate `?const fn` and let `const fn` be that instead.
What we have instead then is an implementation (desugared form..) like:

```rust
// T: const Default, U: const Default ⊢ (T, U): const Default
// T: Default, U: Default ⊢ (T, U): Default,
impl<T: Default, U: Default> Default for (T, U) {
    const fn default() -> Self {
        (T::default(), U::default())
    }
}
```

The sugared form would be:

```rust
impl<T: Default, U: Default> const Default for (T, U) {
    fn default() -> Self {
        (T::default(), U::default())
    }
}
```

....

## `impl const Trait`

### Universal quantification

Similar to the way that you may write `argument: impl Trait`,
you can also write `argument: impl const Trait` and `argument: impl ?const Trait`.
The behaviour is similar to that of writing `<T: const Trait>` and `argument: T`.
The behaviour with respect to turbo-fish and other such considerations is same
to that for `argument: impl Trait`.

A simple example:

```rust
const fn foo(universal: impl const Into<usize>) -> usize {
    universal.into()
}
```

### Existential quantification

And just like you can write `-> impl Trait` to existentially quantify in the
return position of a function, you may do the same with `-> impl const Trait`.
Of course, in this case the caller may assume that any method of `Trait` is
usable within a `const` context. An example of `-> impl const Trait` is:

```rust
fn foo() -> impl const Default + const From<()> {
    struct X;
    impl const From<()> for X { fn from(_: ()) -> Self { X } }
    impl const Default  for X { fn default() -> Self { X } }
    X
}
```

## `dyn const Trait`

Similar to static-dispatch existential quantification with `-> impl const Trait`,
you may also use dynamic-dispatch existential quantification with `dyn const Trait`.
An example:

```rust
const fn foo(boxed: Box<dyn const Iterator>) {
    // ..
}

const fn bar(reffed: &(dyn const Iterator)) {
    // ..
}
```

The syntax `Box<const Iterator>` being deprecated with the introduction of
`dyn Trait` is not supported to encourage the adoption of `dyn Trait` and to
not support legacy syntax.

## `#[derive(StandardLibraryTrait)]` and `?const`

Say we have the following type:

```rust
#[derive(Default)]
struct Foo<T> {
    tvar_field: T,
    bar_field: Bar,   
}
```

The compiler will currently generate the following implementation:

```rust
impl<T: Default> Default for Foo<T> {
    fn default() -> Self {
        Self {
            tvar_field: T::default(),
            bar_field: Bar::default(),
        }
    }
}
```

With this RFC, the compiler will instead generate a `?const Default`
implementation instead:

```rust
impl<T: ?const Default> ?const Default for Foo<T> {
    fn default() -> Self {
        Self {
            tvar_field: T::default(),
            bar_field: Bar::default(),
        }
    }
}
```

**TODO: HUSTON, WE HAVE A PROBLEM!**
If `Bar` is not `?const Default` due to manual impl,
then you have a breaking change??

## In relation to [specialization][RFC 1210]

[RFC 1210] defines specialization. In relation to that RFC and the specialization,
you may restrict further in specialized implementations compared to their base
implementation. However, loosening restrictions from the base implementation is
*not* permitted. An example of restricting further is:

```rust
pub trait New {
    fn new() -> Self;
}

struct Foo<T>(T);

impl<T: ?const New> New for Foo<T> {
    default ?const fn new() -> Self {
        Foo(T::new())
    }
}

impl New for Foo<()> {
    const fn new() -> Self {
        Foo(())
    }
}
```

Note however that restricting `T: ?const New` to `T: const New` in another
implementation does not substitute a valid specializing implementation.
That is, the following would be rejected by a Rust compiler:

```rust
impl<T: New> New for Foo<T> {
    default fn new() -> Self {
        Foo(T::new())
    }
}

impl<T: ?const New> ?const New for Foo<T> {
// ^-- error[E0119]: conflicting implementations...
    default fn new() -> Self {
        Foo(T::new())
    }
}

impl<T: const New> const New for Foo<()> {
// ^-- error[E0119]: conflicting implementations...
    fn new() -> Self {
        Foo(())
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

# Drawbacks
[drawbacks]: #drawbacks

TODO

# Rationale and alternatives
[alternatives]: #alternatives

TODO

## Syntax: `impl const Trait` vs. `const impl Trait`

TODO

## Syntax: `?const` vs. `const?`

TODO

# Prior art
[prior-art]: #prior-art

TODO

# Possible future work
[possible-future-work]: #possible-future-work

TODO

# Unresolved questions
[unresolved]: #unresolved-questions

TODO
