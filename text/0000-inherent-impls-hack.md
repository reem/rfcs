- Start Date: Jan 29 2014
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

<!-- FIXME: Expand with details -->
Add a new feature called extension impls for the purpose of providing
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

Add extension impls to alleviate this issue, allowing inherent-like impls on
types defined in other crates.

The theory here is that many people really want free functions "semantically," but because they are not ergonomic in Rust people prefer to use methods.  So we want to allow people to define "method-like" free functions in other crates.  In order to ensure coherence, the new impls must be explicitly imported in order to bring the new methods into scope.  The general idea is that the methods thus defined would work just like inherent implementations--you wouldn't be able to reuse them elsewhere, or make trait objects out of them, but they would provide much nicer syntactic sugar for free functions.

There are several possible approaches to how this would actually be implemented.  One approach would be to simply allow the `impl SomeType` syntax for types not defined in the current module (which is currently disallowed).  Any extension impls thus defined would be automatically imported by `use`ing the module in which the inherent impl was defined (much like enum variants used to work).  If two extension impls conflicted, the UFCS disambiguation syntax would be used with the path to the module after the as: `<Type as path::to::module::with::extension::impl>::method()` or `<Type as path::to::module::with::type>::method()`.  Note that traits and modules cannot have overlapping namespaces, so this will not be ambiguous with the `<Type as Trait>` variant.

## Temporary Hack

Since extension impls are a large-ish feature and 1.0 is around the corner,
there is a much simpler hack that can be added in the meantime: add a feature
flag called `rustc_inherent_impls_on_primitives` which allows inherent impls
on all primitive types. This obviously breaks coherence, so should never be
stabilized, never used outside `core` or `std`, and will be removed as soon as
extension impls land. In the interim it will just provide a way to remove the
`*Ext` trait currently used for this purpose.

## Extension impls

# Drawbacks

* Adds some extra complexity, as it would be a new feature for Rust.
* Nothing else in Rust is imported directly when you import a module, so this would stand in contrast to existing practice.  Might make reexport harder?

# Alternatives

Do nothing and never change the signatures of any of the methods on any
primitive types.

Use the temporary hack and don't add extension impls. This is a simpler
solution but doesn't allow other users to get the same advantage for their own
types and uses.

Another approach is to create a new type of trait, `extension trait Foo`.  This would still have the same characteristics as the extension impls above (it would not be usable in a trait object, it would have to be explicitly imported, etc.), but it could be reused within the current crate, and you would explicitly import the trait itself rather than the module.  This would address point 2 in the drawbacks, but it would also require an extra keyword, making it not 100% backwards compatible (but maybe you could find an alternative kyword).  It also seems a bit weird to have it be reusable at all, since the idea here is to deal with situations where you *don't* want to make the interface reusable.

There are also various syntactic changes that could be made to the proposal, e.g. require `extension impl` to be explicit, or having you write out the path to the full type for UFCS resolution rather than just to its module (but this would have to be made to work for types like `Option<MyType>`, since you can have an extension impl for such types under this proposal).

It might be possible to get the same benefits (free functions with method syntax) by actually letting you call free functions with method syntax.  That approach would need to be developed further to make sure it didn't conflict with type-based name resolution.

# Unresolved questions

* How to reexport inherent impls?  Is just letting it live on the module sufficient, or do we want a better approach?  This is resolved by the extension trait approach, but see the other reservations.
