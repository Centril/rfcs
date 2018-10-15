- Feature Name: `infer_singleton`
- Start Date: 2018-10-13
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

Allow *value* inference of singleton types with `_` in expression positions.

# Motivation
[motivation]: #motivation

## Ergonomics

[`PhantomData`]: https://doc.rust-lang.org/nightly/std/marker/struct.PhantomData.html

While the cases in which this RFC will apply are in the grand scheme
of things not that major, it does provide a reduction in some tedious
and boring boilerplate where trivial values such as [`PhantomData`] need
to be constructed.

## Readability

While you may think that writing out `Empty { _marker: PhantomData }`
and similar is more readable because it is more explicit,
we argue that explicitly writing out trivially and unambiguously
inferable expressions instead contributes to line noise that obscures
the important code that carries the intent the author has.

By writing out `_`, we can communicate to the reader that the value
is trivial, so *don't bother thinking about it*.

## Uniform treatment of `_`

Pattern syntax work best when the construction syntax is "the same".
For example, we can construct a struct literal with `Foo(1, 2)`
and then we can match on that exact same case with `Foo(1, 2)`.

In pattern matching, we can use the pattern `_` to signify *"don't care"*.
However, there doesn't exist an expression equivalent to the `_` pattern.
For singleton types, this RFC proposes the equivalent.
While this won't work for non-singleton types, we do gain a modicum of
unification. We argue that such gains in consistency improves learning.

[`const` generics]: https://github.com/rust-lang/rfcs/pull/2000

Also note that `_` is used in type contexts to mean "please infer" just like
proposed for expressions in this RFC. Furthermore, with [`const` generics],
situations will occur where `const` parameters can be inferred because they are
uniquely determined by some other value or a type. In such situations,
you should be able to write `_` to for example infer the size of an array.
This RFC would be consistent with such inference.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Vocabulary

[zst]: https://doc.rust-lang.org/nomicon/exotic-sizes.html#zero-sized-types-zsts
[unit]: https://en.wikipedia.org/wiki/Unit_type
[terminal]: https://en.wikipedia.org/wiki/Initial_and_terminal_objects

+ A *[Zero-Sized Type][zst] (ZST)* is a type for which
  `std::mem::size_of::<TheType>()` evaluates to `0`.

+ A *singleton type* or *[unit type][unit]* is a type which can only be
  constructed in *one way* wherefore it carries no information.
  This is not always the same as a ZST but it often is.
  One property of a singleton type `S` is that for *any* type `T`,
  it is always possible to define a non-diverging function `fn(T) -> S`.
  This is because all singleton types are *[terminal]* objects.

## Proposal

What is proposed in this RFC is to allow all singleton types, that is,
all zero sized types for which all fields are visible in the current scope,
and for which the type is not `#[non_exhaustive]`,
to be inferred with `_` in expression contexts.

A trivial but uninteresting example is:

```rust
fn foo() {
    _ // The value () is inferred for the type (). 
}
```

This also works:

```rust
fn foo() -> Result<(), u8> {
    Ok(_) // We still infer () for _ here, so we get Ok(()).
}
```

Aggregate types also work:

```rust
struct Empty<T> {
    _marker: PhantomData<T>,
}

impl<T> Empty<T> {
    pub fn new() -> Self { _ }
}
```

But visibility is taken into account:

```rust
mod foo {
    pub struct Empty<T> {
        _marker: PhantomData<T>,
    }
    
    impl<T> Empty<T> {
        pub fn new() -> Self { _ } // Ok!
    }
}

fn bar() -> Empty<T> {
    _
//  ^ Error! Cannot infer `Empty { _marker: PhantomData }` because
//    the field `_marker` is not visible in this context.
}
```

And so is `#[non_exhaustive]`:

```rust
#[non_exhaustive]
struct Foo;

fn bar() -> Foo {
    _
//  ^ Error! The type `Foo` is `#[non_exhaustive]`.
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Grammar

To the expression grammar of the language, we add:

```rust
expr : ... | "_" ;
```

## Static semantics

Let:

+ `x`, `v`, and `f` range over valid identifiers.

+ `Γ` be a well-formed typing environment.

+ `τ` denote a well-formed expression in context `Γ`.

+ `σ` denote a well-formed type in context `Γ`.

+ `const(τ)` be a predicate on `τ`, in context `Γ`,
  holding iff `τ` is evaluable at compile time.

+ `uninhabited(σ)` be a predicate on `σ`, in context `Γ`,
  holding iff `σ` is uninhabited.

+ `non_exhaustive(σ)` be a predicate on `σ`, in context `Γ`,
  holding iff `σ` has `#[non_exhaustive]` applied to it.

+ `visible(ρ)` denote if the projection `ρ` is visible in `Γ`.

### Definition of `singleton` types

Let `singleton(σ)` be a predicate on `σ` in context `Γ` which holds iff
`σ` is a singleton type.

It is defined inductively like so:

1. Whatever type `σ` is, `PhantomData<σ>` is always `singleton`.
   For example, `PhantomData<String>` is `singleton` even though `String` is not.

   ```prolog
   ----------------------------- SingPhantomData
   Γ ⊢ singleton(PhantomData<σ>)
   ```

2. A tuple is `singleton` iff all the types it is made up of are also `singleton`.
   It follows from this rule that the zero-tuple `()` is `singleton`.

   ```prolog
   Γ ⊢ singleton(σ0), ..., Γ ⊢ singleton(σn)
   ----------------------------------------- SingTuple
   Γ ⊢ singleton( (σ0, ..., σn) )
   ```

3. An array is `singleton` if the element type is `singleton`.

   ```prolog
   Γ ⊢ singleton(σ)
   Γ ⊢ τ : usize
   Γ ⊢ const(τ)
   --------------------- SingArray
   Γ ⊢ singleton([σ; τ])
   ```

4. An array of size `0` is always `singleton` even if the element type isn't.

   ```prolog
   Γ ⊢ τ : usize
   Γ ⊢ const(τ)
   Γ ⊢ τ ⟶ 0usize
   --------------------- SingArrayZero
   Γ ⊢ singleton([σ; τ])
   ```

5. A struct of form `struct x { f0: σ0, ..., fn: σn }` is `singleton`
   iff `#[non_exhaustive]` does *not* apply
   and all fields are visible and `singleton`.

   Tuple structs are treated as named structs but with integer fields
   instead of normal identifiers.

   Structs of form `struct x;` are treated the same as `struct x {}`.

   ```prolog
   Γ ⊢ σ = struct x<...> { f0 : σ0, ..., fn : σn }
   Γ ⊢ non_exhaustive(σ) = ⊥
   Γ ⊢ visible(x.f0), ..., Γ ⊢ visible(x.fn)
   Γ ⊢ singleton(σ0), ..., Γ ⊢ singleton(σn)
   ----------------------------------------------- SingStruct
   Γ ⊢ singleton(σ)
   ```

6. An enum is `singleton` iff: `#[non_exhaustive]` does not apply,
   it has a variant where all field types are `singleton`,
   and all other variants are uninhabited.

   Tuple and unit variants are, as with structs, treated the same way as
   if they had named fields.

   ```prolog
   Γ ⊢ σ = enum x<...> {
           v0 { f00: σ00, ..., f0m: σ0m },
           ...,
           vn { fn0: σn0, ..., fnm: σnm },
       }
   Γ ⊢ non_exhaustive(σ) = ⊥
   Γ ⊢ ∃ i. [
           singleton(σi0), ..., singleton(σin) ∧
           ∀ j ≠ i. uninhabited( (σj0, ..., σjn) )
       ]
   -------------------------------------------------- SingEnum
   Γ ⊢ singleton(σ)
   ```

   Visibility does not apply because `pub` is forced for all fields and variants.

   Note that the rules wrt. inhabitedness entail that `Result<(), !>` is
   `singleton` because it has one inhabited variant `Ok(..)` and all the
   field types of that are `singleton` (because `()` is).

7. All other types are not `singleton`.

## Changes to term typing

We introduce the term typing rule:

```prolog
Γ ⊢ _ ⇝ σ
Γ ⊢ singleton(σ)
---------------- TmInferSingleton
Γ ⊢ _ : σ
```

This means that if `_` infers to the type `σ`, which is a `singleton` type,
then `_` is a well-formed expression typed at `σ` in the context `Γ`.
If the type `σ` is not `singleton` in `Γ` then a type error will be raised.
This entails that if we don't know enough about the type, e.g. if we have
`(T,)` where `singleton(T)` cannot be derived because it is either a
type unification variable or an explicitly quantified type variable, 
then we cannot write `(_,)` or `_`.

## Elaboration / Desugaring

We define an elaboration of `_` into a Rust with explicitly given values with
the relation `⇝iv`. The elaboration is given in small-step style.
To fully elaborate into a Rust without `_` at the expression level,
the reflexive and transitive closure `⇝iv*` is used.

1. When `_` is inferred to be of type `PhantomData<T>` for some type `T` then
   it elaborates to `PhantomData`.

   ```prolog
   Γ ⊢ σ1 = PhantomData<σ0>
   Γ ⊢ _ : σ1
   ------------------------ InferValPhantomData
   Γ ⊢ _ ⇝iv PhantomData
   ```

2. When `_` is inferred to be of a tuple type which is `singleton`,
   then it elaborates to `(_, ..., _)`.

   ```prolog
   Γ ⊢ _ : (σ0, ... σn)
   Γ ⊢ singleton(σ0), ..., Γ ⊢ singleton(σn)
   ----------------------------------------- InferValTuple
   Γ ⊢ _ ⇝iv (_, ..., _)
   ```

3. When `_` is inferred to be of an array type which is `singleton`,
   then it elaborates to `[_, array_size]`.

   ```prolog
   Γ ⊢ τ : usize
   Γ ⊢ const(τ)
   Γ ⊢ _ : [σ; τ]
   Γ ⊢ singleton(σ)
   ---------------- InferValArray
   Γ ⊢ _ ⇝iv [_; τ]
   ```

4. When `_` is inferred to be of an array of size `0` it elaborates to `[]`.

   ```prolog
   Γ ⊢ τ : usize
   Γ ⊢ const(τ)
   Γ ⊢ τ ⟶ 0usize
   Γ ⊢ _ : [σ; τ]
   ---------------- InferValArrayZero
   Γ ⊢ _ ⇝iv []
   ```

   Rules 3 and 4 are confluent because `[]` and `[_; 0]` are equivalent.

5. When `_` is inferred to a struct, it elaborates to `x { f0: _, ..., fn: _ }`.
   As in the section on `singleton`s, we treat tuple and unit structs as
   if they had named fields.

   ```prolog
   Γ ⊢ σ = struct x<...> { f0 : σ0, ..., fn : σn }
   Γ ⊢ non_exhaustive(σ) = ⊥
   Γ ⊢ visible(x.f0), ..., visible(x.fn)
   Γ ⊢ singleton(σ0), ..., singleton(σn)
   Γ ⊢ _ : σ
   ----------------------------------------------- InferValStruct
   Γ ⊢ _ ⇝iv x { f0: _, ..., fn: _ }
   ```

6. When `_` is inferred to an enum with a `singleton` variant and where
   all other variants are uninhabited,
   then `_` elaborates to `x::vi { fi0: _, ..., fin: _ }`.
   Here, `i` denotes the sole `singleton` variant and where
   `fi0, ... fin` denote the fields of said variant.

   As before, we treat tuple and unit variants as if they had named fields. 

   ```prolog
   Γ ⊢ σ = enum x<...> {
           v0 { f00: σ00, ..., f0m: σ0m },
           ...,
           vn { fn0: σn0, ..., fnm: σnm },
       }
   Γ ⊢ non_exhaustive(σ) = ⊥
   Γ ⊢ ∃ i. [
           singleton(σi0), ..., singleton(σin) ∧
           ∀ j ≠ i. uninhabited( (σj0, ..., σjn) )
       ]
   Γ ⊢ _ : σ
   ----------------------------------------------- InferValEnum
   Γ ⊢ _ ⇝iv x::vi { fi0: _, ..., fin: _ }
   ```

7. Otherwise there's nothing to elaborate.

## Dynamic semantics

This proposal has no effect on the dynamic semantics of Rust since elaboration
eliminates all uses of `_` to explicitly given values.

# Drawbacks
[drawbacks]: #drawbacks

Depending on your point of view, this might:

1. Increase language complexity (though a rather simple mechanism is used that
   is similar to how `_` is used in pattern matching and type contexts
   and which will also need to be allowed for value inference with const generics).

2. Reduce readability. We give rationale for why we think this isn't the case
   in the [motivation].

   Claims around reduced readability go along the same lines as they do for
   inference in general (including inferring types). Annotating explicitly
   that a binding or expression is of a certain type can help readability,
   and yet we do have type inference. Arguably, writing out types is more
   helpful than explicitly writing out expressions for singleton values is.
   By replacing such expressions with `_` you know syntactically that it is
   singleton; which you wouldn't know from `Foo::Bar`.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## The syntax

We already use `_` in type and pattern matching contexts to denote
"please infer" or its dual "I don't care". It therefore makes sense to use
the same concrete syntax for inferring values as used for types to make the
language minimally additionally complex. By doing so, we improve or don't
make the learning experience notably worse.

## The semantics

The semantics and where `_` can be used are fairly conservative.
Singleton types are those where the user explicitly hasn't declared that they
might add additional fields or variants and where there is unambiguously one
value that inhabits the given type. We infer only those types because we don't
want to be opinionated and make arbitrary choices for the user.

# Prior art
[prior-art]: #prior-art

[Agda]: https://agda.readthedocs.io/en/v2.5.4.1/language/record-types.html
[Coq]: https://coq.inria.fr/
[Idris]: https://www.idris-lang.org/

In the dependently typed language [Agda], which has typed equality,
there are eta rules which allow you to for example write:

```agda
record A : Set where
  constructor B

foo : A
foo = _
```

This would be the semantic equivalent of writing:

```rust
enum A { B }

fn main() { // Not necessary in Agda but it is in Rust.
    let foo: A = _;
}
```

in Rust which this RFC would allow.

Meanwhile, [Coq] and [Idris], which are also dependently typed languages with
untyped equality you cannot get away with writing that.
However, in Idris, you can still employ proof search to achieve the same thing:

```idris
infer : {a : Type} -> {auto x : a} -> a
infer {x} = x;

data A = B

foo : A
foo = infer
```

The proposed mechanism in this RFC is similar to a proof search.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Should `&'a T` where `singleton(T)` be inferable with `_`?

   Technically, these are not singleton types because the address is observable.
   However, the observability of the address is often uninteresting for most
   use cases. This also applies to `&'a mut T`.

# Future work
[future-work]: #future-work

## Using `...` to infer multiple `_`

In cases where you would write `Foo(42, _, _, _)` you could gain in
ergonomics by allowing all the `_`s to be inferred with `Foo(42, ...)`.
As we don't know how frequently this would occur, we punt on this for the
time being. One conjecture is that having many singleton typed fields is
somewhat unusual and that one or two would be more common.

## Using `T::default()` in other cases

The expression form `_` could be made more useful, depending on your viewpoint,
by allowing it to desugar to `T::default()` in cases where `T` is not a
`singleton` type. A drawback to this would be that we can no longer be sure
that the meaning of `_` is always trivial and so less information is given
to the reader by the expression form `_`. This possibly interacts with `...`
per the notes in the previous section.

## In the event of GADTs

If we would add some form of GADTs, either in full or parts of it,
there would likely be more cases in which types should be `singleton`.

For example, if we have:

```rust
enum Foo<T> {
    Bar,
    Baz(T) where T: Copy
}
```

Then `singleton(Foo<String>)` should presumably hold because
`Foo::Baz(_)` cannot be constructed due to `String: Copy` not holding.
Thus, the only inhabited variant would be `Foo::Bar` and that one is `singleton`.
