- Start Date: 10/14/2014
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Add anonymous enum types called joins, using `A | B` for types, no explicit
instantiation syntax, and subtyping relationships.

# Motivation

There are two primary motivations for joins:
  1. They allow the creation of small, "throw-away" enums where you might
     otherwise want `Result<A, B>` or even `Either3<A, B, C>` without creating
     and using verbose types.
  2. They allow creating enum-like types with *no explicit instantiation*
     syntax, which can be useful for creating noise-free DSLs or other systems
     where the explicit instantiation of enums is just line-noise.

## 1.

Small anonymous enums are very useful for representing error types in small
libraries or in applications, where restricting which of many errors a function
can return is very useful.

Outside of error types, small joins are also useful for representing which types
can be used as input to functions that can accept a variety of types when a trait
would be overly general and create unnecessary code-bloat and an enum would be too
noisy.

Right now, most of these functions are forced to use a trait which must be
separately-defined and has to be exported (allowing additional implementors
even when that may be undesirable), since we no longer support private traits
in public bounds.

## 2.

Explicit instantiation syntax is often a good feature, since it makes it clear
what type is being instantiated and it is easy to find the definition of the
variant and enum.

However, there are many cases where it is useful to have implicit
instantiation, particularly for function arguments, errors outside of `try!`,
noise-free DSLs and other cases where enums create unnecessarily verbose
instantiation syntax.

Here is a small program motivating 2:

In this program, we will be designing a type-safe representation of
JSON that is optimized for noise-free creation of JSON in Rust through a
DSL-like syntax. It assumes a hypothetical `map!` macro for quickly creating
instances of `HashMap`.

Here is our program using join types:

```rust
// The defition of Json is very short and easy to read.
type Json = Vec<Json> | f64 | JsonString | HashMap<JsonString, Json> | Null;
type JsonString = String | &'static str;

struct Null;

let json = vec![46.0, "hello", map! { "key": Null }];
```

Compare this to the equivalent in JavaScript, where JSON is native (pretty close!):

```js
var json = [46, "hello", { key: null }];
```

Here is the same program using enums:

```rust
use std::str::SendStr;
use std::borrow::Cow::{Borrowed, Owned};

use self::Json::*;

enum Json {
    Array(Vec<Json>),
    Number(f64),
    String(SendStr),
    Object(HashMap<SendStr, Json>),
    Null
}

let json = Array(vec![
    Number(46.0),
    String(Borrowed("hello")),
    Object(map! { String(Borrowed("key")): Null }
]);
```

The additional structure added by using explicit constructors is often
useful, but in many cases it is almost *entirely* noise and makes this use case
much more verbose.

# Detailed design

There are many details to cover when introducing this feature, so this section
is split into many sub-sections that will discuss important parts of the
design.

## General Use

The general use of this feature is best described through some short examples.

`A | B` names a type and can be used anywhere a type is expected.

```rust
type AorB = A | B;
```

There is no explicit instantiation syntax for values of join types, instead
if a binding or value is a join type then any value included in the join type
can be used to instantiate that join.

```rust
struct A { x: uint }

let val: A | B = A { x: 8u };
```

Join types are order-independent, even cross-crate, so `A | B` is the same type
as `B | A`.

```rust
let val: A | B = A { x: 8u };
let val: B | A = val;
```

You can convert between join types using `as`. `as` can be used to convert
a join to another join whose types are a superset of the types in the original
join. `as` can also be used to change the local ordering of a join in cases
where order becomes important (see Proposal B under "Interaction with Generics").

Joins are *not* converted implicitly.

```rust
let val: A | B = A { x: 17u };
let val = val as A | B | C;
let val = val as B | A | C;

// Error: can't shrink a join using as.
let val = val as A | B;
```

Join types are always flattened as much as possible, breaking off at non-join
types. This means `(A | B) | C` is the same type as `A | B | C`. The only
exceptions is when `A` and `B` are generics and `A | B` is used as a join - if
`A` or `B` are instantiated as joins, then those joins are only flattened from
the perspective of the caller.

### Traits

Joins have the same rules as tuples when it comes to trait implementations -
you can implement a trait for a join in a crate if the trait originates in
the crate or if any types in the join originate in the crate.

### Matching

To disambiguate a join, you use `match` with an additional `as $type` clause:

```rust
let val: A | B | C = ...;
match val {
    $pat as A { .. },
    $pat as B { .. },
    $pat as C { .. }
}
```

As a special case, `_` can be used without type bounds to capture many types,
since it does not capture a value.

You can also use `if let` and `while let` with the extended `as $type` syntax
like so:

```rust
if let $pat as $type = val { .. }
```

## Grammar

`|` has a lower precedence than the pointer syntax, meaning `&A | B` is
`(&A) | B)` and `&(A | B)` is needed to get a reference to a join.

This is to avoid weirdness where `&A | B` is a reference to the join but
`B | &A` is a join with a reference.

## Interaction with Generics (Dealing with Duplicates)

There are several possible designs that reflect how joins interact with
generics, motivated by issues arising from the possibility of duplicate
or unifying types in joins.

To compare all the proposal on fair ground, we will be discussing the viability
of a join such as:

```rust
struct Nothing;
type Optional<T> = T | Nothing;
```

### Proposal A: Forbid possibly-unifying types in Joins

Proposal A outright bans the creation of Joins where one type may unify
with another, for instance `<A, B> A | B`. In the future, if we grow type
equality or unification bounds, such a join may be allowed as `<A, B> A | B where A != B`.

#### Pros:

Proposal A is simple and can use the existing compiler infrastructure for
checking if two generic or concrete types unify that is used in checking
multidispatch trait implementations.

#### Cons:

Proposal A significantly hampers the utility of joins in generic contexts
until (read: if) we get type equality bounds, negative bounds, and/or negative
trait impls.

For instance, under Proposal A even the simple `Optional` join would require
one of:

Type Equality Bounds:

```rust
struct Nothing;
type Optional<T != Nothing> = T | Nothing;
```

Negative Trait Impls:

```rust
struct Nothing;
type Optional<T: Something> = T | Nothing;

trait Something {}
impl<T> Something for T {}
!impl Something for Nothing {}
```

or Negative Bounds:

```rust
struct Nothing;
type Optional<T: !NotAThing> = T | Nothing;

trait NotAThing {}
impl NotAThing for Nothing {}
```

Even if we do get one or all of these features, it remains verbose to use joins
in this case.

### Proposal B: Use ordering annotations to disambiguate unifying types.

Proposal B allows unifying or duplicate types, but requires ordering
annotations to match against joins which may contain duplicate types.

Under Proposal B, `Optional` could be defined as it is in the example code
above.

Matching against `Optional<T>` where `T` cannot be `Nothing` requires no
additional annotations.

If `T` can be `Nothing`, then you can no longer match using the `as T` syntax,
you must instead use an ordering annotation, which takes the form `as T.0` to
say you'd like to match against the first (0-indexed) variant whose type is T.

These ordering annotations are *always* relative to the local definition of the
join, i.e. even though `A | B` is the same as `B | A`, the ordering information
in the declaration is preserved in a local context for these annotations.

### Pros:

Proposal B allows using duplicate types in joins with minimal additional
complexity.

### Cons:

Proposal B introduces a new weird grammar to matching and requires
additional analysis to decide when ordering annotations are necessary, though
it is likely that existing infrastructure can be reused for this check.

The compiler has to track the ordering of various definitions of the "same"
join type, and recognize different ordering annotations associated with them.

### Unanswered Questions with Proposal B:

How do ordering annotations work with "flattening", i.e. how do we match `A`
in `(A | B) | C` where `A`, `B`, and `C` can unify.

### Alternative Syntax for Proposal B:

We could use `as 0` or `as .0` or any other syntax for ordering.

## Interaction with Type Inference

Types are not ever inferred to be join types - if you want a join type, you
have to explicitly request it somewhere.

## Memory Representation

The in-memory representation of a join is the same as an enum with the
equivalent members and number of variants. The ordering of the variants of a
join (i.e. which discriminant value matches which variant) is unspecified.

Of course, the compile must still be able to decide which variant a join is, so
a strategy for compiling joins and deciding their order that is viable even in
cross-crate cases is discussed in the "Compilation Strategy" section.

## Compilation Strategy

Even though the order of variants is unspecified, the compiler must know the
order of variants. Additionally, the order of variants of joins of the same
types must be *consistent* between crates, even when each crate is compiled
completely separately.

Given this constraint, the best solution the author can come up with is to sort
the variants of the join by their `TypeId`, or some other cross-crate unique
piece of data, and use that as the ordering of variants.

# Drawbacks

It's a highly complex feature that introduces new forms of implicit behavior
and a relatively complicated compilation strategy, adding complexity to the
compiler and language.

It doesn't enable anything that was actually impossible previously, just makes
certain cases easier or much shorter.

# Alternatives

Don't do this.

Create a different syntax for implicit instantiation of regular enums.

# Unresolved questions

Should joins come with subtyping and conversions, i.e. should there be an
implicit conversion of a join of types `Ts` to a join of types `Ts'` if `Ts` is
a subtype of `Ts'`?

