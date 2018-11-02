- Feature Name: `gradual_struct_init`
- Start Date: 2018-10-24
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

Permit the gradual initialization of structs like so:

```rust
struct Point<T> { x: T, y: T }

let pt: Point<_>;
pt.x = 42;
pt.y = 24;

drop(pt);
```

# Motivation
[motivation]: #motivation

The RFC has two main motivations.
TODO: improve section prelude.

## Ergonomics and readability (TODO: concretize the title)

TODO: Felix and/or Niko: please fill in this section :)

## Immovable self-referential structs

Currently, if you want to encode an immovable self-referential type,
you'll have to use `Cell`s and `Option`s such as with:

```rust
#[derive(Debug)]
struct S<'a> {
    val: u32,
    ptr: Cell<Option<&'a u32>>,
}

fn main() {
    let s = S { val: 10, ptr: Cell::new(None), };
    s.ptr.set(Some(&s.val));
    
    println!("s: {:?}", s);
}
```

With gradual initialization, we can instead allow you to write:

```rust
#[derive(Debug)]
struct S<'a> {
    val: u32,
    ptr: &'a u32,
}

fn main() {
    let s: S;
    s.val = 10;
    s.ptr = &s.val;

    println!("s: {:?}", s);
}
```

You still cannot move `s` because `s.val` is borrowed in `s.ptr`;
However, the advantage here is that we avoid the indirection of `Option<T>`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

[structurally typed]: https://en.wikipedia.org/wiki/Structural_type_system
[nominally typed]: https://en.wikipedia.org/wiki/Nominal_type_system
[product type]: https://en.wikipedia.org/wiki/Product_type
[tuple type]: https://doc.rust-lang.org/nightly/reference/types.html#tuple-types
[struct type]: https://doc.rust-lang.org/nightly/reference/types.html#struct-types
[visibility]: https://doc.rust-lang.org/nightly/reference/visibility-and-privacy.html
[place]: https://doc.rust-lang.org/nightly/reference/expressions.html#place-expressions-and-value-expressions
[variable]: https://doc.rust-lang.org/nightly/reference/variables.html

## Definitions, concepts, and vocabulary

Let's first get some definitions out of the way; if you are familiar with the
Rust language, you can skip this section.

- A *local binding* or *[variable]* is a name for a component on the stack
  which holds a value. An example (1):

  ```rust
  let foo = 1;
  ```
  
  A binding may be *immutable*, which `foo` is, or *mutable*, which `bar` is (2):

  ```rust
  let mut foo = 1;
  ```

- An *uninhabited type* is a type which cannot be constructed since it has
  no values that inhabit the type. An example of such a type is:

  ```rust
  enum Void {}
  ```

- *Initialization* is the process of assigning a value to a binding or more
  generally *[place]* for the first time. Examples are in (1) and (2).

- *Reinitialization* is the process of assigning a value to a
  place that has already been initialized but then moved out of.
  Immutable bindings may not be reinitialized.

- A *[tuple type]* is denoted as `(T0, ..., Tn)`, where `Ti` are types.
  Tuples are *[structurally typed]* homogeneous *[product type]s*.

- A *[struct type]*, declared as `struct Name { field: T0, ... }` is a
  *[nominally typed]* homogeneous product type. Structs consist of a
  set of *fields* with types `Ti`. Given a place `x` of type `Name`
  it is possible to project out the value of a field with `x.field`.

  Each field of a struct type may optionally be assigned a *[visibility]*.
  If no visibility is specified, it is assumed to be private by default.
  Visibility is context dependent; a field may be visible in some context
  but not in some other.

  A struct type can also be of the tuple variety and is then
  declared with the syntax `struct Name(T0, ..., Tn);`.
  In that case, the field names are positional indices (`i ∈ 0 ... n`).

  A struct type may also be of the unit variety, e.g. `struct Name;` in which
  case it has no fields.

## Proposal

### The basics

Suppose that you have a `struct` (1):

```rust
struct Foo<T> {
    bar: usize,
    qux: T,
}
```

Suppose also that you have a function that consumes a `Foo<T>` (2):

```rust
fn consume<T>(_foo: Foo<T>) {}
```

Currently, if you want to make a `Foo` then you have to write (3):

```rust
let foo = Foo {
    bar: 42usize,
    qux: 24u8,
};
```

**_We propose_** that you should be able to also write (4):

```rust
let foo;
foo.bar = 42usize;
foo.qux = 24u8;

consume(foo); // OK!
```

### Immutable and mutable bindings

Note that in snippet (4), you are *not* mutating `foo`, `foo.bar`, or `foo.qux`.
These fields are only being initialized. If you however assign to `foo.bar`
twice as in (5):

```rust
let foo: Foo<u8>;

foo.bar = 42usize;
foo.qux = 24u8;

// Error! Second time you assign to `foo.bar` but `foo` is not a mutable binding!
foo.bar = 43;

drop(foo);
```

then the compiler will reject the program as ill-formed because
`foo` was not declared as a *mutable* binding with `mut foo`.
To remedy this, **_you may write_** (6):

```rust
let mut foo: Foo<u8>;

foo.bar = 42usize;
foo.qux = 24u8;

foo.bar = 43; // OK!

drop(foo);
```

### Definitive initialization and usage

Note that in both (4) and (6), you can only `drop(foo)`, `consume(foo)`,
or take references to `foo` because you have fully initialized `foo`.
Suppose instead that you wrote (7):

```rust
let mut foo;

foo.qux = 42;

consume(foo); // Error! `foo.bar` is not initialized.
```

The snippet in (7) would be *rejected* by the compiler because
otherwise `foo.bar` would be uninitialized and thus you would not
have a valid `Foo`, thus causing the type system to be unsound and
the program to exhibit undefined behaviour.

In our proposal, it is also not legal to take a reference to `foo`, as with
`&mut foo` or `&foo` while parts of it isn't initialized. You also cannot
copy `foo` before it is fully initialized. Neither may you take references to,
move, or copy the parts of `foo` (in this case `foo.bar`) that are not yet
initialized. This extends to conditional initialization. **_The compiler will
allow you to write (8)_**:

```rust
let foo;

foo.bar = 42;

if random_bool() {
    foo.qux = 1;
} else {
    foo.qux = 2;
}

// Nota bene:
// In this case foo.qux = if random_bool() { 1 } else { 2 };
// is preferable as a matter of style.

consume(foo); // OK!
```

The snippet in (8) is allowed because when `consume(foo)` is reached, `foo.qux`
is provably initialized no matter what branch is taken in the conditional.
However, the following snippet would be rejected because `foo.qux` may be
uninitialized in the `else` branch (9):

```rust
let foo;

foo.bar = 42;

if random_bool() {
    foo.qux = 1;
} else {
    // nothing here.
}

consume(foo); // ERROR! `foo.qux` not initialized in `else { .. }`.
```

The general condition here is that all fields must be *definitely initialized*.

### Partial initialization and referencing fields

We previously noted that you may not move or reference parts of `foo` that are
not yet initialized. The converse also applies. **_You may reference, move,
or copy the parts that are initialized while the whole type isn't._** (10):

```rust
let mut foo: Foo<u8>;

foo.qux = 1;

drop(foo.qux);

foo.bar = 2;

{
    let my_bar: &usize = &foo.bar; // OK!
} // <- Shared borrow ends here.

{
    let my_mut_bar: &mut usize = &mut foo.bar; // OK!
} // <- Mutable borrow ends here.

foo.qux = 1;
```

Note in particular that `foo.qux` is not initialized when we refer to `foo.bar`.
As long as you don't refer to `foo.qux` and `foo` while it is uninitialized this
is ok. You must also make sure that `foo` is definitively initialized at some
point even if you move out of some fields before `foo` goes out of scope.
Furthermore, it is not sufficient to simply initialize parts of `foo` at
various times; all of `foo` must be initialized at the *same* point.
These restriction apply to make adding fields cause type errors so that
you can refactor more easily.

### Immovable "self-referential" types

Because you are now able to construct `Foo<T>` piecemeal, this enables you
to **_reference parts of `Foo<T>` that have already been initialized in other
parts of `Foo<T>`_**. For example, you may write (11):

```rust
let foo: Foo<&usize>;

foo.bar = 42;       // <--
                    //   |
foo.qux = &foo.bar; // --/ The compiler will ensure that this is dropped first
                    //     so that there are no dangling references.

// We can take a reference to `&foo`, that doesn't move `foo` anywhere:
drop(&foo);
```

So far so good. However, if you add a line at the end and try to move `foo`
itself, you will run into trouble (12):

```rust
// Error! We can't move `foo` because `foo.qux` borrows `foo.bar`.
consume(foo);
```

With the addition of (12), an error arises because if it were otherwise,
`foo.qux` would point to the old address of `foo.bar` which would now be
invalid. This would be unsound due to the dangling reference.

### Reinitialization

Suppose that you have `consume`d `foo` as in (4).
In such a case, assuming `Foo<T>` is not `Copy`,
the binding `foo` becomes uninitialized because we have moved out of `foo`.
Before this RFC, you had to reinitialize `foo` with `foo = Foo { ... }`.
**_We propose that the gradual method should work for reinitialization as well_** (13):

```rust
let mut foo;
foo.bar = 42usize;
foo.qux = 24u8;

consume(foo); // OK!

foo.bar = 1;
foo.qux = 2;

consume(foo); // OK!
```

### Uninhabited types

Thus far, only `Foo<u8>` and `Foo<&usize>` have been used as the type of `foo`,
which means `T` in `Foo<T>` has always been inhabited.
Suppose instead that you defined an uninhabited type (14):

```rust
enum Void {}
```

Let's then write (15):

```rust
let foo: Foo<Void>;

foo.bar = 42;

foo.qux = ???;

consume(foo);
```

What do you replace `???` with? The field `qux` is of the uninhabited type `Void`.
Thus, there can be nothing you initialize `foo.qux` with or otherwise it
wouldn't be of an uninhabited type. Indeed, you cannot initialize `foo.qux`
and so you will never be able to form a valid value of type `Foo<Void>`.
This is to be expected, after all, you can't use the syntax
`Foo { bar: 42, qux: <value> }` for this either because you cannot produce a
`<value>` of type `Void`.

### Respecting privacy

Until now, all examples have been in contexts where the fields `bar` and `qux`
have been visible. In a case where the fields are not visible, you won't be able
to construct a value of `Foo<T>` with `Foo { bar: x, qux: y }`. The gradual
method is no different; to initialize the fields with the gradual method,
you must be able to refer to the fields. If you are not, then it follows that
you won't be able to construct a `Foo<T>`. In other words, the following would
not be a valid Rust program (16):

```rust
mod my_module {
    pub struct Point {
        x: u8,
        y: u8
    }
}

fn main() {
    let pt: Point;

    pt.x = 1; // Error! pt.x is not visible in this context.
    pt.y = 2; // Error! pt.y isn't visible either.

    drop(pt);
}
```

[RFC 2008]: https://github.com/rust-lang/rfcs/blob/master/text/2008-non-exhaustive.md

Per [RFC 2008], the type system will also respect `#[non_exhaustive]`
annotations with respect to gradual initialization. That is, if you mark a
`struct` with `#[non_exhaustive]` in one crate, then a downstream crate won't
be able to use the gradual syntax to make a value of the type.

### No `Drop` types

Consider a type which implements `Drop` by printing something (17):

```rust
struct SpecialDrop {
    alpha: u8,
    beta: u16,
    gamm: u32,
}

impl Drop for SpecialDrop {
    fn drop(&mut self) {
        println!("Dropping in a special way!");
    }
}
```

Let's then use the gradual initialization mechanism in this RFC (18):

```rust
fn foo() {
    let sd: SpecialDrop;

    sd.alpha = 1;
    sd.beta = 2;

    some_action_that_can_panic();

    sd.gamma = 3;
}
```

If you don't have the definition of `SpecialDrop` in your near vicinity,
it can become difficult to know whether the destructor, and thus the side effect
in `SpecialDrop::drop` will run or when it will run. In (18), to know if and
when the destructor will run, you would need to know that `SpecialDrop` has the
fields `alpha`, `beta`, and `gamma` and that `gamma` hasn't been initialized yet.

Thus, while it would be sound to allow it, if a type implements `Drop`,
the compiler will reject gradual initialization.

### Tuples and tuple structs also work

Hitherto, we have only gradually initialized `struct`s with named fields.
However, this RFC makes no distinction between those and other forms of
heterogeneous product types. In other words, **_you can also initialize
tuple structs and normal tuples with the gradualist syntax._**
That is, you can write (19):

```rust
struct Point(f32, f32);

let pt: Point;

pt.0 = 42.24;
pt.1 = 13.37;

drop(pt);
```

as well as (20):

```rust
fn consume_pair((x, y): (u8, u8)) { ... }

let pt;
pt.0 = 42;
pt.1 = 24;

consume_pair(pt);
```

### Gradual initialization in fields

What we have said so far does not just apply to local bindings.
You can use the gradualist mechanism in fields as well.
For example, you may use the gradual notation as a sort of builder notation (21):

```rust
struct Config {
    window: WindowConfig,
    runtime: RuntimeConfig,
}

struct WindowConfig {
    height: usize,
    width: usize,
}

struct RuntimeConfig {
    threads: usize,
    max_memory: usize,
}

fn make_config() -> Config
    let cfg;

    cfg.window.width = 1920;
    cfg.window.height = 1080;
    // cfg.window is definitely initialized now.

    cfg.runtime.threads = 8;
    cfg.runtime.max_memory = 1024;
    // cfg.runtime is definitely initialized now.
    // therefore cfg is also.

    cfg
}
```

The usual same rules with respect to eventual definite initialization,
privacy, uninhabited types, prohibitions on `Drop` types, etc. apply
here as well.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Grammar

There are no changes to the grammar.

## Semantics

Let:

- `n` range over the natural numbers.

- `Γ` be a well-formed typing environment.

- `σ` be a well-formed type in `Γ`.

- `typeof(e)` denote the type `σ` of an expression `e` in `Γ`.

- `downstream(σ)` be a predicate holding iff `σ` is not defined in the current
  crate.

- `non_exhaustive(σ)` be a predicate holding iff:
   - `σ` has `#[non_exhaustive]` directly applied to it,
   - `downstream(σ)` holds.

- `visible(σ, f)` be a predicate holding iff for `σ`,
  the field `f` is visible in `Γ`.

- `implemented(σ, τ)` be a predicate holding iff `σ` implements the trait `τ`.

- `D` range over valid identifiers.

- `G` denote the generic parameters on a data type definition.

- `P` be all the product types in `σ`, namely:

    ```rust
    P : struct D ; // Unit data types.
      | struct D < G > ( σ_0, ..., σ_n ); // Tuple struct data types.
      | struct D < G > { f_0: σ_0, ..., f_n: σ_n } // Named field struct data types.
      | ( σ_0, ..., σ_n ) // Structural tuple types.
      ;
    ```

    (If structural records were ever to be added they would also be included.)

- `PG = { σ ∈ P | implemented(P, Drop) ≡ ⊥ ∧ non_exhaustive(σ) ≡ ⊥ }`

- `p` denote a place expression.

- `m(p)` be a predicate holding iff `p` is a mutable place.

Given a place `p`, iff `typeof(p) ∈ PG`, then `p` may be *gradually* initialized.
This means that:

1. an uninitialized field `f` of `p`, whether `f` be a numbered tuple
   index or a named field, may be initialized with an assignment expression
   `p.f = value_expr`, if `visible(typeof(p), f)`.

2. iff `m(p)`, then `p.f` may be assigned to more than once.
   Otherwise, it may only be assigned to once in any path in the control graph.

3. a field or sub-field `f` of `p` may be moved out of
   (unless `implemented(typeof(p), Drop)` or if `p` or `p.f` is borrowed) or
   borrowed (including mutably iff `m(p)`) if `p.f` is definitely initialized.

4. the place `p` is valid, meaning that it may be moved or referenced,
   once all of its fields have been definitely initialized.

5. before `p` goes out of scope, `p` must have been definitively initialized
   at some point at least once.

# Drawbacks
[drawbacks]: #drawbacks

The usual drawbacks with respect to having more ways to do it applies,
in particular, it could lead to decision fatigue between the struct literal
approach and the gradualist approach.

Another drawback is that we don't provide an similar way for enums to be
gradually constructed.

Furthermore, the gradual approach could be considered more imperative than
functional which some functional programmers may not prefer and this would
increase the likelihood of that in the ecosystem. However, the author of
this proposal is a functional programmer (of the Haskell variety) and they
do like what they are proposing.

Yet another drawback is that it could become harder to see when a value is
constructed.

Perhaps the most major drawback in this proposal is the complexity increase
in the type system. This will make alternative (to `rustc`) Rust compilers
more difficult to write.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

With respect to the current design, the choice of surface syntax is quite
straightforward. We simply extend the assignment syntax that already exists.

However, some tweaks in various directions could be made to the RFC:

1. We could limit gradual initialization to top level bindings.
   This would limit the power and is seemingly an arbitrary restriction
   that could cause surprises.

1. We could require that a gradually initialized place be definitively
   initialized before allowing references or moving out of parts.
   This would mean that we could not write:

   ```rust
   let foo: Foo;
   foo.bar = 1;
   foo.baz = &foo.bar;
   ```

   That would remove one of the uniquely useful things about the
   proposed syntax.

3. We could limit this to nominal types or just to types with named fields.
   While that is possible, it complicates understanding of the type system.

4. We could allow `Drop` types to be gradually initialized.
   This is discussed in the [unresolved questions][unresolved-questions].

# Prior art
[prior-art]: #prior-art

None to our knowledge.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. Should we lift the requirement that each field must be definitively
   initialized at least once before a gradually initialized place goes
   out of scope?

   We have imposed this restriction to make sure that adding a new field
   causes a type error in all cases; if we lifted the restriction,
   that would no longer hold.

2. Should we allow gradual initialization for types implementing `Drop`?

   While this might make the language more consistent and uniform,
   it might also become too much magic. In particular, it becomes less
   clear when destructors will run and understanding when it happens might
   require tracking "more boolean variables" in your head.
   For example, if we write:

   ```rust
   let foo;
   foo.bar = <expr>;
   foo.baz = <expr>;
   foo.quux = <expr>;
   may_panic();

   // a few lines later...
   foo.wibble = <expr>;
   ```

   Then we have to know that `foo` has 4 fields to know that `may_panic()`
   will not run the destructor of `typeof(foo)`.

   It is however unclear if this would be a problem in practice and
   it could be solved by privacy.

   If we do however allow `Drop` types to be gradually initialized,
   then we should also lift the restriction on moving out of `Drop`
   types to make things consistent.

   If we want to allow `Drop` types, then the restriction in 1.
   is helpful because there's less to track in your head.

3. If we permit structural records, and if the user writes:

   ```rust
   let foo;
   foo.bar = 1;
   foo.baz = true;
   foo.qux = "hello";
   drop(foo);
   ```

   Should then `foo`, if not otherwise constrained, be inferred to the type?:

   ```rust
   { bar: i32, baz: bool, qux: &'static str }
   ```

   This could be quite ergonomic, but it could also be too much magic.

# Future possibilities
[future-possibilities]: #future-possibilities

[RFC 2534]: https://github.com/rust-lang/rfcs/pull/2534

Partially initialized objects is not exposed to the user in the type system.
We could allow this with `&uninit T` references. That would allow uninitialized
fields to pass into functions which can then initialize them.
See [RFC 2534] for a discussion on `&uninit T`.
