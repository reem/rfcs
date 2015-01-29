- Start Date: Jan 29 2014
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

<!-- FIXME: Expand with details -->
Add a new feature called extension classes for the purpose of providing
inherent-like impls on types from foreign crates.

# Motivation

Currently, changing the signature of method on a trait in any way is a
backwards incompatible. As a result, several of the extension traits in std are
a major backwards compatibility hazard, particularly the extension traits used
to attach methods to primitive types.

The use of these traits rather than inherent implementations on `[T]` and `str`
means that it will be impossible to ever change the signatures of these methods
to be more generic. This is undesirable.

# Detailed design

Add extension classes to alleviate this issue, allowing inherent-like impls on
types defined in other crates.

## Temporary Hack

Since extension classes are a large-ish feature and 1.0 is around the corner,
there is a much simpler hack that can be added in the meantime: add a feature
flag called `rustc_inherent_impls_on_primitives` which allows inherent impls
on all primitive types. This obviously breaks coherence, so should never be
stabilized, never used outside `core` or `std`, and will be removed as soon as
extension classes land. In the interim it will just provide a way to remove the
`*Ext` trait currently used for this purpose.

## Extension Classes

<!-- Extension Classes detailed design goes here -->

# Drawbacks

Adds complexity.

# Alternatives

Do nothing and never change the signatures of any of the methods on any
primitive types.

Use the temporary hack and don't add extension classes. This is a simpler
solution but doesn't allow other users to get the same advantage for their own
types and uses.

# Unresolved questions

Syntax.

