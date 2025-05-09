- Feature Name: `const_trait_methods`
- Start Date: 2024-12-13
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#67792](https://github.com/rust-lang/rust/issues/67792)

# Summary
[summary]: #summary

Make trait methods callable in const contexts. This includes the following parts:

* Allow marking `trait` declarations as const-implementable.
* Allow marking `trait` impls as `const`.
* Allow marking trait bounds as `const` to make methods of them callable in const contexts.

Fully contained example:

```rust
const trait Default {
    const fn default() -> Self;
}

impl const Default for () {
    const fn default() {}
}

struct Thing<T>(T);

impl<T: const Default> const Default for Thing<T> {
    const fn default() -> Self { Self(T::default()) }
}

const fn default<T: const Default>() -> T {
    T::default()
}

fn compile_time_default<T: const(always) Default>() -> T {
    const { T::default() }
}

const _: () = Default::default();

fn main() {
    let () = default();
    let () = compile_time_default();
    let () = Default::default();
}
```

# Motivation
[motivation]: #motivation

Const code is currently only able to use a small subset of Rust code, as many standard library APIs and builtin syntax things require calling trait methods to work.
As an example, in const contexts you cannot use even basic equality on anything but primitives:

```rust
const fn foo() {
    let a = [1, 2, 3];
    let b = [1, 2, 4];
    if a == b {} // ERROR: cannot call non-const operator in constant functions
}
```

## Background

This RFC requires familarity with "const contexts", so you may have to read [the relevant reference section](https://doc.rust-lang.org/reference/const_eval.html#const-context) first.

Calling functions during const eval requires those functions' bodies to only use statements that const eval can handle. While it's possible to just run any code until it hits a statement const eval cannot handle, that would mean the function body is part of its semver guarantees. Something as innocent as a logging statement would make the function uncallable during const eval.

Thus we have a marker, `const`, to add in front of functions that requires the function body to only contain things const eval can handle. This in turn allows a `const` annotated function to be called from const contexts, as you now have a guarantee it will stay callable.

When calling a trait method, this simple scheme (that works great for free functions and inherent methods) does not work.

Throughout this document, we'll be revisiting the example below. We'll stick to static methods call, though the same problems and solutions apply to methods calls with `self` receivers and `dyn Trait`.

```rust
const fn default<T: Default>() -> T {
    T::default()
}

// Could also be `const fn`, but that's an orthogonal change
fn compile_time_default<T: Default>() -> T {
    const { T::default() }
}
```

Neither of the above should (or do) compile.
The first, because you could pass any type T whose default impl could:

* Mutate a global static,
* Read from a file,
* Just allocate memory,
* Fire all the nukes.

Which are all not possible right now in const code, and some can't be done in Rust in const code at all.

It should be possible to write `default` in a way that allows it to be called in const contexts
for types whose `Default` impl's `default` method satisfies all rules that `const fn` must satisfy
(including some annotation that guarantees this won't break by accident).
It must always be possible to call `default` outside of const contexts with no limitations on the generic parameters that may be passed.

Similarly it should be possible to write `compile_time_default` in a way that also requires calls
outside of const contexts to only pass generic parameters whose `Default::default` method satisifies
the usual `const fn` rules. This is necessary in order to allow a const block
(which can access generic parameters) in the function body to invoke methods on the generic parameter.

So, we need some annotation that differentiates a `T: Default` bound from one that gives us the guarantees we're looking for.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Nomenclature and new syntax concepts

### Const traits

All traits can be declared as `const`. Doing so is not a breaking change, and doesn't invalidate existing impls.

```rust
const trait Trait {
    // ...
}
```

The methods of a `const` trait must all be declared as `const`, which means their default bodies, if any, will be const contexts:

```rust
const trait Trait {
    const fn method();

    const fn other_method() {
        // only do const things
    }
}
```

You *cannot* write const methods in a non-const trait.

If you want to declare some methods as `const` and some methods as non-`const`, you can only do so by splitting up your trait into multiple traits, though we expect this to be very uncommon in practice.

Note that on nightly the syntax is

```rust
#[const_trait]
trait Trait {
    fn thing();
}
```

and a result of this RFC would be that we would remove the attribute and add the `const trait` syntax for trait declarations, and the `const fn` syntax for methods.


### Const trait impls

If a trait is declared as `const`, it is possible (but *not* mandatory) to implement it with a `const` impl:

```rust
impl const Trait for Type {
    const fn method() {
        // ...
    }

    const fn other_method() {
        // ...
    }
}
```

As above, the methods of a `const` impl all be declared as `const`, which means their bodies must be const contexts.

You *cannot* write const methods in a non-const impl.


### Const trait bounds

Some items (see below) that can have trait bounds can also have `const Trait` bounds.

Examples:

* `T: const Trait`, requiring any type that `T` is instantiated with to have a `const Trait` impl.
* `trait Foo: const Bar {}`, requiring every type that has an impl for `Foo` to have a `const Trait` impl.
* `trait Foo { type Bar: const Trait; }`, requiring all the impls to provide a type for `Bar` that has a `const Trait` impl.

(`dyn const Trait` and `impl const Trait` are outstide the scope of this RFC. See [the Future Possibilities section](#future-possibilities).)

These bounds means "If this is `const`, then the impl for Trait must be `const`".

Therefore, **only items that establish a potential const context can have `const Trait` bounds**.

This means that this is valid:

```rust
const fn default<T: const Default>() -> T {
    T::default()
}
```

But this will lead to a compile error:

```rust
fn compile_time_default<T: const Default>() -> T {
    //                     ^^^^^
    // error: `const` can only be applied in `const fn`
    const { T::default() }
}
```

#### Always-const trait bounds

If you want to enforce that a function is always called with a type parameter implementing the `const` version of a trait, you must add a `const(always)` trait bound:

```rust
fn compile_time_default<T: const(always) Default>() -> T {
    const { T::default() }
}

let _: u32 = compile_time_default(); // OK
let _: String = compile_time_default(); // OK
let _: MyNonConstType = compile_time_default(); // ERROR
```

Note that `const(always)` is only used for trait bounds, *not* `trait`/`impl`/`fn` declarations.

There is no need for a `const(always) fn` syntax, for example, because a function either establishes a const context or doesn't.


### Const trait example: Default

To see an example of how const traits, const impls, const methods and const bounds come together, see the Default trait:

```rust
const trait Default {
    const fn default() -> Self;
}

impl const Default for () {
    const fn default() {}
}

impl<T: const Default> const Default for Box<T> {
    const fn default() -> Self { Box::new(T::default()) }
}

// When called from a const context, this function requires
// a `const` impl for the type passed for T.
const fn default<T: const Default>() -> T {
    T::default()
}

const _: () = default();

fn main() {
    let _: Box<u32> = default();
}
```

### Const trait example: Add

Another example:

```rust
struct MyStruct<T>(T);

impl<T: const Add<Output = T>> const Add for MyStruct<T> {
    type Output = MyStruct<T>;
    const fn add(self, other: MyStruct<T>) -> MyStruct<T> {
        MyStruct(self.0 + other.0)
    }
}

impl<T> const Add for &MyStruct<T>
where
    for<'a> &'a T: const Add<Output = T>,
{
    type Output = MyStruct<T>;
    const fn add(self, other: &MyStruct<T>) -> MyStruct<T> {
        MyStruct(&self.0 + &other.0)
    }
}
```


## Const contexts and trait resolution

`const fn` items have always been and will stay "always const" functions.

It may appear that a function is suddenly "not a const fn" if it gets passed a type that doesn't satisfy
the constness of the corresponding trait bound. E.g.

```rust
struct Foo;

impl Clone for Foo {
    fn clone(&self) -> Self {
        Foo
    }
}

const fn bar<T: const Clone>(t: &T) -> T { t.clone() }

const BAR: Foo = bar(Foo); // ERROR: `Foo`'s `Clone` impl is not for `const Clone`.
```

But `bar` is still a `const` fn and you can call it from a const context, it will just fail some trait bounds. This similar to:

```rust
const fn dup<T: Copy>(a: T) -> (T, T) {(a, a)}
const FOO: (String, String) = dup(String::new());
```

Here `dup` is always const fn, you'll just get a trait bound failure if the type you pass isn't `Copy`.

**In general, adding or removing const annotations from your items and bounds can only change which programs are accepted or rejected, and will never change program semantics.**

Rust has no close equivalent to SFINAE or method overloading which would let you say "call this method if it's `const`, otherwise call this other method", and has historically rejected these kinds of features over and over again.


## Derives

Most of the time you don't want to write out your impls by hand, but instead derive them as the implementation is obvious from your data structure.

```rust
#[derive_const(PartialEq, Eq)]
struct MyStruct<T>(T);
```

generates:

```rust
impl<T: const PartialEq> const PartialEq for MyStruct<T> {
    const fn eq(&self, other: &Rhs) -> bool {
        self.0 == other.0
    }
}

impl<T: const Eq> const Eq for MyStruct<T> {}
```

For this RFC, we stick with `derive_const`, because it interacts with other ongoing bits of design work (e.g., RFC 3715)
and we don't want to have to resolve all design questions at once to do anything.
We encourage another RFC to integrate const/unsafe and potentially other modifiers into the derive syntax in a better way.
If this lands prior to stabilization, we should implement the const portion of it, otherwise we'll deprecate `derive_const`.


## Notable traits

### `PartialEq`

You can use `==` operators on most types from libstd from within const contexts.

```rust
const _: () = {
    let a = [1, 2, 3];
    let b = [4, 5, 6];
    assert!(a != b);
};
const _: () = {
    let a = Some(42);
    let b = a;
    assert!(a == b);
};
```

Note that the use of `assert_eq!` is waiting on `Debug` impls becoming `const`, which
will likely be tracked under a separate feature gate under the purview of T-libs.
Similarly other traits will be made `const` over time, but doing so will be
unblocked by this feature.


### `const Destruct`

The `Destruct` trait is a bound for whether a type has drop glue.
This is trivally true for all types.

The `const Destruct` trait enables dropping types within a const context:

```rust
const fn foo<T>(t: T) {
    // ERROR: `t` is dropped here, but we don't know if we can
    // evaluate its `Drop` impl (or that of its fields' types)
}

const fn baz<T: Copy>(t: T) {
    // Fine, `Copy` implies that no `Drop` impl exists
}

const fn bar<T: const Destruct>(t: T) {
    // Fine, we can safely invoke the destructor of `T`.
}
```

When a value of a generic type goes out of scope, it is dropped and its `Drop` impl (if it has one) gets invoked.
This situation seems no different from other trait bounds, except that types can be dropped without implementing `Drop`
(as they can contain types that implement `Drop`). In that case the type's drop glue is invoked.

`const Destruct` trait bounds are satisfied only if the type's `Drop` impl (if any) is `const` and all of the types of
its components are `const Destruct`.

While this means that it's a breaking change to add a type with a non-const `Drop` impl to a type,
that's already true and nothing new:

```rust
pub struct S {
    x: u8,
    y: Box<()>, // Adding this field breaks code.
}

const fn f(_: S) {}
//~^ ERROR: Destructor of `S` cannot be evaluated at compile-time
```


### `const Fn*` traits

All `const fn` implement the corresponding `const Fn()` trait:

```rust
const fn foo<F: const Fn()>(f: F) {
    f()
}

const fn bar() {
    foo(baz)
}

const fn baz() {}
```

Arguments and the return type of such functions and bounds follow the same rules as
their non-const equivalents, so you may have to add `const` bounds to other generic
parameters, too:


```rust
const fn foo<T: const Debug, F: const Fn(T)>(f: F, arg: T) {
    f(arg)
}

const fn bar<T: const Debug>(arg: T) {
    foo(baz, arg)
}

const fn baz<T: const Debug>() {}
```

For closures and them implementing the `Fn` traits, see the [Future possibilities](#future-possibilities) section.


## Crate authors: Making your own custom types easier to use

You can implement the const versions of many standard library traits for your own types.
While it was often possible to write the same code in inherent methods, operators were
covered by traits from `std::ops` and thus not available for const contexts.

The refactor process will usually be:

- Add `const` after the `impl`.
- Make the methods `const fn`.
- Make the various trait bounds `const`.

Assuming your trait implementation doesn't perform I/O or mutate global variables, you shouldn't need to do anything else.
Advanced features like `const(always)` should rarely be needed.

You can also make your traits `const`, to let your crate's users write const implementations. This has one caveat: the default method bodies will now need to be const.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## How does this work in the compiler?

The new `const` and `const(always)` trait bounds desugar to normal trait bounds without modifiers, plus an additional constness bound that has no surface level syntax.

A much more detailed explanation can be found in https://hackmd.io/@compiler-errors/r12zoixg1l#What-now

**TODO: Fill out this section. I don't want to just copy-paste oli-obk's explanation, because I don't know how applicable it is to this version of the RFC.**

While this could be modelled with generic parameters in the type system, that:

* Has been attempted and is really complex (fragile) on the impl side and on the reasoning about things side.
* Appears to permit more behaviours than are desirable (multiple such parameters, math on these parameters, ...), so they need to be prevented, adding more checks.
* Is not necessary unless we'd allow much more complex kinds of bounds. So it can be kept open as a future possibility, but for now there's no need.
* Does not quite work in Rust due to the constness then being early bound instead of late bound, cause all kinds of problems around closures and function calls.
* Technically cause two entirely separate MIR bodies to be generated, one for where the effect is on and one where it is off. On top of that it then theoretically allows you to call the const MIR body from non-const code.

Thus that approach was abandoned after proponents and opponents cooperated in trying to make the generic parameter approach work, resulting in all proponents becoming opponents, too.


### Sites where `const Trait` bounds can be used

* `const fn`
* `impl const Trait` blocks.
* NOT in inherent impls, the individual `const fn` need to be annotated instead
* `const trait` declarations
    * super trait bounds
    * where bounds
    * associated type bounds


### Sites where `const(always) Trait` bounds can be used

Everywhere non-const trait bounds can be written.


### `const` desugaring

In a-mir-formality

```rust
const fn default<T: const Default>() -> T {
    T::default()
}
```

desugars to

```rust
const fn default<T>() -> T
where
    T: Default,
    do: <T as Default>,
{
    T::default()
}
```


### `const(always)` desugaring

In a-mir-formality

```rust
fn compile_time_default<T: const(always) Default>() -> T {
    const { T::default() }
}
```

desugars to

```rust
fn compile_time_default<T>() -> T
where
    T: Default<do: const>,
{
    const { T::default() }
}
```


### Why not both?

```rust
const fn checked_default<T>() -> T
where
    T: const Default,
    T: const(always) Default,
    T: const PartialEq,
{
    let a = const { T::default() };
    let b = T::default();
    if a == b {
        a
    } else {
        panic!()
    }
}
```

Has a redundant bound. `T: const(always) Default` implies `T: const Default`, so while the desugaring will include both (but may filter them out if we deem it useful on the impl side),
there is absolutely no difference (just like specifying `Fn() + FnOnce()` has a redundant `FnOnce()` bound).


## `const` methods and `Self` bounds

Methods essentially have implicit bounds on their `Self` type. For instance:

```rust
const trait Foobar {
    const fn foo();
    const fn bar();
    const fn plop();
}
```

is essentially short for:

```rust
const trait Foobar {
    const fn foo() where Self: const Foobar;
    const fn bar() where Self: const Foobar;
    const fn plop() where Self: const Foobar;
}
```

Users might want to either relax or tighten these implicit bounds:

```rust
const trait Foobar {
    const fn foo() where Self: Foobar;
    const fn bar() where Self: const Foobar;
    const fn plop() where Self: const(always) Foobar;
}
```

This is explicitly out of scope for the RFC, and arguably shouldn't be allowed at all.


## `const Destruct` super trait

The `Destruct` marker trait is used to name the previously unnameable drop glue that every type has.
It has no methods, as drop glue is handled entirely by the compiler,
but in theory drop glue could become something one can explicitly call without having to resort to extracting the drop glue function pointer from a `dyn Trait`.

Traits that have `self` (by ownership) methods, will almost always drop the `self` in these methods' bodies unless they are simple wrappers that just forward to the generic parameters' bounds.

The following never drops `T`, because it's the job of `<T as Add>` to handle dropping the values.

```rust
struct NewType<T>(T);

impl<T: const Add<Output = T>> const Add for NewType<T> {
    type Output = Self;
    const fn add(self, other: Self) -> Self::Output {
        NewType(self.0 + other.0)
    }
}
```

But if any code path could drop a value...

```rust
struct NewType<T>(T, bool);

struct Error;

impl<T: const Add<Output = T>> const Add for NewType<T> {
    type Output = Result<Self, Error>;
    const fn add(self, other: Self) -> Self::Output {
        if self.1 {
            Ok(NewType(self.0 + other.0, other.1))
        } else {
            // Drops both `self.0` and `self.1`
            Err(Error)
        }
    }
}
```

... then we need to add a `const Destruct` bound to `T`, to ensure
`NewType<T>` can be dropped.

This bound in turn will be infectious to all generic users of `NewType` like

```rust
const fn add<T: const Add>(
    a: NewType<T>,
    b: NewType<T>,
) -> Result<NewType<T::Output>, Error> {
    a + b
}
```

which now need a `T: const Destruct` bound, too.
In practice we have noticed that a large portion of APIs will have a `const Destruct` bound.
This bound has little value as an explicit bound that appears almost everywhere.
Especially since it is a fairly straight forward assumption that a type that has trait impls with `const` methods will also have a `Drop::drop` method that is `const` or only contain `const Destruct` types.

In the future we will also want to support `dyn const Trait` bounds, which invariably will require the type to implement `const Destruct` in order to fill in the function pointer for the `drop` slot in the vtable.
While that can in generic contexts always be handled by adding more `const Destruct` bounds, it would be more similar to how normal `dyn` safety
works if there were implicit `const Destruct` bounds for (most?) `const Trait` bounds.

Thus we lint all `const trait`s with `const` methods that take `self` by value to also have a `const Destruct` super trait bound to ensure users don't need to add `const Destruct` bounds everywhere.
Other traits may want to add them, and some traits with `self` by value methods may not want to add them. Since it is not backwards compatible to require or relax that super trait bound later,
we aren't requiring users to choose either, but are suggesting good defaults via lints.


## `const` bounds on `Drop` impls

It is legal to add `const` to `Drop` impls' bounds, even when the struct doesn't have them:

```rust
const trait Bar {
    const fn thing(&mut self);
}

struct Foo<T: Bar>(T);

impl<T: const Bar> const Drop for Foo<T> {
    const fn drop(&mut self) {
        self.0.thing();
    }
}
```

Our usual `Drop` rules enforce that an impl must have the same bounds as the type.

`const` modifiers are special here, because they are only needed in const contexts.
While they cause exactly the divergence that we want to prevent with the `Drop` impl rules:
a type can be declared, but not dropped, because bounds are unfulfilled, this is:

* Already the case in const contexts, just for all types that aren't trivially free of `Drop` types.
* Exactly the behaviour we want.

Extraneous `const Trait` bounds where `Trait` isn't a bound on the type at all are still rejected:

```rust
impl<T: const Bar + const Baz> const Drop for Foo<T> {
    const fn drop(&mut self) {
        self.0.thing();
    }
}
```

errors with

```
error[E0367]: `Drop` impl requires `T: Baz` but the struct it is implemented for does not
  --> src/lib.rs:13:22
   |
13 | impl<T: const Bar + const Baz> Drop for Foo<T> {
   |                     ^^^^^^^^^
   |
note: the implementor must specify the same requirement
  --> src/lib.rs:8:1
   |
8  | struct Foo<T: Bar>(T);
   | ^^^^^^^^^^^^^^^^^^
```

# Drawbacks
[drawbacks]: #drawbacks

## Adding any feature at all around constness

We've reached the point where all critics have agreed that this one kind of effect system is unavoidable since we want to be able to write maintainable generic code for compile time evaluation.

So the main drawback is that it creates interest in extending the system or add more effect systems, as we have now opened the door with an effect system that supports traits.

TODO - Rewrite.
Even though I personally am interested in adding an effect for panic-freedom, I do not think that adding this const effect system should have any bearing on whether we'll add
a panic-freedom effect system or other effect systems in the future. This feature stands entirely on its own, and even if we came up with a general system for many effects that is (e.g. syntactically) better in the
presence of many effects, we'll want the syntax from this RFC as sugar for the very common and simple case.

## It's hard to make constness optional with `#[cfg]`

One cannot `#[cfg]` just the `const` keyword in `const Trait`, and even if we made it possible by sticking with `#[const_trait]` attributes, and also adding the equivalent for impls and functions,
`const Trait` bounds cannot be made conditional with `#[cfg]`. The only real useful reason to have this is to support newer Rust versions with a cfg, and allow older Rust versions to compile
the traits, just without const support. This is surmountable with proc macros that either generate two versions or just generate a different version depending on the Rust version.
Since it's only necessary for a transition period while a crate wants to support both pre-const-trait Rust and
newer Rust versions, this doesn't seem too bad. With a MSRV bump the proc macro usage can be removed again.

# Alternatives
[alternatives]: #alternatives

## What is the impact of not doing this?

We would require everything that wants a const-equivalent to have duplicated traits and not
use `const` fn at all, but use associated consts instead. Similarly this would likely forbid
invoking builtin operators. This same concern had been brought up for the `const fn` stabilization
[7 years ago](https://github.com/rust-lang/rust/issues/24111#issuecomment-385046163).

Basically what we can do is create

```rust
trait ConstDefault {
    const DEFAULT: Self;
}
```

and require users to use

```rust
const FOO: Vec<u8> = ConstDefault::DEFAULT;
```

instead of

```rust
const fn FOO: Vec<u8> = Default::default();
```

This duplication is what this RFC is suggesting to avoid.

Since it has already been possible to do all of this on stable Rust for years, and no major
crates have popped and gotten used widely, I assume that is either because

* it's too much duplication, or
* everyone was waiting for the work (that this RFC wants to stabilize) to finish, or
* both.

So while it is entirely possible that rejecting this RFC and deciding not to go down this route
will lead to an ecosystem for const operations to be created, it would result in duplication and
inconsistencies that we'd rather like to avoid.

Such an ecosystem would also make `const fn` obsolete, as every `const fn` can in theory be represented
as a trait, it would just be very different to use from normal rust code and not really allow nice abstractions to be built.

```rust
const fn add(a: u32, b: u32) -> u32 { a + b }

struct Add<const A: u32, const B: u32>;

impl<const A: u32, const B:u32> Add<A, B> {
    const RESULT: u32 = A + B;
}

const FOO: u32 = add(5, 6);
const BAR: u32 = Add<5, 6>::RESULT;
```

## Use `const Trait` bounds for always-const, other syntax for conditionally-const

Previous proposals have suggested various ways to mark that a bound is "const if called from a const context":

* `const? Trait`
* `?const Trait`
* `~const Trait`
* `(const) Trait`
* `impl<const B: bool> ... const<B> Trait`

This RFC instead attempts to "pave the cowpath": given that most of the time, users will want to express something like "be as const as you reasonably can given your imports", these use-cases should get the simple syntax.

Cases where users want a bound to be const even when called from a non-const context are relatively rare and get the `const(always)` syntax.

Another advantage of this syntax is its explicitness: writing `T: const(always) Clone` clearly expresses "T must implement `const` Clone no matter what", whereas the semantics of e.g. `T: (const) Clone` vs `T: const Clone` are less immediately clear.

### Use `Trait<const>` or `Trait<bikeshed#effect: const>` instead of `const Trait`

To avoid new syntax before paths referring to traits, we could treat the constness as a generic parameter or an associated type.
This means

```rust
const fn foo<T: const Trait + const(always) OtherTrait>(t: T) { ... }
```

could be written as:

```rust
const<C> fn foo<T, const C: bool>(t: T)
where
    T: Trait + OtherTrait,
    <T as Trait>::effect#constness = const<C>,
    <T as OtherTrait>::effect#constness = const<true>,
{
    ...
}
```

We do not know of any cases where such an explicit syntax would be useful.
It only makes sense if you can do math on the bool, which we specifically want to avoid.


### Make all `const fn` arguments `const Trait` by default and require an opt out `?const Trait`

We could default to making all `T: Trait` bounds be const if the function is called from a const context, and require a `T: ?const Trait` opt out
for when a trait bound is only used for its associated types and consts.

This requires an opt-out attribute (e.g. `#[next_const_fn]`) and/or an edition change, as existing `const fn` items already have trait bounds that
do not require const trait impls even if used in const contexts.

So:

```rust
const trait Foo: const Bar + Baz {
    const fn foo();
}

const fn foo<T: const Foo>() {
    <T as Bar>::bar(); // OK
    <T as Baz>::baz(); // ERROR: Baz not const
}
```

could be written as:

```rust
const trait Foo: Bar + ?const Baz {
    fn foo();
}

#[next_const_fn]
const fn foo<T: Foo>() -> T {
    <T as Bar>::bar(); // OK
    <T as Baz>::baz(); // ERROR: Baz not const
}
```

There's a few arguments against that syntax:

- `?Sized` bounds have a history of being error-prone to implement, and the lang team doesn't want to add anything similar (https://github.com/rust-lang/rust/issues/135229, https://github.com/rust-lang/rust/pull/132209).
- It creates potential footguns where you can add more bounds to your function than needed.
- Opt-out bounds in general increase the cognitive load of a feature.

Opt-in bounds are easier to explain and easier to implement correctly by default in the compiler.


## Mixing const and non-const trait methods

The [previous RFC](https://github.com/oli-obk/rfcs/blob/const-trait-impl/text/0000-const-trait-impls.md) proposed a syntax for const traits which lets the user a lot of freedom in mixing const and non-const features:

- `const` methods in non-`const` traits.
- `const` and non-`const` methods in the same trait.
- Trait methods which are either `(const)` or `const`, which essentially means they either have a `Self: const Trait` or `Self: const(always) Trait` bound.

This RFC is much more restrictive:

- `const` traits and `const` impls must *only* have `const` methods.
- non-`const` traits and non-`const` impls must *only* have non-`const` methods.
- `const` methods always have a `Self: const Trait` bound.

While this may be limiting, in practice we expect that the overwhelming majority of traits will not need to mix const and non-const methods.

If a trait does need to mix them, the recommended solution is to split it into multiple traits:

```rust
const trait Foobar {
    fn launch_missiles(&self);
    const(always) compile_time_value();
    const fn foobar(&self);
}

// becomes ...

trait Foo {
    fn launch_missiles(&self);
}

const trait Bar {
    const compile_time_value();
}

const trait Foobar: Foo + const(always) Bar {
    const fn foobar(&self);
}
```

The explicit split lets us see clearly what the relation between these traits is without resorting to special syntax.


## `#[derive_const]` vs `#[const_derive]`

Both macros are plausible variants of `#[derive]` for const traits.

`#[const_derive]` arguably rolls off the tongue better, and is an intutive extension of the nominalization of the "derive" verb: just like we talk about traits and thus const traits, we tend to talk about "derives" and thus "const derives".

`#[derive_const]` follows the trend of keeping the `const` keyword "close" to the trait. We will likely have a `#[derive(const Foo)]`, and in the meantime we could have `#[derive_const(Foo)]`.

However, it really doesn't matter. Either macro will likely be temporary, until we get `#[derive(const Foo)]` or something like it.

This RFC picked `#[derive_const]`.
Please do not debate between either alternative in the comments.


# Prior art
[prior-art]: #prior-art

* [RFC #3762](https://github.com/oli-obk/rfcs/blob/const-trait-impl/text/0000-const-trait-impls.md) by oli-obk, which this document is based on.
* [RFC #2632](https://github.com/rust-lang/rfcs/pull/2632) by oli-obk.
    * While that moved to [FCP](https://github.com/rust-lang/rfcs/pull/2632#issuecomment-481395097), it had concerns raised.
    * [T-lang discussed this](https://github.com/rust-lang/rfcs/pull/2632#issuecomment-567699174) and had the following open concerns:
        * This design has far-reaching implications and we probably aren't going to be able to work them all out in advance. We probably need to start working through the implementation.
        * This seems like a great fit for the "const eval" project group, and we should schedule a dedicated meeting to talk over the scope of such a group in more detail.
        * Similarly, it would be worth scheduling a meeting to talk out this RFC in more detail and make sure the lang team is understanding it well.
        * We feel comfortable going forward with experimentation on nightly even in advance of this RFC being accepted, as long as that experimentation is gated.
    * All of the above have happened in some form.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
    * We've already handled this since the last RFC, there are no more implementation concerns.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
    * This RFC's syntax is entirely unrelated to discussions on effect syntax.
        * If we get an effect system, it may be desirable to allow expressing const traits with the effect syntax, this design is forward compatible with that.
        * If we get an effect system, we will still want this shorthand, just like we allow you to write:
            * `T: Iterator<Item = U>` and don't require `where T: Iterator, <T as Iterator>::Item = U`.
            * `T: Iterator<Item: Debug>` and don't require `where T: Iterator, <T as Iterator>::Item: Debug`.
    * RTN for per-method bounds: `T: Trait<some_fn(..): const Fn(A, B) -> C>` could supplement this feature in the future.
        * Alternatively `where <T as Trait>::some_fn(..): const` or `where <T as Trait>::some_fn \ {const}`.
        * Very verbose (need to specify arguments and return type).
        * Want short hand sugar anyway to make it trivial to change a normal function to a const function by just adding some minor annotations.
        * Significantly would delay const trait stabilization (by years).
        * Usually requires editing the trait anyway, so there's no "can constify impls without trait author opt in" silver bullet.
    * New RTN-like per-method bounds: `T: Trait<some_fn(_): const>`.
        * Unclear if soundly possible.
        * Unclear if possible without incurring significant performance issues for all code (may need tracking new information for all functions out there).
        * Still requires editing traits.
        * Still want the `const Trait` sugar anyway.

## Should we start out without `const Trait` bounds

We do not need to immediately allow using methods on generic parameters of const fn, as a lot of const code is nongeneric.

The following example could be made to work with just const traits and const trait impls.

```rust
struct MyStruct(i32);

impl PartialEq for MyStruct {
    const fn eq(&self, other: &MyStruct) -> bool {
        self.0 == other.0
    }
}

const fn foo() {
    let a = MyStruct(1);
    let b = MyStruct(2);
    if a == b {}
}
```

Things like `Option::map` or `PartialEq` for arrays/tuples could not be made const without const trait bounds,
as they need to actually call the generic `FnOnce` argument or nested `PartialEq` impls.


# Future possibilities
[future-possibilities]: #future-possibilities

## Better derive syntax than `#[derive_const(Trait)]`

Once `unsafe` derives have been finalized, we can separately design const derives and
deprecate `derive_const` at that time (mostly by just removing it from any documents explaining it,
so that the ecosystem slowly migrates, maybe with an actual deprecation warning later).

## `const fn()` pointers

Just like `const fn foo(x: impl const Trait) { x.method() }` and `const fn foo(x: &dyn const Trait) { x.method() }` we want to allow
`const fn foo(f: const fn()) { f() }`.

These require changing the type system, making the constness of a function pointer part of the type.
This in turn implies that an `fn()` function pointer, a `const fn()` function pointer and a `const(always) fn()` function pointer could have
different `TypeId`s, which is something that requires more design and consideration to clarify whether supporting downcasting with `Any`
or just supporting `TypeId` equality checks detecting constness is desirable.

Furthermore `const fn()` pointers introduce a new situation: you can transmute arbitrary values (e.g. null pointers, or just integers) to
`const fn()` pointers, and the type system will not protect you. Instead the const evaluator will reject that when it actually
evaluateds the code around the function pointer or even as late as when the function call happens.

## `const` closures

Closures need explicit opt-in to be callable in const contexts.
You can already use closures in const contexts today to e.g. declare consts of function pointer type.
So what we additionally need is some syntax like `const || {}` to declare a closure that implements
`const Fn()`. See also [this tracking issue](https://github.com/rust-lang/project-const-traits/issues/10)
While it may seem tempting to just automatically implement `const Fn()` (or `const(always) Fn()`) where applicable,
it's not clear that this can be done, and there are definite situations where it can't be done.
As further experimentation is needed here, const closures are not part of this RFC.

## Allow impls to refine any trait's methods

We could allow writing `const fn` in impls without the trait opting into it.
This would not affect `T: Trait` bounds, but still allow non-generic calls.

This is similar to other refinings in impls, as the function still satisfies everything from the trait.

Example: without adjusting `rand` for const trait support at all, users could write

```rust
struct CountingRng(u64);

impl RngCore for CountingRng {
    const fn next_u32(&mut self) -> u32 {
        self.next_u64() as u32
    }

    const fn next_u64(&mut self) -> u64 {
        self.0 += 1;
        self.0
    }

    const fn fill_bytes(&mut self, dest: &mut [u8]) {
        let mut left = dest;
        while left.len() >= 8 {
            let (l, r) = { left }.split_at_mut(8);
            left = r;
            let chunk: [u8; 8] = rng.next_u64().to_le_bytes();
            l.copy_from_slice(&chunk);
        }
        let n = left.len();
        let chunk: [u8; 8] = rng.next_u64().to_le_bytes();
        left.copy_from_slice(&chunk[..n]);
    }

    const fn try_fill_bytes(&mut self, dest: &mut [u8]) -> Result<(), Error> {
        Ok(self.fill_bytes(dest))
    }
}
```

and use it in non-generic code.
It is not clear this is doable soundly for generic methods.

## Macro matcher

In the future, we may want to provide a macro matcher for this optional component of a function declaration or trait declaration, similar to `:vis` for an optional visibility. This would allow macros to match it conveniently, and may encourage forwards compatibility with future things in the same category. However, we should not add such a matcher right away, until we have a clearer picture of what else we may add to the same category.
