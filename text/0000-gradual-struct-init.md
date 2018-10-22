- Feature Name: `gradual_struct_init`
- Start Date: 2018-10-11
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

The main motivation of this RFC is 

TODO

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
    baz: T,
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
    baz: 24u8,
};
```

**_We propose_** that you should be able to also write (4):

```rust
let foo;
foo.bar = 42usize;
foo.baz = 24u8;

consume(foo); // OK!
```

### Immutable and mutable bindings

Note that in snippet (4), you are *not* mutating `foo`, `foo.bar`, or `foo.baz`.
These fields are only being initialized. If you however assign to `foo.bar`
twice as in (5):

```rust
let foo: Foo<u8>;

foo.bar = 42usize;
foo.baz = 24u8;

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
foo.baz = 24u8;

foo.bar = 43; // OK!

drop(foo);
```

### Definitive initialization and usage

Note that in both (4) and (6), you can only `drop(foo)`, `consume(foo)`,
or take references to `foo` because you have fully initialized `foo`.
Suppose instead that you wrote (7):

```rust
let mut foo;

foo.baz = 42;

consume(foo); // Error! `foo.bar` is not initialized.
```

The snippet in (7) would be *rejected* by the compiler because otherwise `foo.bar`
would be uninitialized and thus you would not have a valid `Foo`, thus causing
the type system to be unsound and the program to exhibit undefined behaviour.

In our proposal, it is also not legal to take a reference to `foo`, as with
`&mut foo` or `&foo` while parts of it isn't initialized. You also cannot
copy `foo` before it is fully initialized. Neither may you take references to,
move, or copy the parts of `foo` (in this case `foo.bar`) that are not yet
initialized. This extends to conditional initialization. **_The compiler will
allow you to write (8)_**:

```rust
let foo;

foo.baz = 42;

if random_bool() {
    foo.bar = 1;
} else {
    foo.bar = 2;
}

consume(foo); // OK!
```

The snippet in (8) is allowed because when `consume(foo)` is reached, `foo.bar`
is provably initialized no matter what branch is taken in the conditional.
However, the following snippet would be rejected because `foo.bar` may be
uninitialized in the `else` branch (9):

```rust
let foo;

foo.baz = 42;

if random_bool() {
    foo.bar = 1;
} else {
    // nothing here.
}

consume(foo); // ERROR! `foo.bar` not initialized in `else { .. }`.
```

The general condition here is that all fields must be *definitely initialized*.

### Partial initialization and referencing fields

We previously noted that you may not move or reference parts of `foo` that are
not yet initialized. The converse also applies. **_You may reference, move, or
copy parts the parts that are initialized while the whole type isn't._** (10):

```rust
let mut foo: Foo<u8>;

foo.bar = 1;

{
    let my_bar: &usize = &foo.bar; // OK!
}


{
    let my_mut_bar: &mut usize = &mut foo.bar; // OK!
}

drop(foo.bar); // OK!
```

### Immovable "self-referential" types

Because you are now able to construct `Foo<T>` piecemeal, this enables you to
**_reference parts of `Foo<T>` that have already been initialized in other parts
of `Foo<T>`_**. For example, you may write (11):

```rust
let foo: Foo<&usize>;

foo.bar = 42;       // <--
                    //   |
foo.baz = &foo.bar; // --| The compiler will ensure that this is dropped first
                    //     so that there are no dangling references.

// We can take a reference to `&foo`, that doesn't move `foo` anywhere:
drop(&foo);
```

So far so good. However, if you add a line at the end and try to move `foo`
itself, you will run into trouble (12):

```rust
// Error! We can't move `foo` because `foo.baz` borrows `foo.bar`.
consume(foo);
```

With the addition of (12), an error arises because if it were otherwise,
`foo.baz` would point to the old address of `foo.bar` which would now be
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
foo.baz = 24u8;

consume(foo); // OK!

foo.bar = 1;
foo.baz = 2;

consume(foo); // OK!
```

### Uninhabited types

Thus far, only `Foo<u8>` and `Foo<&usize>` have been used as the type of `foo`,
which means `T` in `Foo<T>` has always been inhabited.
Suppose instead that you defined an uninhabited type (14):

```rust
enum Void {}
```

Let's then write (16):

```rust
let foo: Foo<Void>;

foo.bar = 42;

foo.baz = ???;

consume(foo);
```

What do you replace `???` with? `foo.baz : Void` is an uninhabited type,
thus, there can be nothing you initialize `foo.baz` with or otherwise it
wouldn't be of an uninhabited type. Indeed, you cannot initialize `foo.baz`
and so you will never be able to form a valid value of type `Foo<Void>`.
This is to be expected, after all, you can't use the syntax
`Foo { bar: 42, baz: <value> }` for this either.

### Respecting privacy

Until now, all examples have been in contexts where the fields `bar` and `baz`
have been visible. In a case where the fields are not visible, you won't be able
to construct a value of `Foo<T>` with `Foo { bar: x, baz: y }`. The gradual
method is no different; to initialize the fields with the gradual method,
you must be able to refer to the fields. If you are not, then it follows that
you won't be able to construct a `Foo<T>`. In other words, the following would
not be a valid Rust program (17):

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

### No `Drop` types

Consider a type which implements `Drop` in some way (18):

```rust
struct SpecialDrop {
    field: T,
}

impl Drop for SpecialDrop {
    fn drop(&mut self) {

    }
}
```

Let's then use the gradual initialization mechanism in this RFC (19):

```rust
let sd: SpecialDrop;

sd.field = 
```



TODO

### Tuples also work

TODO

### Gradual initialization in fields

TODO

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
