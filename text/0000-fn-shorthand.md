- Feature Name: `fn_shorthand`
- Start Date: 2018-11-21
- RFC PR: _
- Rust Issue: _

# Summary
[summary]: #summary

Allow functions to be defined with `fn foo(...) -> R = expr;`.
In other words, the following becomes legal:

```rust
fn double(x: u8) -> u8 = x * 2;

/// Normalize a `meta: Meta` into the forms accepted in `#[proptest(<meta>)]`.
fn normalize_meta(meta: Meta) -> Option<NormMeta> = Some(match meta {
    Meta::Word(_) => NormMeta::Plain,
    Meta::NameValue(nv) => NormMeta::Lit(nv.lit),
    Meta::List(ml) => if let Some(nm) = util::match_singleton(ml.nested) {
        match nm {
            NestedMeta::Literal(lit) => NormMeta::Lit(lit),
            NestedMeta::Meta(Meta::Word(word)) => NormMeta::Word(word),
            _ => return None
        }
    } else {
        return None
    },
});
```

# Motivation
[motivation]: #motivation

## Reducing rightward drift

As we have seen in the [summary], by permitting the user to write
`= match e { ... }` we were able to reduce the rightward drift of the code.
While this is just one level of indent, it does not take many levels
(particularly if you restrict yourself to 80 characters, but also with 100) to
push the code outside of the text area of an editor. By removing that first
initial step in some cases we can reduce the drift in many cases and thus
improve readability.

### Real world use cases

The benefit is in particular noticeable on functions that directly `match`
on some arguments such as:

```rust
use syn::Expr;

pub fn eval_expr(expr: &Expr) -> Option<u128> = match expr {
    Expr::Lit(expr) => eval_lit(expr),
    Expr::Binary(expr) => eval_binary(expr),
    Expr::Unary(expr) => eval_unary(expr),
    Expr::Paren(expr) => eval_expr(&expr.expr),
    Expr::Group(expr) => eval_expr(&expr.expr),
    _ => None,
};
```

Functions that are so simple that they can fit in one line which yet make
sense as units are also good candidates:

```rust
fn compile_error(msg: &str) -> TokenStream = quote! { compile_error!(#msg); };
```

Additionally, using `try { .. }` as the immediate expression of a function
becomes quite ergonomic, for example:

```rust
#[post("/github-webhook", data = "<event>")]
pub fn github_webhook(event: Event) -> DashResult<()> = try {
    let conn = &*DB_POOL.get()?;

    match event.payload {
        Payload::Issues(issue_event) =>
            handle_issue(conn, issue_event.issue, &issue_event.repository.full_name)?,
        Payload::PullRequest(pr_event) =>
            handle_pr(conn, pr_event.pull_request, &pr_event.repository.full_name)?,
        Payload::IssueComment(comment_event) => if comment_event.action != "deleted" {
            handle_issue(
                conn,
                comment_event.issue,
                &comment_event.repository.full_name,
            )?;
            handle_comment(
                conn,
                comment_event.comment,
                &comment_event.repository.full_name,
            )?;
        },
        Payload::Unsupported => (),
    }

    // Notice the lack of `Ok(())` because of `try { .. }`.
};
```

### In the standard library

One instance of this is in `src/core/iter/mod.rs`. For example:

```rust
impl<I: Iterator> Iterator for Peekable<I> {
    type Item = I::Item;

    fn next(&mut self) -> Option<I::Item> = match self.peeked.take() {
        Some(v) => v,
        None => self.iter.next(),
    };

    fn count(mut self) -> usize = match self.peeked.take() {
        Some(None) => 0,
        Some(Some(_)) => 1 + self.iter.count(),
        None => self.iter.count(),
    };

    ...
}
```

## Encouraging small function definitions

[SRP]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[SoC]: https://en.wikipedia.org/wiki/Separation_of_concerns

As a result of reduced rightward drift, and removing the need to always
have 2 extra lines for the braces, as seen in the definition of `double`,
we can make it more ergonomic to write small function definitions. This in
turn can in many cases improve readability as it becomes easier to define self
contained units that respect the *[single responsibility principle (SRP)][SRP]*,
*[separation of concerns (SoC)][SoC]* or *modularity* in general.

## A more syntactically unified feeling

Finally, a rather minor motivation, which still bears mentioning, is
that by permitting `fn foo(...) -> R = expr;` as a syntax, we bring `const`
and `static` items closer to `fn` items syntactically which gives a smoother
experience overall. In other words, we can now write:

```rust
static WIBBLE: Wobble = make_wobble();

const FOO: Bar = make_bar();

fn baz() -> Quux = make_quux();
```

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Simply put, this RFC allows you to, in addition to the existing syntax,
define the body of a function, whether the function is free or associated,
with the following shorthand syntax:

```rust
fn foo(arguments...) -> ReturnType = expression;

// If the return type is () you can omit `-> ReturnType` as usual.
```

For example, we may write:

```rust
pub fn self_ty() -> syn::Type = parse_quote!(Self);

impl Params {
    pub fn empty() -> Self = Params(Vec::new());

    pub fn len(&self) -> usize = self.0.len();
}

impl UseTracker {
    pub fn no_track(&mut self) = self.track = false;

    pub fn consume(self) -> syn::Generics = self.generics;

    fn use_type(&mut self, ty: syn::Type) = self.where_types.insert(ty);
}
```

This is particularly useful when your function immediately matches on some
argument such as with:

```rust
use syn::Fields;

pub fn fields_to_vec(fields: syn::Fields) -> Vec<syn::Field> = match fields {
    Fields::Named(fields) => fields.named.into_iter().collect(),
    Fields::Unnamed(fields) => fields.unnamed.into_iter().collect(),
    Fields::Unit => vec![]
};
```

The RFC does not make any distinction between `const fn`s, visibilities,
`unsafe fn`, `async fn`, `extern` modifiers and so on. That is, wherever
a function definition can currently have a body, then the shorthand syntax
as proposed here may be used.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The grammar of `fn` item definitions is amended such that you may also
use the syntax `= $expr;` for the body of the function instead of
`{ $stmt* $tail_expr }`. Note that `= $expr;` comes after a `where` clause
if any such clause is present. Examples include:

```rust
fn foo(bar: u8) -> u8 = bar * 2;

fn baz(quux: usize) -> &'static str = match quux % 3 {
    0 => "divisible by 3",
    1 => "not divisible, rest 1",
    2 => "not divisible, rest 2",
};
```

The static and dynamic semantics of the `= $expr` form is equivalent
in all respects to that of `{ $expr }`. Conversely the existing form
`{ $stmt* $tail_expr }` is equivalent to `= { $stmt* $tail_expr }`.

# Drawbacks
[drawbacks]: #drawbacks

1. This is another way to do things and while it is a nicety,
   it does not any new powers or abilities or add any sort of syntactic sugar.
   Instead, the focus is on ergonomics and readability.

2. Many C-family languages do not use the `= $expr` syntax and for uses who
   come from those languages it may be unfamiliar syntax.
   However, the syntax is fairly intuitive.
   Furthermore, there is actually some precedent for the syntax with
   some C-style languages such as C# and Kotlin.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## `=` or `=>`

We have chosen the concrete syntax `=` instead of `=>`.
While there's precedent for using `=>` from match arms in Rust itself and
from C#, we believe that the overall consistency is better if a concrete
syntax is used that related `const` and `static` items as well as `let`
bindings more closely together. Since these use `=`, we use that instead
of `=>`. Furthermore, the precedent in other languages for using `=`
is also stronger than for the alternative.

## Expression oriented languages

In our survey in the [prior-art], we note that it is quite common for expression
oriented languages to have a syntactic form for function definitions akin to
the one proposed in this RFC. As Rust is also a expression oriented language,
it does make sense for Rust to have such a syntactic form as reasons for and
benefits of having the `=` form in those languages also apply to Rust.

## Not doing this

As always, we can elect not to provide such a shorthand for defining functions.
If we decide this, it would not be the end of the world. The changes proposed
in this RFC are strictly for improved ergonomics in many cases readability
and maintainability as well. The RFC does however not provide any increase
in power such that certain previously inexpressible things now become expressible.
The benefits to ergonomics are however pervasive as this impacts a central
piece of the language rather than some syntax in the periphery.

# Prior art
[prior-art]: #prior-art

There are a number of languages which support the equational `=` style of
defining functions.

For the sake of simplicity, we'll use the identity function as our running
example to facilitate comparisons.

## Functional Languages

Chiefly among languages that supports the syntax `=` are functional programming
languages. These languages use this syntax because they are expression oriented
languages, which Rust also is.

### The Haskell family

In the Haskell family of languages, i.e. those that resemble Haskell syntactically,
including at least Haskell, Elm, PureScript, Idris, and Agda, you can write:

```haskell
identity x = x
```

Optionally, you can also write a type annotation; In Haskell this is:

```haskell
identity :: a -> a
identity x = x
```

Meanwhile, in Idris and Agda you must write the type annotation due to the
lack of global type inference.

### The ML family

Being an MLs, Ocaml, F#, and F* allow you to write:

```ocaml
let identity x = x
```

However, Standard ML uses the following syntax instead:

```sml
fun identity x = x
```

## Hybrid languages

There are languages which embrace both the OO/Imperative style as well as the
functional style. Let's go through some of those. Rust being a hybrid language
itself, these language are of particular interest.

### Scala

To define the identity function in Scala, we write:

```scala
def identity[T](x: T): T = x
```

Interestingly, while Scala has the `=` syntax and recommends it, the language
also has what it calls the "procedure syntax":

```scala
def printBar(bar: Baz) {
    println(bar)
}
```

This way of writing things is discouraged.
Instead, users are encouraged to write:

```scala
def printBar(bar: Bar): Unit = {
    println(bar)
}
```

When a function needs to introduce some bindings or has some mutable state,
a use can use this syntax and write:

```scala
def sum(ls: List[String]): Int = {
  val ints = ls map (_.toInt)
  ints.foldLeft(0)(_ + _)
}
```

### Kotlin

A language which is even more interesting than Scala is Kotlin.

The standard way to define the identity function is:

```kotlin
fun <T> identity(x: T): T = x
```

The syntax is reminiscent of a mix between Java and Scala.
Notice that the `=` syntax is supported, but only when the body of the
function is a single expression.

Like Rust, the language also supports the braced variants when so desired:

```kotlin
fun double(x: Int): Int {
    return x * 2
}
```

## Imperative

On the more imperative side of languages there's also support for the idea
behind `=`.

### C#

C# provides the usual braced C-style function definition, e.g.:

```csharp
public void DisplayName() {
    Console.WriteLine(ToString());
}
```

However, we can also use the "expression-bodied members" syntax:

```csharp
public void DisplayName() => Console.WriteLine(ToString());
```

The concrete syntax used here is `=>` which is not exactly the same as `=`
but captures the same basic idea of a light weight syntax for expressions
on the right hand side.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

None.

We leave all style decisions up for future considerations to allow for
decisions based on experimentation.

# Future possibilities
[future-possibilities]: #future-possibilities

## `LambdaCase`

TODO
