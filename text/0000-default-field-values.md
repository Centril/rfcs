- Feature Name: `default_field_values`
- Start Date: 2018-12-17
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Allow `struct` definitions and `enum` variants to provide default values for
individual fields and thereby allowing those to be omitted from initializers.
When deriving `Default`, the provided values will then be used. For example:

```rust
#[derive(Default)]
struct ExprType {
    expr: Box<Expr>,
    ty: Box<Ty>,
    attrs: Vec<Attribute> = Vec::new(),
}
```

An attribute `#[default]`, usable on `enum` variants, is also introduced,
thereby allowing enums to work with `#[derive(Default)]`.

```rust
#[derive(Default)]
enum Expr {
    #[default]
    Field {
        base: Box<Expr>,
        member: Ident,
        attrs: Vec<Attribute> = Vec::new(),
    },
    // other variants...
}

let ascribe = ExprType { expr: make_expr(), ty, .. };
let field = Expr::Field { base, member, .. };
```

# Motivation
[motivation]: #motivation

## Boilerplate reduction

### For `struct`s

[FRU]: https://doc.rust-lang.org/1.31.0/book/ch05-01-defining-structs.html#creating-instances-from-other-instances-with-struct-update-syntax

Rust allows you to create an instance of a `struct` using the struct literal
syntax `Foo { bar: expr, baz: expr }`. To do so, all fields in the `struct`
must be assigned a value. This makes it inconvenient to create large `struct`s
whose fields usually receive the same values. Struct literals also cannot be
used to initialize `struct`s with fields which are inaccessible due to privacy.
*[Functional record updates (FRU)][FRU]* can reduce noise when a `struct` derives
`Default`, but are also invalid when the `struct` has inaccessible fields.

To work around these shortcomings, you can create constructor functions:

```rust
struct Foo {
    alpha: &'static str,
    beta: bool,
    gamma: i32,
}

impl Foo {
    /// Constructs a `Foo`.
    fn new(alpha: &'static str, gamma: i32) -> Self {
        Self {
            alpha,
            beta: true,
            gamma
        }
    }
}

let foo = Foo::new("Hello", 42);
```

[`process::Command`]: https://doc.rust-lang.org/stable/std/process/struct.Command.html

The problem with a constructor is that you need one for each combination
of fields a caller can supply. To work around this, you can use builders,
such as [`process::Command`] in the standard library.
Builders enable more advanced initialization, but require additional boilerplate.

As a part of this RFC, a solution is proposed for improving `struct` literal
ergonomics such that they can be used for `struct`s with private fields,
and to reduce boilerplate for simple scenarios (as seen in the [summary]).
This is achieved by letting callers omit fields from initialization when
a default is specified for that field.

Field defaults allow a caller to initialise a `struct` with default values
without needing builders or a constructor function:

```rust
struct Foo {
    alpha: &'static str = "Hello",
    beta: bool = true,
    gamma: i32 = 42,
}

// Overriding a single field default:
let foo = Foo {
    beta: false,
    ..
};

// Override multiple field defaults:
let foo = Foo {
    alpha: "Overriden",
    gamma: 1,
    ..
};

// Override no field defaults:
let foo = Foo { .. };
```

This can be useful when implementing `Default` doesn't make sense and
where many fields have defaults but one does not. For example:

```rust
struct LaunchCommand {
    cmd: String,
    args: Vec<String> = vec![],
    some_special_setting: Option<FancyConfig> = None,
    setting_most_people_will_ignore: Option<FlyMeToTheMoon> = None,
}
```

Here, it is not sensible to give `cmd` a default since all cases would need
to customize the field and thus `Default` shouldn't be implemented.
However, it does make sense to default at least the latter two fields
since the `None` value's fit most use cases.

### `PhantomData` ergonomics

[nomicon]: https://doc.rust-lang.org/nightly/nomicon/phantom-data.html

Say that you want to define our own `Vec<T>` type. As the [nomicon] says,
you need to use [PhantomData] to make sure that drop-checking is sound like so:

```rust
use std::marker::PhantomData;

struct Vec<T> {
    data: *const T, // *const for variance!
    len: usize,
    cap: usize,
    _marker: PhantomData<T>,
}
```

You then go on to define some operations for `Vec<T>`:

```rust
impl<T> Vec<T> {
    pub fn new() -> Self {
        Self {
            // other fields...
            _marker: PhantomData,
        }
    }

    pub fn with_capacity(capacity: usize) -> Self {
        Self {
            // other fields...
            _marker: PhantomData,
        }
    }
}
```

Here, `_marker: PhantomData` is just provided to satisfy the compiler.
`PhantomData` carries no useful information since it is a ZST.
Therefore, the result of having to add a field of type `PhantomData`
is a regression to ergonomics throughout. With default values you can
improve the situation slightly and provide the value in a single place:

```rust
use std::marker::PhantomData;

struct Vec<T> {
    data: *const T, // *const for variance!
    len: usize,
    cap: usize,
    _marker: PhantomData<T> = PhantomData,
}
```

You then go on to define some operations for `Vec<T>`:

```rust
impl<T> Vec<T> {
    pub fn new() -> Self {
        Self {
            // other fields...
            ..
        }
    }

    pub fn with_capacity(capacity: usize) -> Self {
        Self {
            // other fields...
            ..
        }
    }
}
```

### Defaults for `enum`s

So far, you've have seen how ergonomics were improved for `struct`s.
For `enum`s, FRU cannot be used as for structs and neither can
`#[derive(Default)]` be used for each variant or the entire enum.
Instead, at best, a constructor function can be offered for each variant.
For a large enum which for example encodes an expression language such
this could become quite a lot of variants.

By allowing each individual variant of an `enum` to have default values,
more profound improvements to ergonomics than for `struct`s can be achieved.
For example, you may write:

```rust
enum Foo<'a> {
    Bar {
        alpha: u8 = 42,
        beta: &'a str = "beta's default value",
    },
    Baz {
        gamma: Vec<u8> = Vec::new(),
        delta: f32
    }
}
```

and then these defaults may be used like so:

```rust
let a = Foo::Bar { .. };
let b = Foo::Bar { alpha: 1, .. };
let c = Foo::Bar { beta: "another beta", .. };

let d = Foo::Baz { delta: 1.0, .. };
```

## `#[derive(Default)]` in more cases

The `#[derive(..)]` ("custom derive") mechanism works by defining procedural
*macros*. Because they are macros, these operate on abstract *syntax* and
don't have more information available. Therefore, when you `#[derive(Default)]`
on a data type definition as with:

```rust
#[derive(Default)]
struct Foo {
    bar: u8,
    baz: String,
}
```

it only has the immediate "textual" definition available to it.

Because Rust currently does not have an in-language way to define default values,
you cannot `#[derive(Default)]` in the cases where you are not happy with the
natural default values that each field's type provides. By extending the syntax
of Rust such that default values can be provided, `#[derive(Default)]` can be
used in many more circumstances and thus boilerplate is further reduced.

Currently, `#[derive(Default)]` is not usable for `enum`s. To rectify this
situation and to allow enum variants to take advantage of their newfound
ability to have default field values, a `#[default]` attribute is introduced
that can be attached to variants. This allows you to use `#[derive(Default)]`
on enums wherefore you can now write:

```rust
// From rocket:

#[derive(Default)]
enum Referrer {
    #[default]
    NoReferrer,
    NoReferrerWhenDowngrade,
    ...
}

#[derive(Default)]
enum ExpectCt {
    #[default]
    Enforce {
        duration: Duration = Duration::days(30)
    },
    ...
}

// From git2:

#[derive(Default)]
enum FileFavor {
    #[default]
    Normal,
    Ours,
    Theirs,
    Union,
}
```

Note that while `#[default]` makes field defaults more useful and is useful
on its own, it is technically orthogonal in that it can be extracted out of
this RFC if need be. However, since it completes the picture, it is proposed
together with field defaults.

## Clearer documentation and more local reasoning

Providing good defaults when such exist is part of any good design that makes
a physical tool, UI design, or even data-type more ergonomic and easily usable.
However, that does not mean that the defaults provided can just be ignored
and that they need not be understood. This is especially the case when you
are moving away from said defaults and need to understand what they were.
Furthermore, it is not too uncommon to see authors writing in the documentation
of a data-type that a certain value is the default.

All in all, the defaults of a data-type are therefore important properties.
By encoding the defaults right where the data-type is defined gains can be made
in terms of readability particularly wrt. the *skimmability* of code.
In particular, it is easier to see what the defaults are if you can directly
look at the `rustdoc` page and read:

```rust
#[derive(Default)]
enum Foo {
    #[default]
    Bar {
        alpha: u8 = 42,
    },
    Baz {
        beta: u16 = 24,
        gamma: bool,
    }
}
```

This way, you do not need to open up the code of the `Default` implementation
to see what the values encoded are.

## Usage by other `#[derive(..)]` macros

[`serde`]: https://serde.rs/attributes.html

Custom derive macros exist that have a notion of or use default values.

### `serde`

For example, the [`serde`] crate provides a `#[serde(default)]` attribute that
can be used on `struct`s, and fields. This will use the field's or type's
`Default` implementations. This works well with field defaults; `serde` can
either continue to rely on `Default` implementations in which case this RFC
facilitates specification of field defaults; or it can directly use the default
values provided in the type definition.

### `structopt`

Another example is the `structopt` crate with which you can write:

```rust
#[derive(Debug, StructOpt)]
#[structopt(name = "example", about = "An example of StructOpt usage.")]
struct Opt {
    /// Set speed
    #[structopt(short = "s", long = "speed", default_value = "42")]
    speed: f64,
    ...
}
```

By having default field values in the language, `structopt` could let you write:

```rust
#[derive(Debug, StructOpt)]
#[structopt(name = "example", about = "An example of StructOpt usage.")]
struct Opt {
    /// Set speed
    #[structopt(short = "s", long = "speed")]
    speed: f64 = 42,
    ...
}
```

### `derive_builder`

[`derive_builder`]: https://docs.rs/derive_builder/0.7.0/derive_builder/#default-values

A third example comes from the crate [`derive_builder`]. As the name implies,
you can use it to `#[derive(Builder)]`s for your types. An example is:

```rust
#[derive(Builder, Debug, PartialEq)]
struct Lorem {
    #[builder(default = "42")]
    pub ipsum: u32,
}
```

### Conclusion

As seen in the previous sections, rather than make deriving `Default`
more magical, by allowing default field values in the language,
user-space custom derive macros can make use of them.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Providing field defaults

Consider a data-type such as (1):

```rust
pub struct Probability {
    value: f32,
}
```

You'd like encode the default probability value to be `0.5`;
With this RFC now you can provide such a default directly where `Probability`
is defined like so (2):

```rust
pub struct Probability {
    value: f32 = 0.5,
}
```

Having done this, you can now construct a `Probability` with a struct
initializer and leave `value` out to use the default (3):

```rust
let prob = Probability { .. };
```

## Deriving `Default`

Previously, you might have instead implemented the `Default` trait like so (4):

```rust
impl Default for Probability {
    fn default() -> Self {
        Self { value: 0.5 }
    }
}
```

You can now shorten this to (5):

```rust
impl Default for Probability {
    fn default() -> Self {
        Self { .. }
    }
}
```

However, since you had specified `value: f32 = 0.5` in the definition of
`Probability`, you can take advantage of that to write the more simpler
and more idiomatic (6):

```rust
#[derive(Default)]
pub struct Probability {
    value: f32 = 0.5,
}
```

Having done this, a `Default` implementation equivalent to the one in (5)
will be generated for you.

## More fields

As you saw in the [summary], you are not limited to a single field and all
fields need not have any defaults associated with them. Instead, you can freely
mix and match. Given the definition of `LaunchCommand` from the [motivation] (7):

```rust
struct LaunchCommand {
    cmd: String,
    args: Vec<String> = vec![],
    some_special_setting: Option<FancyConfig> = None,
    setting_most_people_will_ignore: Option<FlyMeToTheMoon> = None,
}
```

you can omit all fields but `cmd` (8):

```rust
let ls_cmd = LaunchCommand {
    cmd: "ls".to_string(),
};
```

You can also elect to override the provided defaults (9):

```rust
let ls_cmd2 = LaunchCommand {
    cmd: "ls".to_string(),
    args: vec!["-lah".to_string()],
    some_special_setting: make_special_setting(),
    // setting_most_people_will_ignore is still defaulted.
};
```

## Default fields values are [`const` context]s

[`const` context]: https://github.com/rust-lang-nursery/reference/blob/66ef5396eccca909536b91cad853f727789c8ebe/src/const_eval.md#const-context

As you saw in (7), `Vec::new()`, a function call, was used.
However, this assumes that `Vec::new` is a *`const fn`*. That is, when you
provide a default value `field: Type = value`, the given `value` must be a
*constant expression* such that it is valid in a [`const` context].
Therefore, you cannot write something like (10):

```rust
fn launch_missilies() -> Result<(), LaunchFailure> {
    authenticate()?;
    begin_launch_sequence()?;
    ignite()?;
    Ok(())
}

struct BadFoo {
    bad_field: u8 = {
        launch_missilies().unwrap();
        42
    },
}
```

Since launching missiles interacts with the real world and has *side-effects*
in it, it is not possible to do that in a `const` context since it may violate
deterministic compilation.

## Privacy interactions

In examples thus far, initialization literal expressions `Foo { bar, .. }` have
been used in the same module as where the constructed type was defined in.

Let's now consider a scenario where this is not the case (11):

```rust
pub mod foo {
    pub struct Alpha {
        beta: u8 = 42,
        gamma: bool = true,
    }
}

mod bar {
    fn baz() {
        let x = Alpha { .. };
    }
}
```

Despite `foo::bar` being in a different module than `foo::Alpha` and despite
`beta` and `gamma` being private to `foo::bar`, a Rust compiler will accept
the above snippet. It is legal because when `Alpha { .. }` expands to
`Alpha { beta: 42, gamma: true }`, the fields `beta` and `gamma` are considered
in the context of `foo::Alpha`'s *definition site* rather than `bar::baz`'s
definition site.

## `#[non_exhaustive]` interactions

[RFC 2008]: https://github.com/rust-lang/rfcs/blob/master/text/2008-non-exhaustive.md#structs-1

[RFC 2008] introduced the attribute `#[non_exhaustive]` that can be placed
on `struct`, `enum`, and `enum` variants. The RFC notes that upon defining
a `struct` in *crate A* such as (12):

```rust
#[non_exhaustive]
pub struct Config {
    pub width: u16,
    pub height: u16,
}
```

it is **_not_** possible to initialize a `Config` in a different *crate B* (13):

```rust
let config = Config { width: 640, height: 480 };
```

This is forbidden when `#[non_exhaustive]` is attached because the purpose of
the attribute is to permit adding fields to `Config` without causing a
breaking change. However, the RFC goes on to note that you can pattern match
if you allow for the possibility of having fields be ignored with `..` (14):

```rust
let Config { width, height, .. } = config;
```

Because this RFC gives a way to have default field values, you can now simply
invert the pattern expression and initialize a `Config` like so (15):

```rust
let config = Config { width, height, .. };
```

## Defaults for `enum`s

The ability to give fields default values is not limited to `struct`s.
Fields of `enum` variants can also be given defaults (16):

```rust
enum Ingredient {
    Tomato {
        color: Color = Color::Red,
        taste: TasteQuality,
    },
    Onion {
        color: Color = Color::Yellow,
    }
}
```

Given these defaults, you can then proceed to initialize `Ingredient`s
as you did with `struct`s (17):

```rust
let sallad_parts = vec![
    Ingredient::Tomato { taste: Yummy, .. },
    Ingredient::Tomato { taste: Delicious, color: Color::Green, },
    Ingredient::Onion { .. },
];
```

Note that `enum` variants have public fields and in today's Rust,
this cannot be controlled with visibility modifiers on variants.

Furthermore, when `#[non_exhaustive]` is specified directly on an `enum`,
it has no interaction with the defaults values and the ability to construct
variants of said enum. However, as specified by [RFC 2008], `#[non_exhaustive]`
is permitted on variants. When that occurs, the behaviour is the same as if
it had been attached to a `struct` with the same fields.

### Specifying the `#[default]` variant

The ability to add default values to fields of `enum` variants does not mean
that you can suddenly `#[derive(Default)]` on the enum. A Rust compiler will
still have no idea which variant you intended as the default.
This RFC adds the ability to mark one variant with `#[default]` (18):

```rust
#[derive(Default)]
enum Ingredient {
    Tomato {
        color: Color = Color::Red,
        taste: TasteQuality,
    },
    Onion {
        color: Color = Color::Yellow,
    },
    #[default]
    Lettuce,
}
```

Now the compiler does know that `Ingredient::Lettuce` should be considered
the default and will accordingly generate an appropriate implementation of
`Default for Ingredient` (19):

```rust
impl Default for Ingredient {
    fn default() -> Self {
        Ingredient::Lettuce
    }
}
```

Note that after any `#[cfg_attr(..)]`-stripping has occurred, it is an error
to have `#[default]` specified on more than one variant.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Field default values

### Grammar

Let the grammar of record fields in `struct`s and `enum` variants be defined
like so (in the `.lyg` notation):

```rust
RecordField = attrs:OuterAttr* vis:Vis? name:IDENT ":" ty:Type;
```

Then, `RecordField` is changed into:

```rust
RecordField = attrs:OuterAttr* vis:Vis? name:IDENT ":" ty:Type { "=" def:Expr }?;
```

Further, given the following partial definition for the expression grammar:

```rust
Expr = attrs:OuterAttr* kind:ExprKind;
ExprKind =
  | ...
  | Struct:{ path:Path "{" attrs:InnerAttr* fields:StructExprFieldsAndBase "}" }
  ;

StructExprFieldsAndBase =
  | Fields:{ fields:StructExprField* % "," ","? }
  | Base:{ ".." base:Expr }
  | FieldsAndBase:{ fields:StructExprField+ % "," "," ".." base:Expr }
  ;
StructExprField = attrs:OuterAttr* kind:StructExprFieldKind;
StructExprFieldKind =
  | Shorthand:IDENT
  | Explicit:{ field:FieldName ":" expr:Expr }
  ;
```

the rule `StructExprFieldsAndBase` is extended with:

```rust
StructExprFieldsAndBase =| FieldsAndDefault:{ fields:StructExprField+ % "," "," ".." };
```

### Static semantics

#### Defining defaults

Given a `RecordField` where the default is specified, i.e.:

```rust
RecordField = attrs:OuterAttr* vis:Vis? name:IDENT ":" ty:Type "=" def:Expr;
```

all the following rules apply when type-checking:

1. The expression `def` must be a constant expression.

2. The expression `def` must unify with the type `ty`.

When lints check attributes such as `#[allow(lint_name)]` are placed on a
`RecordField`, it also applies to `def` if it exists.

#### Initialization expressions

Given an expression `τ` of the form (e.g. `Foo { x: 1, .. }`):

```rust
path:Path "{" attrs:InnerAttr* fields:StructExprField+ % "," "," ".." "}"
```

all the following rules apply when type-checking:

1. All `fields` in `τ` mentioned must exist in the `struct` or `enum` variant
   referenced by `path`. No field in `fields` may be mentioned twice.

2. For each `field: expr` pair in `fields` of `τ`,
   `expr` must unify with the type of `field` in the `struct`
   or `enum` variant referenced by `path`.

3. For each `field: expression` pair in `fields` of `τ`,
   `field` must be visible in the context which `τ` occurs in.

4. For each field `f` in `τ` *not* mentioned in `fields`, a default value must
   exist for `f` in the `struct` or `enum` variant referenced by `path`.

---------------------

**Lemma 1:** If `τ` is of a type `σ` where `σ: Sync`, and if all `field: expr`
has an `expr` component which is a valid constant expression, then `τ` is
also a valid constant expression.

**Proof:** It trivially holds that if all `expr`s in `τ` are
constant expressions and all default values for fields omitted
are also constant expressions, then `τ` is a constant expression.

**Lemma 2:** If an expression `x` of form `Path { .. }` is of a type `σ` where
`σ: Sync`, then `x` is a constant expression.

**Proof:** `x` is a special case of `τ` with all fields according to the default
values. Since there are no explicitly provided fields, by Lemma 1 it holds.

### Dynamic semantics

Given a well-formed expression `τ` according to the aforementioned form,
the operational semantics is the same as the semantics of an expression
`path { new_fields }` with `new_fields = fields ∪ default_fields` where:

+ `∪` computes the left-biased union where equal elements are determined
  solely on equality of field names.

+ `fields` are the set of explicitly given `field: expr` pairs.

+ `default_fields` are the fields in `τ`'s data-type or variant which
  have provided default values.

## `#[default]` on `enum`s

A built-in attribute `#[default]` is provided the compiler and may be legally
placed solely on `enum` variants. The attribute has no semantics on its own.
Placing the attribute on anything else will result in a compilation error.
Furthermore, if after `cfg`-stripping and macro expansion is done,
the attribute occurs on more than one variant of the same `enum` data-type,
this will also result in a compilation error.

## `#[derive(Default)]`

When generating an implementation of `Default` for a `struct` named `$s` on
which `#[derive(Default)]` has been attached, the compiler will omit all fields
which have default values provided in the `struct`. The the associated function
`default` shall then be defined as (where `$f_i` denotes a vector of the fields
of `$s`):

```rust
fn default() -> Self {
    $s { $f_i: Default::default(), .. }
}
```

Placing `#[derive(Default)]` on an `enum` named `$e` is permissible iff that
enum has some variant `$v` with `#[default]` on it. In that event, the compiler
shall generate an implementation of `Default` where the function `default`
is defined as (where `$f_i` denotes a vector of the fields of `$e::$v`):

```rust
fn default() -> Self {
    $e::$v { $f_i: Default::default(), .. }
}
```

# Drawbacks
[drawbacks]: #drawbacks

The usual drawback of increasing the complexity of the language applies.
However, the degree to which complexity is increased is not substantial.

In particular, the syntax `Foo { .. }` mirrors the identical and already
existing pattern syntax. This makes the addition of `Foo { .. }` at worst
low-cost and potentially cost-free.

The main complexity comes instead from introducing `field: Type = expr`.
However, as seen in the [prior-art], there are several widely-used languages
that have a notion of field / property / instance-variable defaults.
Therefore, the addition is intuitive and thus the cost is seen as limited.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Besides the given [motivation], there are some specific design choices
worthy of more in-depth discussion, which is the aim of this section.

## Provided associated items as precedent

While Rust does not have any support for default values for fields or for
formal parameters of functions, the notion of defaults are not foreign to Rust.

Indeed, it is possible to provide default function bodies for `fn` items in
`trait` definitions. For example:

```rust
pub trait PartialEq<Rhs: ?Sized = Self> {
    fn eq(&self, other: &Rhs) -> bool;

    fn ne(&self, other: &Rhs) -> bool { // A default body.
        !self.eq(other)
    }
}
```

In traits, `const` items can also be assigned a default value. For example:

```rust
trait Foo {
    const BAR: usize = 42; // A default value.
}
```

Thus, to extend Rust with a notion of field defaults is not an entirely alien
concept.

## Pattern matching follows construction

[dual]: https://en.wikipedia.org/wiki/Duality_(mathematics)

In mathematics there is a notion of one thing being the *[dual]* of another.
Loosely speaking, duals are often about inverting something.
In Rust, one example of such an inversion is expressions and patterns.

Expressions are used to *build up* and patterns *break apart*;
While it doesn't hold generally, a principle of language design both in Rust
and other languages with with pattern matching has been that the syntax for
patterns should, to the extent possible, follow that of expressions.

For example:

+ You can match on or build up a struct with `Foo { field }`.
  For patterns this will make `field` available as a binding
  while for expressions the binding `field` will be used to build a `Foo`.

  For a tuple struct, `Foo(x)` will work both for construction and matching.

+ If you want to be more flexible, both patterns and expressions permit
  `Foo { field: bar }`.

+ You can use both `&x` to dereference and bind to `x` or
  construct a reference to `x`.

+ An array can be constructed with `[a, b, c, d]` and the same is a valid
  pattern for destructuring an array.

The reason why matching should follow construction is that it makes languages
easier to understand; you simply learn the expression syntax and then reuse
it to run the process in reverse.

In some places, Rust could do a better job than it currently does of adhering to
this principle. In this particular case, the pattern syntax `Foo { a, b: c, .. }`
has no counterpart in the expression syntax. This RFC rectifies this by
permitting `Foo { a, b: c, .. }` as an expression syntax; this is identical
to the expression syntax and thus consistency has been gained.

However, it is not merely sufficient to use the same syntax for expressions;
the semantics also have to be similar in kind for things to work out well.
This RFC argues that this is the case because in both contexts, `..` indicates
something partially ignorable is going on: "I am *destructuring*/*constructing*
this struct, and by the way there are some more fields I don't care about
*and let's* drop those* / *and let's fill in with default values*".
In a way, the use of `_` to mean both a catch-all pattern and type / value
placeholder is similar to `..`; in the case of `_` both cases indicate something
unimportant going on. For patterns, `_` matches everything and doesn't give
access to the value; for types, the placeholder is just an unbounded inference
variable.

## On privacy

This RFC specifies that when field defaults are specified, the desugaring
semantics of `Foo { .. }` spans `field: default_value` with the context in
which the type is defined as opposed to where `Foo { .. }` occurs.
In other words, the following is valid:

```rust
pub mod alpha {
    pub struct Foo {
        field: u8 = 42,
    }
}

use alpha::Foo;

mod beta {
    // `field` isn't visible here.
    fn bar() -> Foo {
        Foo { .. } // OK!
    }
}
```

By permitting the above snippet, you are able to construct a default value
for a type more ergonomically with `Foo { .. }`. Since it isn't possible for
functions in `beta` to access `field`'s value, the value `42` or any other
remains at all times private to `alpha`. Therefore, privacy, and by extension
soundness, is preserved.

If a user wishes to keep other modules from constructing a `Foo` with
`Foo { .. }` they can simply add, or keep, one private field without a default.
Situations where this can be important include those where `Foo` is some token
for some resource and where fabricating a `Foo` may prove dangerous or worse
unsound. This is however no different than carelessly adding `#[derive(Default)]`.

## On `const` contexts

To recap, the expression a default value is computed with must be constant one.
There are many reasons for this restriction:

+ If *determinism* is not enforced, then just by writing the following snippet,
  the condition `x == y` may fail:

  ```rust
  let x = Foo { .. };
  let y = Foo { .. };
  ```

  This contributes to surprising behaviour overall.

  Now you may object with an observation that if you replace `Foo { .. }` with
  `make_foo()` then a reader no longer know just from the syntactic form whether
  `x == y` is still upheld. This is indeed true. However, there is a general
  expectation in Rust that a function call may not behave deterministically.
  Meanwhile, for the syntactic form `Foo { .. }` and with default values,
  the whole idea is that they are something that doesn't require close attention.

+ The broader class of problem that non-determinism highlights is that of
  *side*-effects. These effects wrt. program behaviour are prefixed with
  *"side"* because they happen without being communicated in the type system
  or more specifically in the inputs and outputs of a function.

  In general, it is easier to do formal verification of programs that lack
  side-effects. While programming with Rust, requirements are usually not
  that demanding and robust. However, the same properties that make pure
  logic easier to formally verify also make for more *local reasoning*.

  [reasoning footprint]: https://blog.rust-lang.org/2017/03/02/lang-ergonomics.html#implicit-vs-explicit

  _By requring default field values to be `const` contexts, global reasoning
  can be avoided. Thus, the [reasoning footprint] for `Foo { .. }` is reduced._

+ By restricting ourselves to `const` contexts, you can be sure that default
  literals have a degree of *cheapness*.

  While `const` expressions form a turing complete language and therefore
  have no limits to their complexity other than being computable,
  these expressions are evaluated at *compile time*.
  Thus, *`const` expressions cannot have unbounded complexity at run-time*.
  At most, `const` expressions can create huge arrays and similar cases;

  Ensuring that `Foo { .. }` remains relatively cheap is therefore important
  because there is a general expectation that literal expressions have a small
  and predictable run-time cost and are trivially predictable.
  This is particularly important for Rust since this is a language that aims
  to give a high degree of control over space and time as well as predictable
  performance characteristics.

+ Keeping default values limited to `const` expressions ensures that if
  the following situation develops:

  ```rust
  // Crate A:
  pub struct Foo {
      bar: u8 = const_expr,
  }

  // Crate B:
  const fn baz() -> Foo {
      Foo { .. }
  }
  ```

  then crate A cannot suddenly, and unawares, cause a semver breakage
  for crate B by replacing `const_expr` with `non_const_expr` since
  the compiler would reject such a change (see lemmas 1-2).
  Thus, enforcing constness gives a helping hand in respecting semantic version.

  Note that if Rust would ever gain a mechanism to state that a
  function will not diverge, e.g.:

  ```rust
  nopanic fn foo() -> u8 { 42 } // The weaker variant; more easily attainable.
  total fn bar() -> u8 { 24 } // No divergence, period.
  ```

  then the same semver problem would manifest itself for those types of
  functions. However, Rust does not have any such enforcement mechanism
  right now and if it did, it is generally harder to ensure that a function
  is total than it is to ensure that it is deterministic; thus, while
  it is regrettable, this is an acceptable trade-off.

+ Finally, note that `const fn`s, when coupled with planned future extensions,
  can become quite expressive. For example, it is already planned that `loop`s,
  `match`es, `let` statements, and `panic!(..)`s will be allowed.
  Another feasible extension in the future is allocation.

  Therefore, constant expressions should be enough to satisfy most expressive
  needs.

## Instead of `Foo { ..Default::default() }`

As an alternative to the proposed design is either explicitly writing out
`..Default::default()` or extending the language such that `Foo { .. }` becomes
sugar for `Foo { ..Default::default() }`. While the latter idea does not satisfy
any of the [motivation] set out, the former does to a small extent.

In particular, `Foo { .. }` as sugar slightly improves ergonomics.
However, it has some notable problems:

+ Because it desugars to `Foo { ..Default::default() }`, it cannot be required
  that the expression is a constant one. This carries all the problems noted in
  the previous section on why default field values should be a `const` context.

+ It provides zero improvements to the ergonomics of *specifying* defaults,
  only for using them. Arguably, the most important aspect of this RFC is
  not the syntax `Foo { .. }` but rather the ability to provide default values
  for field.

+ By extension, the improvement to documentation clarity is lost.

+ No improvement is provided for `enum`s at all; in particular, you cannot
  `impl Default for Enum::Variant { .. }` while this RFC enables that.

+ The trait `Default` must now become a `#[lang_item]`. This is a sign of
  increasing the overall magic in the system; meanwhile, this proposal makes
  the default values provided usable by other custom derive macros.

Thus in conclusion, while desugaring `..` to `Default::default()` has lower cost,
it also provides significantly less value to the point of not being worth it.

## `..` is useful as a marker

One possible change to the current design is to permit filling in defaults
by simply writing `Foo {}`; in other words, `..` is simply dropped from the
expression.

Among the benefits are:

+ To enhance ergonomics of initialization further.

+ To introduce less syntax.

+ To be more in line with how other languages treat default values.

Among the drawbacks are:

+ The syntax `Foo { .. }` is no longer introduced to complement the identical
  pattern syntax. As aforementioned, destruction (and pattern matching)
  generally attempts to follow construction in Rust. Because of that,
  introducing `Foo { .. }` is essentially cost-free in terms of the complexity
  budget. It is arguably even cost-negative.

+ By writing `Foo { .. }`, you have there is explicit indication that default
  values are being used; this enhances local reasoning further.

# Prior art
[prior-art]: #prior-art

## Other languages

This selection of languages are not exhaustive; rather, a few notable or
canonical examples are used instead.

### Java

In Java it is possible to assign default values, computed by any expression,
to an instance variable; for example, you may write:

```java
class Main {
    public static void main(String[] args) {
        new Foo();
    }

    public static int make_int() {
        System.out.println("I am making an int!");
        return 42;
    }

    static class Foo {
        private int bar = Main.make_int();
    }
}
```

When executing this program, the JVM will print the following to `stdout`:

```
I am making an int!
```

Two things are worth noting here:

1. It is possible to cause arbitrary side effects in the expression that
   computes the default value of `bar`. This behaviour is unlike that which
   this RFC proposes.

2. It is possible to construct a `Foo` which uses the default value of `bar`
   even though `bar` has `private` visibility. This is because default values
   act as syntactic sugar for how the default constructor `Foo()` should act.
   There is no such thing as constructors in Rust. However, the behaviour
   that Java has is morally equivalent to this RFC since literals are
   constructor-like and because this RFC also permits the usage of defaults
   for private fields where the fields are not visible.

### Scala

Being a JVM language, Scala builds upon Java and retains the notion of default
field values. For example, you may write:

```scala
case class Person(name: String = make_string(), age: Int = 42)

def make_string(): String = {
    System.out.println("foo");
    "bar"
}

var p = new Person(age = 24);
System.out.println(p.name);
```

As expected, this prints `foo` and then `bar` to the terminal.

### Kotlin

Kotlin is similar to both Java and Scala; here too can you use defaults:

```kotlin
fun make_int(): Int {
    println("foo");
    return 42;
}

class Person(val age: Int = make_int());

fun main() {
    Person();
}
```

Similar to Java and Scala, Kotlin does also permit side-effects in the default
values because both languages have no means of preventing the effects.

### C#

Another language with defaults of the object-oriented variety is C#.
The is behaviour similar to Java:

```csharp
class Foo {
    int bar = 42;
}
```

### C++

Another language in the object-oriented family is C++. It also affords default
values like so:

```cpp
#include <iostream>

int make_int() {
    std::cout << "hello" << std::endl; // As in Java.
    return 42;
}

class Foo {
    private:
        int bar = make_int();
    public:
        int get_bar() {
          return this->bar;
        }
};

int main() {
    Foo x;
    std::cout << x.get_bar() << std::endl;
}
```

In C++ it is still the case that the defaults are usable due to constructors.
And while the language has `constexpr` to enforce the ability to evaluate
something at compile time, as can be seen in the snippet above, no such
requirement is placed on default field values.

### Swift

[Swift]: https://docs.swift.org/swift-book/LanguageGuide/Initialization.html

A language which is closer to Rust is [Swift], and it allows for default values:

```swift
struct Person {
    var age = 42
}
```

This is equivalent to writing:

```swift
struct Person {
    var age: Int
    init() {
        age = 42
    }
}
```

### Agda

Having defaults for record fields is not the sole preserve of OO languages.
The pure, total, and dependently typed functional programming language Agda
also affords default values. For example, you may write:

```agda
-- | Define the natural numbers inductively:
-- This corresponds to an `enum` in Rust.
data Nat : Set where
    zero : Nat
    suc  : Nat → Nat

-- | Define a record type `Foo` with a field named `bar` typed at `Nat`.
record Foo : Set where
    bar : Nat
    bar = zero -- An optionally provided default value.

myFoo : Foo
myFoo = record {} -- Construct a `Foo`.
```

In contrast to languages such as Java, Agda does not have have a notion of
constructors. Rather, `record {}` fills in the default value.

[strongly normalizing]: https://en.wikipedia.org/wiki/Normalization_property_(abstract_rewriting)

Furthermore, Agda is a pure and [strongly normalizing] language and as such,
`record {}` may not cause any side-effects or even divergence. However,
as Agda employs monadic IO in the vein of Haskell,
it is possible to store a `IO Nat` value in the record:

```agda
record Foo : Set where
    bar : IO Nat
    bar = do
        putStrLn "hello!"
        pure zero
```

Note that this is explicitly typed as `bar : IO Nat` and that `record {}` won't
actually run the action. To do that, you will need take the `bar` value and run
it in an `IO` context.

## Procedural macros

There are a number of crates which to varying degrees afford macros for
default field values and associated facilities.

### `#[derive(Derivative)]`

[`derivative`]: https://crates.io/crates/derivative

The crate [`derivative`] provides the `#[derivative(Default)]` attribute.
With it, you may write:

```rust
#[derive(Derivative)]
#[derivative(Default)]
struct RegexOptions {
    #[derivative(Default(value="10 * (1 << 20)"))]
    size_limit: usize,
    #[derivative(Default(value="2 * (1 << 20)"))]
    dfa_size_limit: usize,
    #[derivative(Default(value="true"))]
    unicode: bool,
}

#[derive(Derivative)]
#[derivative(Default)]
enum Foo {
    #[derivative(Default)]
    Bar,
    Baz,
}
```

Contrast this with the equivalent in the style of this RFC:

```rust
#[derive(Default)]
struct RegexOptions {
    size_limit: usize = 10 * (1 << 20),
    dfa_size_limit: usize = 2 * (1 << 20),
    unicode: bool = true,
}

#[derive(Default)]
enum Foo {
    #[default]
    Bar,
    Baz,
}
```

There a few aspects to note:

1. The signal to noise ratio is high as compared to the notation in this RFC.
  Substantial of syntactic overhead is accumulated to specify defaults.

2. Expressions need to be wrapped in strings, i.e. `value="2 * (1 << 20)"`.
   While this is flexible and allows most logic to be embedded,
   the mechanism works poorly with IDEs and other tooling.
   Syntax highlighting also goes out of the window because the highlighter
   has no idea that the string included in the quotes is Rust code.
   It could just as well be a poem due to Shakespeare.
   At best, a highlighter could use some heuristic.

3. The macro has no way to enforce that the code embedded in the strings are
   constant expressions. It might be possible to fix that but that might
   increase the logic of the macro considerably.

4. Because the macro merely customizes how deriving `Default` works,
   it cannot provide the syntax `Foo { .. }`, interact with privacy,
   and it cannot provide defaults for `enum` variants.

5. Like in this RFC, `derivative` allows you to derive `Default` for `enum`s.
   The syntax used in the macro is `#[derivative(Default)]` whereas the RFC
   provides the more ergonomic and direct notation `#[default]` in this RFC.

6. To its credit, the macro provides `#[derivative(Default(bound=""))]`
   with which you can remove unnecessary bounds as well as add needed ones.
   This addresses a deficiency in the current deriving system for built-in
   derive macros. However, the attribute solves an orthogonal problem.
   The ability to specify default values would mean that `derivative` can
   piggyback on the default value syntax due to this RFC. The mechanism for
   removing or adding bounds can remain the same. Similar mechanisms could
   also be added to the language itself.

### `#[derive(SmartDefault)]`

[`smart-default`]: https://crates.io/crates/smart-default

The [`smart-default`] provides `#[derive(SmartDefault)]` custom derive macro.
It functions similarly to `derivative` but is specialized for the `Default` trait.
With it, you can write:


```rust
#[derive(SmartDefault)]
struct RegexOptions {
    #[default = "10 * (1 << 20)"]
    size_limit: usize,
    #[default = "2 * (1 << 20)"]
    dfa_size_limit: usize,
    #[default = true]
    unicode: bool,
}

#[derive(SmartDefault)]
enum Foo {
    #[default]
    Bar,
    Baz,
}
```

+ The signal to noise ratio is still higher as compared to the notation in due
  to this RFC. The problems aforementioned from the `derivative` crate with
  respect to embedding Rust code in strings also persists.

+ Points 2-4 regarding `derivative` apply to `smart-default` as well.

+ The same syntax `#[default]` is used both by `smart-default` and by this RFC.
  While it may seem that this RFC was inspired by `smart-default`, this is not
  the case. Rather, this RFC's author came up with the notation independently.
  That suggests that the notation is intuitive since and a solid design choice.

+ There is no trait `SmartDefault` even though it is being derived.
  This works because `#[proc_macro_derive(SmartDefault)]` is in fact
  not tied to any trait. That `#[derive(Serialize)]` refers to the same
  trait as the name of the macro is from the perspective of the language's
  static semantics entirely coincidental.

  However, for users who aren't aware of this, it may seem strange that
  `SmartDefault` should derive for the `Default` trait.

### `#[derive(new)]`

[`derive-new`]: https://crates.io/crates/derive-new

The [`derive-new`] crate provides the `#[derive(new)]` custom derive macro.
Unlike the two previous procedural macro crates, `derive-new` does not
provide implementations of `Default`. Rather, the macro facilitates the
generation of `MyType::new` constructors.

For example, you may write:

```rust
#[derive(new)]
struct Foo {
    x: bool,
    #[new(value = "42")]
    y: i32,
    #[new(default)]
    z: Vec<String>,
}

Foo::new(true);

#[derive(new)]
enum Enum {
    FirstVariant,
    SecondVariant(bool, #[new(default)] u8),
    ThirdVariant { x: i32, #[new(value = "vec![1]")] y: Vec<u8> }
}

Enum::new_first_variant();
Enum::new_second_variant(true);
Enum::new_third_variant(42);
```

Notice how `#[new(value = "vec![1]")`, `#[new(value = "42")]`,
and `#[new(default)]` are used to provide values that are then omitted
from the respective constructor functions that are generated.

If you transcribe the above snippet as much as possible to the system proposed
in this RFC, you would get:

```rust
struct Foo {
    x: bool,
    y: i32 = 42,
    z: Vec<String> = <_>::default(),
    //               --------------
    //               note: assuming some `impl const Default { .. }` mechanism.
}

Foo { x: true };

enum Enum {
    FirstVariant,
    SecondVariant(bool, u8), // See future possibilities.
    ThirdVariant { x: i32, y: Vec<u8> = vec![1] }
}

Enum::FirstVariant;
Enum::SecondVariant(true, 0);
Enum::ThirdVariant { x: 42 };
```

Relative to `#[derive(new)]`, the main benefits are:

+ No wrapping code in strings, as noted in previous sections.
+ The defaults used can be mixed and matches; it works to request all defaults
  or just some of them.

The constructor functions `new_first_variant(..)` are not provided for you.
However, it should be possible to tweak `#[derive(new)]` to interact with
this RFC so that constructor functions are regained if so desired.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

1. What is the right interaction wrt. `#[non_exhaustive]`?

   In particular, if given the following definition:

   ```rust
   #[non_exhaustive]
   pub struct Config {
       pub height: u32,
       pub width: u32,
   }
   ```

   it is possible to construct a `Config` like so:

   ```rust
   let config = Config { width: 640, height: 480, .. };
   ```

   then adding a field to `Config` can only happen iff that field
   is provided a default value.

   This arrangement, while diminishing the usefulness of `#[non_exhaustive]`,
   makes the ruleset of the language simpler, more consistent, and also
   simplifies type checking as `#[non_exhaustive]` is entirely ignored
   when checking `Foo { fields, .. }` expressions.

   As an alternative, users who desire the semantics described above can
   omit `#[non_exhaustive]` from their type and instead add a private
   defaulted field that has a ZST.

   Resolving this question may be deferred to stabilization of either
   `#[non_exhaustive]` or field defaults, whichever comes last.

# Future possibilities
[future-possibilities]: #future-possibilities

## Tuple structs and tuple variants

Although it could, this proposal does not offer a way to specify default values
for tuple struct / variant fields. For example, you may not write:

```rust
#[derive(Default)]
struct Alpha(u8 = 42, bool = true);

#[derive(Default)]
enum Ingredient {
    Tomato(TasteQuality, Color = Color::Red),
    Lettuce,
}
```

While well-defined semantics could be given for these positional fields,
there are some tricky design choices; in particular:

+ It's unclear whether the following should be permitted:

  ```rust
  #[derive(Default)]
  struct Beta(&'static str = "hello", bool);
  ```

  In particular, the fields with defaults are not at the end of the struct.
  A restriction could imposed to enforce that. However, it would also be
  useful to admit the above definition of `Beta` so that `#[derive(Default)]`
  can make use of `"hello"`.

+ The syntax `Alpha(..)` as an expression already has a meaning.
  Namely, it is sugar for `Alpha(RangeFull)`. Thus unfortunately,
  this syntax cannot be used to mean `Alpha(42, true)`.
  In some future edition, the syntax `Alpha(...)` (three dots)
  could be used for filling in defaults. This would ostensibly entail
  adding the pattern syntax `Alpha(...)` as well.

  Until a syntax such as `Alpha(...)` is made available,
  tuple structs could be constructed with `Alpha { 0: 43 }`.
  However, that is not particularly ergonomic and that diminishes the value
  of having field defaults for tuple structs.

For these reasons, default values for positional fields are not included in
this RFC and are instead left as a possible future extension.

## Integration with structural records

[RFC 2584]: https://github.com/rust-lang/rfcs/pull/2584

In [RFC 2584] structural records are proposed.
These records are structural like tuples but have named fields.
As an example, you can write:

```rust
let color = { red: 255u8, green: 100u8, blue: 70u8 };
```

which then has the type:

```rust
{ red: u8, green: u8, blue: u8 }
```

These can then be used to emulate named arguments,
which there is a longstanding desire for in parts of the Rust community.
For example:

```rust
fn open_window(config: { height: u32, width: u32 }) {
    // logic...
}

open_window({ height: 720, width: 1280 });
```

Since this proposal introduces field defaults, the natural combination with
structural records would be to permit them to have defaults. For example:

```rust
fn open_window(config: { height: u32 = 1080, width: u32 = 1920 }) {
    // logic...
}
```

A coercion could then allow you to write:

```rust
open_window({ .. });
```

This could be interpreted as `open_window({ RangeFull })`, see the previous
section for a discussion... alternatively `open_window(_)` could be permitted
instead for general value inference where `_` is a placeholder expression
similar to `_` as a type expression placeholder
(i.e. a fresh and unconstrained unification variable).

If you wanted to override a default, you would write:

```rust
open_window({ height: 720, });
```

Note that the syntax used to give fields in structural records defaults belongs
to the type grammar; in other words, the following would be legal:

```rust
type RGB = { red: u8 = 0, green: u8 = 0, blue: u8 = 0 };

let color: RGB = { red: 255, };
```

As structural records are not yet in the language,
figuring out designs for how to extend this RFC to them is left
as possible work for the future.
