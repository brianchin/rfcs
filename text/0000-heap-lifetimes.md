- Feature Name: heap_lifetimes
- Start Date: 2015-11-17
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow for fields within a single struct to borrow from other fields whose
contents are on the heap and will not move. Adds syntax and semantics necessary
to create lifetime contexts within structs, and to safely borrow from those
contexts when appropriate.

# Motivation
[motivation]: #motivation

Lifetimes currently are only valid during some range of time in the execution
of a piece of code. While some extensions like SEME will allow these to not be
exclusively nested, they are still represented as a stack-only concept. There
can be no safe borrowing within the scope of a data structure, such as those
formed by internal references.

In general, internal references are unsafe as Rust's general move semantics
require all structs to be byte-movable. Thus any references directly to the
memory of a struct may change, and thus borrowed reference to fields on the
stack are inherently unsafe. However, one case where this is not true is
heap-allocated data. Take for instance the following struct:

```rust
struct Foo<T> {
  owned_box: Box<T>,
  borrowed_ref: &T,
}
```

This is, of course, illegal code as written as we don't specify the lifetime
for the `borrowed_ref` field. Ignoring this for the moment, assume that
`borrowed_ref` points to the contents of `owned_box`. The approximate binary
representation is:

```
+-----------+--------------+
| owned_box | borrowed_ref |
+-----------+--------------+
    |                |
--- | -------------- | -----         
    V     HEAP       |
+----------------+   |
| Stored T Value |<--+
+----------------+
```

Since our Foo<T> value only contains pointers to something on the heap, this
struct is actually safe to move.

This can be implemented with unsafe code in a safe way, but generally must be
done once for each example. There already exists a crate which does this for
some simple types (such as traits and slices) called owning_ref. However, it
does not apply for the general usages of heap lifetimes. `owning_ref` allows the
type of the value that uses the heap-allocated value's lifetime to be a
simple reference (e.g. `&'a T`), but not a value which uses the lifetime of the
heap allocated value as a lifetime parameter (e.g. `T<'a>`).

As a concrete example of the latter, let's attempt to write a data type as a
generalization of strong and weak Rc's that provides an owning pointer to a
type, and weak references. We could define the basics as follows:

```rust
struct Own<T>(Rc<RefCell<T>>);

struct BorrowRef<T>(Weak<RefCell<T>>);
```

With a `BorrowRef`, we want to be able to borrow the contents safely,
ensuring that we don't have a read on it. For this, we need to write a guard
struct that had an appropriate deref. But when we try, we fail:

```rust
struct BorrowGuard<T> {
  strong_ref: Rc<RefCell<T>>,
  read_guard: Ref<T>, // ERROR: This needs a lifetime!
}
```

This fails, as Ref requires a lifetime of a RefCell. As before, we can write
unsafe code to implement this correctly, but if we had a way of handling
heap lifetimes, then this could be handled with the following (using
placeholder syntax):

```rust
struct BorrowGuard<T> {
  strong_ref: Rc<RefCell<T>> as 'a,
  read_guard: Ref<'a, T>, // Much Better
}
```

We'll cover other details of the semantics in the detailed design section below,
but notice that this is now a safely movable struct that can be dropped without
any unsafety.

# Detailed design
[design]: #detailed-design

Aspects of this design are borrowed from the `owning_ref` crate, created by
@kimundi.

## Marker Traits

We define a new marker trait `StableDeref` as follows:

```rust
unsafe trait StableDeref : Deref {}
```

This indicates that the deref of this type will always return the same
reference to memory, even between moves. This must be unsafe, as it is not
generally true with Deref, and if applied incorrectly can cause use-after-move
errors.

## Struct Definitions

**Note**: This following syntax is a temporary placeholder for whatever is
actually chosen, and is in fact guaranteed to be awful.

When a type `Type` implements the `StableDeref` type, then the following can be
defined:

```rust
struct Foo {
  // ...
  fieldName: Type as 'a, // 'a MUST be a unique lifetime name in the
                         // context of the struct.
  // ...
  otherField: BorrowType<'a>, // Can be used only in fields _after_ the
                              // defining field.
  // ...
}
```

## Struct Use

Any instance of such a struct can be used normally. Borrowing from the
lifetime of the struct implies the ability to temporarily borrow the internal
lifetime. Thus code like the following would be fine.

```rust
fn borrow_func(foo: &Foo) {
  do_stuff_with_borrow_type(&foo.otherField);
}
```

*QUESTION*: What can be/should be done with mutable borrows? Should these kinds
of borrows only work for immutable borrows?

## Struct Creation

To create a borrow of this sort, we need to move a base and a borrowed reference
into place at the same time. The code would look something like this:

```rust
let t = Type::new();
let borrow_t = t.borrow(); // Note: Compiler understands that the borrow is
                           // from the contained object.

// Move both t and borrow_t into a Foo object.
let foo = Foo {
  fieldName: t,
  otherField: borrow_t,
}
```

## Struct Dropping

When dropping a struct, fields that depend on the lifetime must be dropped
before the field which provides the lifetime. No other requirements are made
about the order of dropped fields (at least within this RFC), but this is
compatible with any attempts
[to have struct fields dropped in reverse order](https://github.com/rust-lang/rfcs/issues/744).

## Struct Destruction

When a struct is destructed (not dropped), the moved values maintain the same
lifetime relationship as before:

```rust
let outer_t = {
  let Foo { fieldName: t, otherField: borrow_t } = foo;
  t.borrow_mut(); // ERROR: already borrowed by borrow_t
  t
}
outer_t.borrow_mut(); // ERROR: should succeed, as borrow_t has been dropped.
```

# Drawbacks
[drawbacks]: #drawbacks

## Complexity

While it does not necessarily add much new syntax to the language, it does
extend the meaning of lifetimes as used currently by Rust. Some of the static
analysis may be more complicated than initially estimated. In addition, proof
that these semantics are safe (modulo safe implementation of StableDeref) may
be difficult and, at worst, possibly unsound.

## Applicability

Although I have brought up two examples of places where this could be useful,
it's possible that these are nearly the limit of how this feature could be
applied to valid code. As such, even if possible to add, the bang-per-buck
ratio may be too low to give a good reason to implement this approach.

# Alternatives
[alternatives]: #alternatives

## Crates for Necessary Applications

If there are a finite number of ways that this feature may be applied, then
it's possible that implementing those applications using unsafe code may be
the ideal solution. Thus no change to the language is necessary, and crates.io
would provide the safe functionality without further costs.

Considering the existing crate of owning_ref, it appears with the current
rust typesystem that such a general library is not possible. While references
can be borrowed easily enough, it's not possible for all types that take a
lifetime parameter to create a generic framework to do so. That being said,
how often this happens, and how much work is needed to get around it is another
issue entirely.

# Unresolved questions
[unresolved]: #unresolved-questions

## Can we only have immutable borrows implemented this way?

Would it be reasonable to have mutable borrows as an option? This would
naturally require an additional trait that also inherits from DerefMut, but the
use of this is not completely clear. Similar to how Rc only allows for immutable
borrows, yet uses RefCells to provide runtime support for interior mutability,
that could be the better story here.

## Cloneable types

A further division (again, as part of the design of `owning_ref`) between these
kinds of internal references is whether or not the heap-allocated structure
can be cloned while maintaining the same stability guarantee. `Box`, for
example does not provide the guarantee, as the heap-value of the cloned box is
similarly cloned from the original value. On the other hand, Rc does not clone
the value, and a cloned Rc points to the same heap-allocated data as the first.

`owning_ref` supports this by providing another unsafe trait. Similarly, we
could define it analagously like so:

```rust
unsafe trait StableCloneableDeref : StableDeref + Clone {}
```

If a heap-owned type implements this, then the structure _should_ be able to
be cloned. This does add extra complexity to the semantics that must be
supported, as well as adding more mechanics to the Clone deriver.
