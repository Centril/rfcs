- Feature Name: `uniform_generic_bounds`
- Start Date: 2018-10-03
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

TODO improve this section.

Extends universal quantification (generics) in bounds with
`for<...>` to types and `const` values except for in `dyn` contexts.
As a consequence, closures can now be polymorphic on types and `const` values.
In other words, you can now write:

```rust
fn foo<F>(bar: F, baz: impl for<const N: usize> Fn([u8; N]))
where
    F: for<'a, T> Fn(&'a T) -> usize,
{
    bar(1u8) + bar(2u16);

    baz([1, 2]);
    baz([1, 2, 3])
}

fn bar()

foo(
    std::mem::size_of_val,
    |arr| println!("{:#?}", arr)
);
```

Additionally, lifetimes quantified in `for<...>` are allowed outlives requirements.

# Motivation
[motivation]: #motivation

TODO

- some motivation in rationale.

## Extending HRTBs to types improves understanding

TODO

## Use cases

TODO

### Generic closures

TODO

### Instead of boilerplate

TODO

### Scrap your boilerplate

TODO

## In-language encodings helps communication

TODO

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
fn quux(x: &u8) -> (&u8, &u8) { ... }
```

The compiler will desugar (5) to (6):

```rust
fn quux<'a>(x: &'a u8) -> (&'a u8, &'a u8) { ... }
```

In the case of (4) however, we need to add `for<...>` to hang the `'inner` on.

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
This RFC proposes no extension to higher ranked *types* other than to
allow outlives requirements in `for<...>` and only deals with *trait bounds*.

## Overview of the proposal

Henceforth in this RFC,
we will refer to a higher-rank trait bound as a *"generic bound"*.
What we've seen so far and what Rust currently supports are
*lifetime-generic* bounds (LGBs).

This RFC aims to:

[type variables]: https://en.wikipedia.org/wiki/Type_variable
[binders]: https://en.wikipedia.org/wiki/Free_variables_and_bound_variables
[outlives requirements]: https://github.com/rust-lang/rfcs/blob/master/text/2093-infer-outlives.md#motivation
[const-generic]: https://github.com/rust-lang/rfcs/pull/2000
[implied bounds]: https://github.com/rust-lang/rfcs/pull/2089
[RFC 2071]: https://github.com/rust-lang/rfcs/pull/2071
[RFC 2515]: https://github.com/rust-lang/rfcs/pull/2515

1. allow RGBs to optionally contain explicit *[outlives requirements]*
   in the form `F: for<'b: 'a> Fn(Thing<'b>)`.

2. introduce *type-generic* bounds (TGBs) of the form `F: for<T> Fn(Vec<T>)`.
   The *[type variables]* introduced in these `for<T>` *[binders]* may
   optionally be bounded such as with `for<T: Debug>`.

3. introduce *[const-generic]* bounds (CGBs) of the form
   `F: for<const N: usize> Fn([u8; N])`.

4. allow closures to be generic over types and `const` values.

    1. with explicit quantification of generic parameters
       using `for<...> |args...| ...`.

    2. with implicit quantification of generic parameters
       using `|x: impl Trait| ...`.
       Subject to stabilization of [RFC 2071] or [RFC 2515] `impl Trait`
       is also permitted in the explicit return type of a closure.

    3. by inferring the most general type.

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
`for<...>` this way. When you write a requirement such as `'b: 'a`,
the lifetime `'a` must as always be in scope. The construct `for<...>` is no
different in this respect. However, in this RFC, we provide no means to encode
the reverse requirement that `'a: 'b` when writing `for<'b> ...`.

The lifted restriction does not only apply in `fn` contexts but generally
anywhere `for<...>` can occur. This includes `where` clauses, in bounds applied
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

However, in this case, we had to *invent a new trait* and we can't make use of
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
more / less polymorphic.

It is also possible to expect a polymorphic function that itself expects a
polymorphic function. For example, we can write:

```rust
fn rank_3_function<G>(rank_2_fun: G)
where
    G: for<F: for<T: fmt::Debug> Fn(T)> Fn(F)
{
    rank_2_fun(send_messages_to_mars); // OK!

    rank_2_fun(|x: u8| {
        //         ^^ ERROR! `rank_2_fun` expects a *polymorphic* function.
        println!("x = {:#?}", x)
    });
}

fn main() {
    rank_3_function(debug_a_few_things_4);
}
```

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
to generate a vtable, it would have to contain generic function pointers.
This is generally not possible or it would have to possibly contain
an unbounded number of function pointers for each type the type
parameter could be instantiatied at which is infeasible.

## Const-generic bounds

[RFC 2000]: https://github.com/rust-lang/rfcs/pull/2000
[dependent products]: https://ncatlab.org/nlab/show/dependent%2Bproduct%2Btype
[dependent types]: https://en.wikipedia.org/wiki/Dependent_type

[RFC 2000] introduced *"const generics"* which let you be polymorphic
over *values* known at compile time. This amounts to a restricted form
of [dependent products] / Π-types (see: value-[dependent types]).

With const generics, we can define a function which works on arrays of any size:

```rust
fn sum_array<const N: usize>(array: [u8; N]) -> u8 {
    array.iter().sum()
}
```

However, as with types, we can't readily accept a function that is polymorphic
over constant values unless we also invent a trait:

```rust
trait ConstPolyFn {
    fn call<const N: usize>() -> u8;
}
```

We propose a more scalable and ergonomic mechanism instead:

```rust
fn foo<F>(accepts_matrix: F)
where
    F: for<const N: usize, const M: usize> Fn([[u8; N]; M])
{
    accepts_matrix([] : [[u8; 0]; 0]);
    accepts_matrix([[1]] : [[u8; 1]; 1]);
    accepts_matrix([[1, 2], [3, 4], [5, 6]] : [[u8; 2]; 3]);
}
```

As you can see, `accepts_matrix` works with matrices of all sizes.

## Generic closures

Hitherto in this RFC we have mainly seen `fn`s being passed to places where
polymorphic functions are expected. However, in the [summary], we did see an
example of a type-polymorphic *closure* being passed. This RFC proposes that
you should be able to construct such closures. We propose both an explicit
and an inferred way to construct polymorphic closures.

### Explicitly quantified

Closures can now be explicitly denoted as being polymorphic:

```rust
let f: impl for<T> Fn(T) -> T = for<T> |x: T| -> T { x };
//                              ------------------------
//                              fully explicit

let g: impl for<T> Fn(T) -> T = for<T> |x: T| x;
//                              ---------------
//                              return type is inferred

let h: impl for<const N: usize> Fn([u8; N]) -> [u8; N]
     = for<const N: usize> |x: [u8; N]| x;
//     ----------------------------------
//     we can quantify a const value

let i: impl for<'a> Fn(&'a u8) -> &'a u8 = for<'a> |x: &'a u8| x;
//                                         ---------------------
//                                         lifetimes also work

let j: impl for<'a, T: 'a, const N: usize> Fn(&'a [T; N]) -> &'a [T; N]
     = for<'a, T: 'a, const N: usize> |x: &'a [T; N]| x;    
//     ------------------------------------------------
//     lifetimes, types and const generics quantified together.
//     We can optionally drop `T: 'a` and just write `T` because the
//     outlives requirement is implied by `&'a [T; N]`.
```

All of the closures above are identity functions that are polymorphic to
varying degrees with `for<T> |x: T| -> T { x }` being most polymorphic.

### Implicitly quantified

Just like functions can use `impl Trait` in argument position
so too can closures now with equivalent meaning.
For example, we can write:

```rust
let f = |x: impl Debug| println!("the value of x is: {:#?}", x);
```

The closure `f` will satisfy the bound `for<T: Debug> Fn(T)`.
Just as with functions, each occurence of `impl Debug` will
become a unique type parameter in `for<...>`.

To further enhance the conistency with `fn` items,
closures can also annotate the return type with `-> impl Trait`
with the same type-hiding semantics as in `fn`.
For example, we can write:

```rust
let g: impl Fn(u8) -> impl Iterator<Item = u8>
                   // ^^^^ Allowed as the return type of Fn(...)
     = |x: u8| -> impl Iterator<Item = u8> { 0..=x };
```

As noted in the overview, `impl Trait` as the return type of
`Fn`, `FnMut`, and `FnOnce` traits as well as closures depends
on the the outcome of [RFC 2071] and [RFC 2515].

### Inferred

In all of the cases above except `f`,
we can also drop the quantifiers `for<...>` and write:

```rust
let g = |x| x;

let h = |x: [_; _]| x;

let i = |x: &u8| x;

let j = |x: &[_; _]| x;
```

In all of these cases, the compiler infers the most general type that can be
assigned to the closures. Here, `h`, `i`, and `j` can't be more polymorphic
than the signatures we gave in the previous section because we have
intentionally constrained them to. However, the compiler will infer
the most general type for any closure. For example, if we write:

```rust
let adder = |x| x + x;
```

then:

```rust
add_to_self : impl for<T: Copy + Add> Fn(T) -> <T as Add>::Output
```

will hold.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A `where` clause contains a comma separated list of *constraints*
(also referred to as *"predicates"*). A list of constraints is also
referred to as an *axiom set*. A constraint is either a *goal* to be proven,
or a *clause* which is a proof/witness that a constraint holds.
A constraint has the kind `Constraint`.
An example of a constraint is `T: Copy + 'a` where `Copy + 'a` is a *bound*.

A bound has the kind `k0 -> Constraint` where `k0 : ' | * ;` is a base kind.
Here `'` is the kind of lifetimes (otherwise referred to as "regions"),
e.g. `'a` and `'static`. Here, `*` is the kind of types, e.g. `Vec<bool>`.
Note that `Vec` is a type constructor of kind `* -> *` and *not* a type.

In a `where` clause, a constraint of form `for<P0, ..., Pn> Type: Bound`
where `Type` or `Bound` references at least one of `P0...Pn` is referred
to as a *generic constraint*.

If instead a `for<P0, ..., Pn>` binder is on the RHS of `:`, that is inside
of `Bound` it is instead referred to as a *generic bound*.

## Grammar

We change the grammar of places where `for<...>` is permitted to:

```rust
list1<p, s> : p (s p)* ;
list_1_term<p, s> : list1<p, s> s? ;

for_binders : "for" "<" (binders ","?)? ">" ;
binders : lifetimes | lifetimes "," tail_vars | tail_vars ;
tail_vars : list1<tail_var, ","> ;
tail_var : const_generic | tyvar_and_bounds ;

const_generic : "const" ident ":" ty ; // Depends on RFC 2000 (const generics)

bounds<bound> : ":" list_1_term<bound, "+">? ;

lifetimes : list1<lifetime_var, ","> ;
lifetime_var : lifetime bounds<lifetime> ;
lifetime : LIFETIME | "'static" ;

tyvar_and_bounds : ident bounds<tyvar_bound> ? ;
tyvar_bound : "?"? for_binders? bound ;
```

Here `LIFETIME` refers to the token representing a lifetime.

We change the grammar of expressions from:

```rust
expr
: ...
| lambda_expr
| "move" lambda_expr
;
```

to:

```rust
expr
: ...
| for_binders? lambda_expr
| "move" for_binders? lambda_expr
;
```

## Static semantics

In this RFC, the static semantics of a generic binder `T` with any set of bounds
`B0 + ... + Bn` applied to it (`<T: B0 + ... + Bn>`) is explained in terms of
the static semantics of desugaring to `<T>` and `where T: B0 + ... + Bn`.
This also applies to `arg: impl for<...> Trait<...>` as well as
`-> impl for<...> Trait<...>`.

### The `'static` lifetime cannot be quantified

While `'static` *is* permitted in the grammar of `lifetimes` above because
the `item` macro fragment matcher allows it, but the compiler will also
disallow it to be quantified after parsing. This RFC does not introduce the
ability to write `for<'static>` syntactically since it is already permitted
in stable compilers of Rust, but we take the opportunity to clarify here.
The restriction on quantifying `'static` can be seen as a restriction
on introducing shadowed parameter names where `'static` exists in the
"global" scope.

### Outlives requirements in `for<...>`

A `for<...>` binder inside a bound or a constraint may contain a list of
lifetimes `L0`...`Ln` (`lifetimes` in concrete syntax) adhering to the grammar
`lifetime_var`. As examples, a Rust compiler will accept a type of form
`Box<dyn for<'b: 'a> Fn(&'b u8)>`, a type of form `for<'b: 'a> fn(&'b u8)`,
a constraint `for<'b, 'c> &'b &'c T: Debug`, or a bound `T: for<'a> MyTrait<'a>`.

None of the lifetimes `Li` may shadow any named and quantified lifetimes
in any ancestor scope. The lifetimes `Li` may each optionally contain
a `+`-separated (or terminated) list of lifetimes `Lij` each of which
the lifetime `Li` must outlive (by outlive we mean `≥` rather than `>`).
Any referenced lifetimes `Lij` must be either be `'static` or must have
been brought into scope in an ancestor scope.

Given a binder `for<L0: L0j, ..., Ln: Lnj> $bound` in set of constraints on a
definition `$def` (including `struct`, `enum`, `union`, `fn`, `trait`, or `impl`),
for a reference to `$def` to be well-formed (for an implementation this means
that this is a requirement for it to be considered implemented),
for each lifetime `Li`, `$bound` must hold for any arbitrary lifetime
which outlives a set of lifetimes `⊆ Lij`. This entails that a reference to
`$def` may not impose an additional set of outlives requirements on `Li` but
may weaken the set of requirements. For example, if `for<'b> Foo<'b>` holds
then so too will `for<'b: 'a> Foo<'b>`.

Conversely, inside `$def`, for each lifetime `Li`, the `$bound` is considered
to hold for any arbitrary lifetime which at least outlive all lifetimes `Lij`.

In all cases, the implied bounds of `$bound` will be taken into account when
determining the full set of outlives requirements that apply to lifetimes `Li`.
For example, if given the constraint `for<'a> &'a &'b u8: $bound` then
`'b: 'a` is implied.

Where a value of a type of form `dyn for<L0: L0j, ..., Ln: Lnj> $bound`
or of form `for<L0: L0j, ..., Ln: Lnj> fn($ty_arg_0, ..., $ty_arg_n) -> $ty_ret`,
is required, the compiler will check that the bound `for<...> $bound` holds
according to the logic above and conversely once a value of such a type
is obtained, the bound `for<...> $bound` may be assumed to hold.

In chalk terms, a constraint `for<L0, ..., Ln> $type: $bound`
is lowered into logic like so:

```rust
forall<L0, .., Ln> {
    if(
        Outlives(L0: L00) && .. Outlives(L0: L0m) &&
        ..
        Outlives(Ln: Ln0) && .. Outlives(Ln: Lnm)
    ) {
        $type: $bound
    }
}
```

Here, `$type` may refer to any of the lifetimes `Li`.
Meanwhile, a constraint `$type: for<L0, ..., Ln> $bound` is lowered
the same way but `$type` may not reference any `Li`.

### `dyn for<P0...Pn>` is not allowed type / `const` parameters

*After* macro expansion, a Rust compiler will emit an error if the bound after
`dyn` contains `for<$params>` where `$params` does not match `lifetimes ","?`.
Do note that this still means that `dyn for<T> $bound` *is* part of the grammar
because of the significance of macros in Rust. The same applies to types.
While `for<T> fn(T)` is a syntactically valid type, it is not semantically
valid and thus it is not well-formed.

### Type-generic bounds and constraints

While the restrictions in the previous section wrt. `dyn` applies,
similar to lifetimes, a `for<...>` binder inside a bound or a constraint
may contain a list of type variables `T0...Tn` where each variable `Ti`
adheres to the grammar `tyvar_and_bounds`. For example, a Rust compiler
will accept a constraint `for<T: Debug> Vec<T>: Debug`, or a constraint
`F: for<T: Add> Fn(T, T) -> T`.

A variable `Ti`, which may not shadow variables in parent scopes, may optionally
contain a `+`-separated (or terminated) list of bounds `Tij` which can either be:
+ a lifetime brought into scope in some parent scope or in the same binder
  or the `'static` lifetime.
+ a reference to a trait, e.g. `Ti: Debug`
+ another bound of form `for<...> $bound`

Given a binder `for<T0: Tij, ..., Tn: Tnj> $bound` in set of constraints on a
definition `$def` (including `struct`, `enum`, `union`, `fn`, `trait`, or `impl`),
for a reference to `$def` to be well-formed (for an implementation this means
that this is a requirement for it to be considered implemented), for each type
variable `Ti`, `$bound` must hold for any arbitrary type for which a set
of bounds `⊆ Tij` holds. Just as with lifetime variables, this entails that a
reference to `$def` may not impose an additional set of constraints on `Ti` but
may weaken the set of requirements. For example, if `for<T> Foo<T>` holds
then so too will `for<T: Debug> Foo<'b>`.

Conversely, inside `$def`, for each variable `Ti`, the `$bound` is considered
to hold for any arbitrary type for which at least all bounds `Tij` holds.
In particular, this means that an arbitrary type may satisfy more bounds
than the set in `Tij` but not fewer.

As with lifetimes, the implied bounds of `$bound` will be taken into account
when determining the full set of constraints on `Ti`. For example, if given
the constraint `for<'a, T> Foo<&'a T>` then `T: 'a` is implied.

In chalk terms, a constraint `for<T0, ..., Tn> $type: $bound`
is lowered into logic like so:

```rust
forall<T0, ..., Ln> {
    if(
        Constraint_00 && ... Constraint_0m &&
        ...
        Constraint_n0 && ... Constraint_nm
    ) {
        $type: $bound
    }
}
```

Here, `$type` may refer to any of the type variables `Ti`.
Meanwhile, a constraint `$type: for<T0, ..., Tn> $bound` is
lowered the same way but `$type` may not reference any `Ti`.

Note that no restriction is imposed on the nesting of `for<...>` binders.
For example, we may write `G: for<F: for<'a, T: fmt::Debug> Fn(&'a T)> Fn(F)`
which will be lowered into:

```rust
forall<F> {
    if(forall<'a, T> {
        if(Outlives(T, 'a) && Implemented(T: ::core::fmt::Debug)) {
            Implemented(F: Fn(&'a T))
        }
    }) {
        Implemented(G: Fn(F))
    }
}
```

### Value-generic bounds and constraints

A `for<...>` binder may contain zero or more `const`-value-variables of
form `const C0: τ0, ..., const Cn: τn`. The semantics of what such a variable
means is given by [RFC 2000]. These variables `Ci` do not have to be placed
together contiguously and may be mixed in any order with type variables,
described in the previous section, `T0...Tn`. However, just like type variables,
`Ci` must come before any lifetime variable.

Most of the logic with respect to type variables applies to const-variables as
well. The notable difference here is that there is no way to bound a value
variable `Ci` as such a possibility does not exist for `<const N: Type>`
variables in prenex form.

Implied bounds apply for `const` variables as well. For example, if you write:
`for<'a, T, const X: &'a T> ...` then `T: 'a` is implied.

### Closure expressions

Let:

+ `Li` denote the `i`th quantified lifetime variable (`lifetime_var`).

  There may be zero or more lifetime variables.

+ `Pi` denote the `i`th quantified type variable (`tyvar_and_bounds`)
  or const value variable (`const_generic`).

  There may be zero or more such variables.

  Note that for both `Pi` and `Li` only the variables in the *immediate*
  `for<...>` binder is included. Variables in nested `for<...>` are not
  part of `Pi` and `Li`.

+ `xi`, denote a pattern which is a parameter of a closure
  including an optional type `Ti` if it is specified.

+ `Tr` denote the optionally specified return type of a closure.

+ `body_expr` denote the expression that makes up the body of a closure.

When type checking a closure of the following form in abstract syntax:

```rust
move? for<L0, ... Ln, P0, ..., Pn> |x0, xi, xn| -> Tr { body_expr }
```

The following steps, which may be reordered if the semantics are preserved,
are taken:

1. A new environment `Γc` is added.

2. All `Li` are added to `Γc` and noted as untouchable variables.

3. All `Pi` are added to `Γc` and noted as untouchable variables.

   By untouchable, we here mean that a variable `Pi` or `Li` is
   not a unification variable since it originaltes from an explicitly
   given signature and thus it cannot be instantiated at a different type.

4. `Γc` is checked to not contain any variables introduced in ancestors
   typing environments collectively known as `Γa`.
   The parent environment and ancestors include variables on
   `impl<...>`, `fn<...>`, and a `for<...>` binder in a ancestor closures.

5. For all occurences of `impl Bounds` in `xi` a type variable
   `$F: Bounds` where `$F` is a fresh name is added to `Γc` and
   the type of `xi` is noted as `$F`.
   This does not apply to `impl Bounds` when it occurs in `$x`
   and `$y` of `$TraitRef($x) -> $y`.

   If `impl Bound` occurs in `Tr` then it is substituted for a fresh
   `existential type $F: Bounds`.

6. `Γc` is checked to be a well formed typing environment and implied bounds
   are added.

7. The return type of the closure is noted in `Γc` as `Tr` and is checked for
   well formedness taking into account implied bounds.
   If `Tr` was not specified, a unification variable `?Tr` is added to 
   `Γc` and set as the return type of the closure.

8. For all patterns `xi`, for any part of the pattern `xi` where
   the type is unknown, a unification variable `?Tij` is added to `Γc`.

9. Given the environments `Γc` and `Γa` (containing mappings of all captures),
   which are known not to have any shadowing due to 5.,
   unification of all variables `?Tr` and `?Tij` is
   attempted with the expression `body_expr`.

   During unification, some variables may be substituted for known values
   or partially known values, including `Li` and `Pi`.
   When doing so, additional variables may be introduced where needed.

   Additionally, during unification, additional constraints discovered
   for `body_expr` to be well formed may be added to `Γc`.
   These constraints may reference variables `Li` or `Pi`.

10. Having unified with `body_expr`, all remaining unification variables in `Γc`
    are substituted for fresh variables in `Γc`.

11. The product type `$C` is constructed with fields for all captured bindings
    taking into account `move` or non-`move` mode.
    
    The implementations of `Fn`, `FnMut`, and `FnOnce`,
    depending on what the closure affords, for `$C` are constructed.

    The variables, including constraints, in `Γc` are added to each
    implementation.

    The associated type `Output` of each implementation becomes `Tr` which is
    either a variable at this point or a concrete type.

    The types `Ti` are set as the argument type of the implementations.

    At this point, the most general type of the closure has been inferred.

## Dynamic semantics

All changed and new constructs in this RFC derives dynamic semantics
from existing dynamic semantics after monomorphization.
In other words, this RFC has no impact on the dynamic semantics of Rust.

# Drawbacks
[drawbacks]: #drawbacks

TODO

- complexity

- type inference decidability?

TODO

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

- motivate `dyn for<T>` in grammar.

- implied bounds

- the combination features, independence, etc.

TODO

# Prior art
[prior-art]: #prior-art

TODO

- Generic closures in C++

- Generic closures in Haskell

- Look into Scala, xML wrt. RankNTypes & generic closures.

- QuantifiedConstraints

- RankNTypes

- Look into dependently typed languages

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Is the behavior with respect to implied bounds right?

# Future work
[future-work]: #future-work

## Shorthand syntaxes `impl<T, ...>` and `dyn<'a, ...>`

A possible extension of this RFC is to permit the syntax `impl<T, ...> $bound`
as argument and return types. This would allow you to write:

```rust
fn debug_a_few_things_4(debug: impl<T: fmt::Debug> Fn(T)) {
    ...
}

fn identity() -> impl<T> Fn(T) -> T { |x| x }
```

One thing to note here is that the binder `<T>` in `impl<T>` applies to the
entire set of bounds separated by `+` as opposed to `impl for<T> X<T> + Y`
which associates as `impl (for<T> X<T>) + Y`.

The main benefit of this syntax is that if you write:

```rust
impl<T> MyTrait<T> for MyType {
    ...
}
```

then the syntax `impl<T> MyTrait<T>` can be used as a type to denote a type
which must satisfy `MyTrait<T>` for all `T`. This correspondence in between
implementation syntax and type syntax could be a boon for learnability.

Similarly, we could allow lifetimes to be quantified in `dyn` as:
`Box<dyn<'a> Fn(&'a u8)>`.

## `impl Fn(impl Debug)`

The syntax `impl Fn(impl Debug)` could be used as a shorthand for
`impl for<T: Debug> Fn()`. This would improve the consistency between functions
written as `fn foo(arg: impl Debug) { ... }` which already exist but also with
closures written as `|arg: impl Debug| ...` as proposed in this RFC.

However, one potential drawback could be that some users might interpret
`arg: impl Fn(impl Debug)` as:

```rust
fn foo<T, F>(arg: F) where T: Debug, F: Fn(T) { ... }
```

The fact that `Fn(...)` uses parenthesis instead of `<...>` might be enough
to justify the difference in semantics.

## `TypeCtor<impl Trait>` in `where`

Another extension with which `for<T: Trait> ...` could be more succinctly
expressed would be to allow `impl Trait` inside of `where` clauses.
For example, one could write:

```rust
where
    Vec<impl Clone>: Clone
```

which could be an equivalent way of expressing:

```rust
where
    for<T: Clone> Vec<T>: Clone
```

In chalk, this would map to the goal:

```rust
forall<T> {
    if(Implemented(T: Clone)) {
        Implemented(Vec<T>: Clone)
    }
}
```

## Denoting *"reverse constraints"* in `for<...>`

In this RFC we have not introduced any mechanism to bound type variables
and lifetimes quantified in `for<...>` with so called *"reverse constraints"*.
An example of a reverse constraints is:

```rust
fn foo_1<T>(arg: Bar)
where
    Bar: Into<T> // Reverse constraints!
{
   ... 
}
```

In this case, the `where` clause can't be removed since you can't write:

```rust
fn foo_2<T, Bar: Into<T>>(arg: Bar) { ... }
```

The function `foo_2` would not be the same as `foo_1` because `Bar` would be
interpreted as a type variable in `foo_2` but as a concrete type in `foo_1`.

Since `for<...>` can only quantify variables and bound them there is no
way denote a reverse constraints. To alleviate this, we might need some
form of local `where` clause which applies to a `for<...>` binder.
Possible syntaxes for such a `where` clause include:

```rust
fn foo<'global, F>()
where
    F: for<'a, 'local> where { 'global: 'local }
       FnOnce(Context<'a, 'local, 'global>)
```

Here `where { ... }` includes braces because there would be ambiguity otherwise.

```rust
fn foo<'global, F>()
where 
    F: for<'a, 'local> FnOnce(Context<'a, 'local, 'global>) 
       where 'global: 'local,
```

Here there is an ambiguity if you want to add more constraints to the outer
`where` clause. This can be resolved with various schemes for disambiguation,
for example, you can write:

```rust
fn foo<'global, F>()
where 
    F: for<'a, 'local> FnOnce(Context<'a, 'local, 'global>) 
       where {
           'global: 'local,
       },
    ...
```

or:

```rust
fn foo<'global, F>()
where 
    F: {
        for<'a, 'local> FnOnce(Context<'a, 'local, 'global>) 
        where'global: 'local,
    },
    ...
```

or:

```rust
fn foo<'global, F>()
where
    {
        F: for<'a, 'local> FnOnce(Context<'a, 'local, 'global>) 
        where'global: 'local,
    },
    ...
```

You could also use `( ... )` as the grouping mechanism instead of braces.
