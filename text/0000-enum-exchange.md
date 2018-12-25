- Feature Name: enum_exchange
- Start Date: 2018-12-20
- RFC PR:
- Rust Issue:

# Summary

Add macros and traits in std library, for minimal support of ad-hoc enums.

The new constructs are:

* Four traits `FromVariant`, `IntoEnum`, `ExchangeFrom`, `ExchangeInto`.

* A new proc-macro derive `Exchange` applicable for `enum`s, generating type
convertion `impl`s defined in these four traits.

* A declarative macro `Enum!` accessing to predefined `Exchange`able enum.

# Motivation

An enum can be utilized to express a set of finite, known elements. Enums
composed of variants that do not refer to domain knowledge are considered as
ad-hoc enums. They serve as a mechanism of code organisations.

Full-fledged ad-hoc enums provide mechanisms not only for gathering values of
variants into enums, but also for gathering variants of an ad-hoc enum, into
another one.

Such a mechanism is not available in Rust, requiring Rustaceans to implement it
themselves when needed.

While it is easy to write maros for "variants => enum" gathering, a general
"enum => enum" convertion is non-trival to implement.

These facts have caused some unfavorable results:

- Programmers are developing such non-trival equivalents in specific domains. A
  notable example is ["error-chain"](https://crates.io/crates/error-chain). It
  had reinvented the wheel for some kind of ad-hoc enum, aka `ErrorKind`.

- Tempting to use `trait` object instead, in cases which ad-hoc `enum` is most
  suitable for.

This RFC addresses these issues by inroducing a minimum support of ad-hoc enums,
aka exchangeable enums.

# Overview

An enum with `#[derive(Exchange)]` is considered as an exchangeable enum.

```rust
#[derive(Exchange)]
enum Info {
    Code(i32),
    Text(&'static str),
}
```

An exchangeable enum can be constructed from one of its variants:

```rust
let info: Info = 42.into_enum();
let info = Info::from_variant(42);
```

An exchangeable enum can be exchanged from/into another exchangeable one, as
long as one has all the variant types appearing in the other one's definition.

```rust
#[derive(Exchange)]
enum Data {
    Num(i32),
    Text(&'static str),
    Flag(bool),
}

let info: Info = 42.into_enum();
let data: Data = info.exchange_into();

let info = Info::from_variant(42);
let data = Data::exchange_from(info);
```

## Enum methods

By now, we call `from_variant()`, `into_enum()`, `exchange_from()`,
`exchange_into()`as enum exchange methods.

## Syntax limits of exchangeable enum

All variants must be in the form of "newtype".

```rust
#[derive(Exchange)]
enum Info {
    Text(String),  // ok, it is newtype
    Code(i32,u32), // compile error
}
```

should cause an error:

```text
1926 | Code(i32,u32),
     | ^^^^^^^^^^^^^ all variants of an exchangeable enum must be newtype
```

## Predefined exchangeable enums

The `Enum!( T0, T1, .. )` macro defines an predefined exchangeable enum composed
of variant types T0, T1, .. etc.

```rust
let info = <Enum!(i32,&'static str)>::from_variant(42);
```

is essentially equvalent to the following:

```rust
let info = __Enum2::from_variant(42);
```

while `__Enum2` is predefined but **may not exposed to programmers**:

```rust
#[derive(Exchange)]
enum __Enum2 {
    _0(i32),
    _1(&'static str),
}
```

Two `Enum!()`s with identical variant type list are identical types.

`<Enum!( T0, T1, .. )>::_0` for the first variant in pattern matching, and so
forth.

```rust
match info {
    <Enum!(i32,&'static str)>::_0(_i) => (),
    <Enum!(i32,&'static str)>::_1(_s) => (),
}
```

Predefined enums are also `Exchange`able enums. They are considered as unnamed
exchangeable enums, while user-defined ones are called named exchangeable enums.

An unnamed exchangeable enum is suitable in such usescases that an ad-hoc enum
does not worth a name, while a named exchangeable enum serves for those do worth
naming. 

From now on, we will prefer using predefined enums in examples, since they are
superior as notations, comparing to user-defined enums.

## Convertion rules

The following 2 rules are considered as the minimum support of ad-hoc enum:

1. variant <=> exchangeable enum

  `from_variant()`/`into_enum()` serve for it.

2. exchangeable enum => exchangeable enum composed of equal or more variants

  `exchange_from()`/`exchange_into()` serve for it.

The following rules are considered perculiar to exchangeable enums, which
distinguish them from "union types" in Typed Racket.

1. An exchangeable enum composed of duplicated variant types is a valid enum,
but it is nonsense because acual uses of its enum exchange methods will cause
compile errors.

```text
9 | let a = <Enum!(i32,i32)>::_0( 3722 );
  |         ^^^^^^^^^^^^^^^^ variants of an exchangeable enum must be unique.
```

2. No automatic flattening

  For example, `Enum!(A,Enum!(B,C))` can not be converted to `Enum!(A,B,C)` via
  enum exchange methods. Further more, making these two equal types will need
  changes in type systems, which is not possible for a proc-macro derive.

# Detailed design

The definition of enum exchange traits are as following:

```rust
pub trait FromVariant<Variant,Index> {
    fn from_variant(v: Variant) -> Self;
}

pub trait IntoEnum<Enum,Index> {
    fn into_enum(self) -> Enum;
}

pub trait ExchangeFrom<Src,Indices> {
    fn exchange_from(src: Src) -> Self;
}

pub trait ExchangeInto<Dest,Indices> {
    fn exchange_into(self) -> Dest;
}
```

Notice that all traits have a phantom index type `Index` or `Indices` in their
generics to hold positional information to help compiler accomplishing type
inferences.

## Distinguish from `std::convert`

Since standard `From`/`Into` does not have such index type, it is not feasible
to implement enum exchange methods in `From`/`Into`. Trying to implement in
`From` will cause compile error because we need to impl multiple `From<Variant>`
but the generic `Variant` type in different `impl`s could be of the same actual
type, resulting in overlapping `impl`s.

## Distinguish between `Index` and `Indices`

Consider the convertion from `Enum!(A,B)` to `Enum!(A,B,Enum!(A,B))`.

Since these two enum types are not equal due to lacking of flattening , there
are two possible ways for this convertion:

1. making the former as the third variant of the latter.

2. matching the former to get an `A` or `B`, then making it as the first or
  second variant of the latter.

This is the root cause we distinguish between `FromVariant` and `ExchangeFrom`.

## Interaction with other feature

The library implementation of enum exchange, aka
[EnumX](https://crates.io/crates/enumx), is a proof that the proposed construct
will left Rust syntax and its type systems as untouched.

Enum variant types, if available, will relax the "newtype limit" in user-defined
exchangeable enum. 

# Drawbacks

- Abusing ad-hoc enums in cases that they are not suitable for.

- Distinguish between `FromVariant` and `ExchangeFrom` will cause extra
annotations, which may be unnecessary in some usecases.

# Rationale and alternatives

Two alternatives for supporting ad-hoc enums:

1. Making it a language-support type, which are big changes for Rust community to
  agree on in near term.

2. Implementing it as a third-party library, which suffers from leaking
  implementation details and misleading compile errors.

## Misuse: always defining types and their convertions explicitly

People misusing this believe it is in such a case that:

1. All types, including ad-hoc enums, should have readable names, either
  handwritten or generated from user-defined macro.

2. Convertions between types should be defined explictly, either handwritten or
  generated from user-defined macro. They work in the way analogy to `friend`
  keyword in C++.

People using exchangeable enums believe it is in such a case that:

1. An ad-hoc enum may or may not worth a readable name.

2. Convertions between ad-hoc enums should be derived from `Exchange`, which
  works in the way analogy to `pub` keyword.

Always naming an ad-hoc enum and generating convertions for it will mislead
readers considering it being deliberate unless they finish reading all the code.

Take two analogies:

1. What if we are not allowed to use closures and local defined functions, but
have to use ad-hoc structs and functions defined far away from their only
invokings, for mimicing closures?

2. What if we are not allowed to use `pub`/`pub(crate)`/`pub(super)`, but have
to explicitly authorize all the possible `friend`s of a certain field?

## Misuse: trait objects

People misusing trait objects believe it is in such a case that:

1. All types, including ad-hoc enums, should implement some trait and may be
  categorized to a hierarchy of traits.

2. All types should be erased, and accesses should be done via public interface.

It is reasonable to use trait objects to express a set of unpredicable elements
having a set of related methods in a trait. Using it to express a set of
predicable elements having unrelated functions is possible, but is a concept
mismatch, which may potentially cause pitfalls:

1. Useless methods in trait.

2. Unnecessary boxing and `'static` lifetime bounds.

3. non-straightforward down-casting.

These are all non-issues for ad-hoc enums to express a set of predicable
elements having unrelated functions.

## sealed traits

Enums are for exstential types while traits are for extending. Another idea is
to fill in the gap with expressing exstential types in trait syntax, aka
"sealed traits". This method has some issues with generics:

1. Generics allowed on implementors that are not bound by the trait will cause
  unsized `dyn Trait`s.

2. If generics allowed on implementors must be bound by the trait, the
  "sealed trait" syntax is more like an enum that enumerates variant types'
  generics, which is a less expressive syntax compare to a plain enum
  enumerating the exhausted and exact variant types in one place.

It is a proof that `enum` is best suitable in such usecases than `trait`.

# Prior art

Concepts similar with ad-hoc enum exist in other language. One example is
union types in Typed Racket. However, it supports more powerful type inferences
such as `Enum!(A)` => `A`, `Enum!(A,A)` => `Enum!(A)`, `Enum!(A,Enum!(B,C))` =>
`Enum!(A,B,C)`. All these seems to bring significant changes to Rust internals.

The [`frunk_core`](https://docs.rs/frunk_core/) library provides `coproduct`
which is similar with ad-hic enums, by which this RFC is inspired. However it
aims at generic programming and the coproduct is nested `enum`s, not supporting
pattern matching.

# Unresolved questions

A particular use case, in which ad-hoc enums are collecting variants
implementing the same trait and act if the enums themselves had implemented it,
may require more elegant solution, than macro-generated `impl`s. 

# Future possibilities

Other proc-macro derives may be proposed. For example, to support automatic
flattening or duplicated variant types, a `#[derive(Sum)]` could be introduced
in the future.

It seems to be a compatible, fine-grained, and practical way to introducing
separate derives. 
