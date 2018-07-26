- Feature Name: `builtin_derive_flexibility`
- Start Date: 2018-07-25
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

When deriving a bultin standard library trait such as "`Debug`" and a field is
a projection of the form `<T as Baz>::Quux` where `T` is a type parameter,
a bound `<T as Baz>::Quux: TheTrait`  is added and a bound `T: TheTrait` is not
(unless if required by other fields).

# Motivation
[motivation]: #motivation

Simply put, the derive macros, provided by the compiler for the standard library
traits, are not as flexible as they could be. By making them more flexible,
we can derive in more cases and thus avoid boilerplate in more cases.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Consider for a moment the following definitions:

```rust
trait Baz {
    type Quux;
}

struct Bar;

impl Baz for Bar {
    type Quux = usize;
}
```

Then, let's add the following type:

```rust
#[derive(Debug)]
struct Foo {
    field: <Bar as Baz>::Quux,
}
```

The compiler is happy to accept this and expands `#[derive(Debug)]` to:

```rust
#[automatically_derived]
#[allow(unused_qualifications)]
impl ::std::fmt::Debug for Foo {
    fn fmt(&self, f: &mut ::std::fmt::Formatter) -> ::std::fmt::Result {
        match *self {
            Foo {
                field: ref __self_0_0,
            } => {
                let mut debug_trait_builder = f.debug_struct("Foo");
                let _ = debug_trait_builder.field("field", &&(*__self_0_0));
                debug_trait_builder.finish()
            }
        }
    }
}
```

So far so good, this type checks fine and we're happy.
Now, we decide to parameterize the type `Bar` which we're projecting on and
instead write:

```rust
#[derive(Debug)]
struct Foo<T: Baz> {
    field: <T as Baz>::Quux,
}
```

This expands to:

```rust
#[automatically_derived]
#[allow(unused_qualifications)]
impl<T: ::std::fmt::Debug + Baz> ::std::fmt::Debug for Foo<T> {
    fn fmt(&self, f: &mut ::std::fmt::Formatter) -> ::std::fmt::Result {
        match *self {
            Wibble {
                field: ref __self_0_0,
            } => {
                let mut debug_trait_builder = f.debug_struct("Foo");
                let _ = debug_trait_builder.field("field", &&(*__self_0_0));
                debug_trait_builder.finish()
            }
        }
    }
}
```

But to our dismay, this does not compile anymore as the required bound
`<T as Baz>::Quux: Debug` needed for debugging `field` does not hold.
Indeed, the compiler says as much with the following error:

```rust
error[E0277]: the trait bound `<T as Baz>::Quux: std::fmt::Debug` is not satisfied
```

But this does not have to be the case. If the generated implementation would
have been the following instead, it would have all worked fine:

```rust
#[automatically_derived]
#[allow(unused_qualifications)]
impl<T: Baz> ::std::fmt::Debug for Foo<T>
where
    <T as Baz>::Quux: ::std::fmt::Debug
{
    // same as above..
}
```

This is exactly what this RFC proposes that we do if a standard library trait
is being derived by the builtin derive macros. This extends to any number of
fields as well as fields in variants of `enum`s. The algorithmic changes
proposed in this RFC should not lead to any sort of breakage.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Derivable standard library traits

The following traits have derive macros provided by the compiler:

+ `Debug`
+ `Default`
+ `Hash`
+ `Clone`
+ `Copy`
+ `PartialEq`
+ `Eq`
+ `PartialOrd`
+ `Ord`

## Changes to derive macros

For all of the built-in derive macros, when processing a type definition,
for each field of that type definition, or one of its variants, which has a type
which is syntactically specified (as opposed to originating from a type macro)
in the type definition as: `<$T as $Bound>::$Projection`,
where `$T` is an identifier which is declared as a type parameter in the head
of the definition, where `$Bound` is a valid trait in that syntactic position,
where `$Projection` is an identifier, then a bound
`<$T as $Bound>::$Projection: $Trait` will be added to the generated implementation
where `$Trait` is the trait being derived.

For the purposes of collecting such bounds, the derive macros will treat the
element types of arrays and of tuples, however nested, but not if applied to a
type macro or a type constructor which is not an array or a tuple, the same
as if they were the type of a field.

Furthermore, if all of the occurrences of a type parameter in the type definition
have the form `<$T as $Bound>::$Projection` as per above, then a bound
`$T: $Trait` will *not* be added to the implementation.

The algorithm to compute all of the above is the following:

```rust
type AddBoundSet = HashMap<Ident, bool>;

let mut impl_generics = ...;

let mut map_add_bound: AddBoundSet = derive_input
    .type_parameters
    .map(|tp| (tp.ident, false))
    .collect();

fn walk_type(
    in_path: bool, ty: Type,
    map: &mut AddBoundSet, generics: &mut Generics
) {
    match ty {
        Never(_) => {},
        Macro(_) => map.values_mut().for_each(|add| *add = true),
        Group(group) => walk_type(in_path, group.elem, map, generics),
        Paren(group) => walk_type(in_path, paren.elem, map, generics),
        Slice(slice) => walk_type(in_path, slice.elem, map, generics),
        Array(array) => walk_type(in_path, array.elem, map, generics),
        Tuple(tuple) => tuple.elems.into_iter().for_each(|elem|
            walk_type(in_path, elem, map, generics)
        ),
        ImplTrait(obj) => obj.bounds.into_iter().for_each(|bound| match bound {
            Lifetime(_) => {},
            Trait(bound) => {
                // TODO
            },
        }),
        TypeTraitObject(obj) => obj.bounds.into_iter().for_each(|bound| match bound {
            Lifetime(_) => {},
            Trait(bound) => {
                // TODO
            },
        }),
        Path(path) => {

        },
        

        //Ptr(ptr) => recurse(ptr.elem),
        //Reference(ptr) => recurse(ptr.elem),
        //TypeBareFn(fun) => {},
    }
}


for ty in derive_input.fields.map(get_types) // including those of variants.
{
}

no_add_set.
```

# Drawbacks
[drawbacks]: #drawbacks



# Rationale and alternatives
[alternatives]: #alternatives



# Prior art
[prior-art]: #prior-art


# Unresolved questions
[unresolved]: #unresolved-questions

