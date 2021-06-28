# Carbon: Generics - Terminology

<!--
Part of the Carbon Language project, under the Apache License v2.0 with LLVM
Exceptions. See /LICENSE for license information.
SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
-->

<!-- toc -->

## Table of contents

-   [Parameterized language constructs](#parameterized-language-constructs)
-   [Generic versus template parameters](#generic-versus-template-parameters)
    -   [Polymorphism](#polymorphism)
        -   [Parametric polymorphism](#parametric-polymorphism)
        -   [Compile-time duck typing](#compile-time-duck-typing)
        -   [Ad-hoc polymorphism](#ad-hoc-polymorphism)
    -   [Constrained genericity](#constrained-genericity)
    -   [Definition checking](#definition-checking)
        -   [Complete definition checking](#complete-definition-checking)
        -   [Early versus late type checking](#early-versus-late-type-checking)
-   [Implicit parameter](#implicit-parameter)
-   [Interface](#interface)
    -   [Structural interfaces](#structural-interfaces)
    -   [Nominal interfaces](#nominal-interfaces)
-   [Impls: Implementations of interfaces](#impls-implementations-of-interfaces)
-   [Compatible types](#compatible-types)
-   [Subtyping and casting](#subtyping-and-casting)
-   [Adapting a type](#adapting-a-type)
-   [Type erasure](#type-erasure)
-   [Facet type](#facet-type)
-   [Extending/refining an interface](#extendingrefining-an-interface)
-   [Witness tables](#witness-tables)
    -   [Dynamic-dispatch witness table](#dynamic-dispatch-witness-table)
    -   [Static-dispatch witness table](#static-dispatch-witness-table)
-   [Instantiation](#instantiation)
-   [Specialization](#specialization)
    -   [Template specialization](#template-specialization)
    -   [Generic specialization](#generic-specialization)
-   [Conditional conformance](#conditional-conformance)
-   [Interface type parameters versus associated types](#interface-type-parameters-versus-associated-types)
-   [Type constraints](#type-constraints)
-   [Type-type](#type-type)

<!-- tocstop -->

## Parameterized language constructs

Generally speaking, when we talk about either templates or a generics system, we
are talking about generalizing some language construct by adding a parameter to
it. Language constructs here primarily would include functions and types, but we
may want to support parameterizing other language constructs like
[interfaces](#interface-type-parameters-versus-associated-types).

This parameter broadens the scope of the language construct on an axis defined
by that parameter, for example it could define a family of functions instead of
a single one.

## Generic versus template parameters

When we are distinguishing between generics and templates in Carbon, it is on an
parameter by parameter basis. A single function can take a mix of regular,
generic, and template parameters.

-   **Regular parameters**, or "dynamic parameters", are designated using the
    "&lt;name>`:` &lt;type>" syntax (or "&lt;value>").
-   **Generic parameters** are temporarily designated using a `$` between the
    type and the name (so it is "&lt;name>`:$` &lt;type>"). However, this is a
    placeholder syntax, subject to change. Some possibilities that have been
    suggested are: `:!`, `:@`, `:#`, and `::`.
-   **Template parameters** are temporarily designated using "&lt;name>`:$$`
    &lt;type>", for similar reasons.

Expected difference between generics and templates:

<table>
  <tr>
   <td><strong>Generics</strong>
   </td>
   <td><strong>Templates</strong>
   </td>
  </tr>
  <tr>
   <td>bounded parametric polymorphism
   </td>
   <td>compile-time duck typing and ad-hoc polymorphism
   </td>
  </tr>
  <tr>
   <td>constrained genericity
   </td>
   <td>optional constraints
   </td>
  </tr>
  <tr>
   <td>name lookup resolved for definitions in isolation ("early")
   </td>
   <td>some name lookup may require information from calls (name lookup may be "late")
   </td>
  </tr>
  <tr>
   <td>sound to typecheck definitions in isolation ("early")
   </td>
   <td>complete type checking may require information from calls (may be "late")
   </td>
  </tr>
  <tr>
   <td>supports separate type checking; may also support separate compilation, for example when implemented using dynamic witness tables
   </td>
   <td>separate compilation only to the extent that C++ supports it
   </td>
  </tr>
  <tr>
   <td>allowed but not required to be implemented using dynamic dispatch
   </td>
   <td>does not support implementation by way of dynamic dispatch, just static by way of <a href="#instantiation">instantiation</a>
   </td>
  </tr>
  <tr>
   <td>monomorphization is an optional optimization that cannot render the program invalid
   </td>
   <td>monomorphization is mandatory and can fail, resulting in the program being invalid
   </td>
  </tr>
</table>

### Polymorphism

Generics and templates provide different forms of
[polymorphism](<https://en.wikipedia.org/wiki/Polymorphism_(computer_science)>)
than object-oriented programming with inheritance. That uses
[subtype polymorphism](https://en.wikipedia.org/wiki/Subtyping) where different
descendants, or "subtypes", of a base class can provide different
implementations of a method, subject to some compatibility restrictions on the
signature.

#### Parametric polymorphism

Parametric polymorphism
([Wikipedia](https://en.wikipedia.org/wiki/Parametric_polymorphism)) is when a
function or a data type can be written generically so that it can handle values
_identically_ without depending on their type.
[Bounded parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism#Bounded_parametric_polymorphism)
is where the allowed types are restricted to satisfy some constraints. Within
the set of allowed types, different types are treated uniformly.

#### Compile-time duck typing

Duck typing ([Wikipedia](https://en.wikipedia.org/wiki/Duck_typing)) is when the
legal types for arguments are determined implicitly by the usage of the values
of those types in the body of the function. Compile-time duck typing is when the
usages in the body of the function are checked at compile-time, along all code
paths. Contrast this with ordinary duck typing in a dynamic language such as
Python where type errors are only diagnosed at runtime when a usage is reached
dynamically.

#### Ad-hoc polymorphism

Ad-hoc polymorphism
([Wikipedia](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism)), also known as
"overloading", is when a single function name has multiple implementations for
handling different argument types. There is no enforcement of any consistency
between the implementations. For example, the return type of each overload can
be arbitrary, rather than being the result of some consistent rule being applied
to the argument types.

Templates work with ad-hoc polymorphism in two ways:

-   A function with template parameters can be
    [specialized](#template-specialization) in
    [C++](https://en.cppreference.com/w/cpp/language/template_specialization) as
    a form of ad-hoc polymorphism.
-   A function with template parameters can call overloaded functions since it
    will only resolve that call after the types are known.

In Carbon, we expect there to be a compile error if overloading of some name
prevents a generic function from being typechecked from its definition alone.
For example, let's say we have some overloaded function called `F` that has two
overloads:

```
fn F[T:$$ Type](x: T*) -> T;
fn F(x: Int) -> Bool;
```

A generic function `G` can call `F` with a type like `T*` that can not possibly
call the `F(Int)` overload for `F`, and so it can consistently determine the
return type of `F`. But `G` can't call `F` with an argument that could match
either overload. (It is undecided what to do in the situation where `F` is
overloaded, but the signatures are consistent and so callers could still
typecheck calls to `F`. This still poses problems for the dynamic strategy for
compiling generics.)

### Constrained genericity

We will allow some way of specifying constraints as part of a function (or type
or other parameterized language construct). These constraints are a limit on
what callers are allowed to pass in. The distinction between constrained and
unconstrained genericity is whether the body of the function is limited to just
those operations that are guaranteed by the constraints.

With templates using unconstrained genericity, you may perform any operation in
the body of the function, and they will be checked against the specific types
used in calls. You can still have constraints, but they are optional. They will
only be used to resolve overloaded calls to the template and provide clearer
error messages.

With generics using constrained genericity, the function body can be checked
against the signature at the time of definition. Note that it is still perfectly
permissible to have no constraints on a type; that just means that you can only
perform operations that work for all types (such as manipulate pointers to
values of that type) in the body of the function.

### Definition checking

Definition checking is the process of semantically checking the definition of
parameterized code for correctness _independently_ of any particular arguments.
It includes type checking and other semantic checks. It is possible, even with
templates, to check semantics of expressions that are not dependent on any
template parameter in the definition. Adding constraints to template parameters
and/or switching them to be generic allows the compiler to increase how much of
the definition can be checked. Any remaining checks are delayed until
[instantiation](#instantiation), which can fail.

#### Complete definition checking

Complete definition checking is when the definition can be _fully_ semantically
checked, including type checking. It is an especially useful property because it
enables _separate_ semantic checking of the definition, a prerequisite to
separate compilation. It also enables implementation strategies that don’t
instantiate the implementation (for example, [type erasure](#type-erasure) or
[dynamic-dispatch witness tables](#dynamic-dispatch-witness-table)).

#### Early versus late type checking

Early type checking is where expressions and statements are type checked when
the definition of the function body is compiled, as part of definition checking.
This occurs for regular and generic values.

Late type checking is where expressions and statements may only be fully
typechecked once calling information is known. Late type checking delays
complete definition checking. This occurs for template dependent values.

## Implicit parameter

An implicit parameter is listed in the optional `[` `]` section right after the
function name in a function signature:

`fn` &lt;name> `[` &lt;implicit parameters> `](` &lt;explicit parameters `) ->`
&lt;return type>

Implicit arguments are determined as a result of pattern matching the explicit
argument values (usually the types of those values) to the explicit parameters.
Note that function signatures can typically be rewritten to avoid using implicit
parameters:

```
fn F[T:$$ Type](value: T);
// is equivalent to:
fn F(value: (T:$$ Type));
```

See more [here](overview.md#implicit-parameters).

## Interface

An interface is an API constraint used in a function signature to provide
encapsulation. Encapsulation here means that callers of the function only need
to know about the interface requirements to call the function, not anything
about the implementation of the function body, and the compiler can check the
function body without knowing anything more about the caller. Callers of the
function provide a value that has an implementation of the API and the body of
the function may then use that API (and nothing else).

### Structural interfaces

A "structural" interface is one where we say a type satisfies the interface as
long as it has members with a specific list of names, and for each name it must
have some type or signature. A type can satisfy a structural interface without
ever naming that interface, just by virtue of having members with the right
form.

### Nominal interfaces

A "nominal" interface is one where we say a type can only satisfy an interface
if there is some explicit statement saying so, for example by defining an
[impl](#impls-implementations-of-interfaces). This allows "satisfies the
interface" to have additional semantic meaning beyond what is directly checkable
by the compiler. For example, knowing whether the "Draw" function means "render
an image to the screen" or "take a card from the top of a deck of cards"; or
that a `+` operator is commutative (and not, say, string concatenation).

We use the "structural" versus "nominal" terminology as a generalization of the
same terms being used in a
[subtyping context](https://en.wikipedia.org/wiki/Subtyping#Subtyping_schemes).

## Impls: Implementations of interfaces

An _impl_ is an implementation of an interface for a specific type. It is the
place where the function bodies are defined, values for associated types, etc.
are given. A given generics programming model may support default impls, named
impls, or both. Impls are mostly associated with nominal interfaces; structural
interfaces define conformance implicitly instead of by requiring an impl to be
defined.

## Compatible types

Two types are compatible if they have the same notional set of values and
represent those values in the same way, even if they expose different APIs. The
representation of a type describes how the values of that type are represented
as a sequence of bits in memory. The set of values of a type includes properties
that the compiler can't directly see, such as invariants that the type
maintains.

We can't just say two types are compatible based on structural reasons. Instead,
we have specific constructs that create compatible types from existing types in
ways that encourage preserving the programmer's intended semantics and
invariants, such as implementing the API of the new type by calling (public)
methods of the original API, instead of accessing any private implementation
details.

## Subtyping and casting

Both subtyping and casting are different names for changing the type of a value
to a compatible type.

[Subtyping](https://en.wikipedia.org/wiki/Subtyping) is a relationship between
two types where you can safely operate on a value of one type using a variable
of another. For example, using C++'s object-oriented features, you can operate
on a value of a derived class using a pointer to the base class. In most cases,
you can pass a more specific type to a function that can handle a more general
type. Return types work the opposite way, a function can return a more specific
type to a caller prepared to handle a more general type. This determines how
function signatures can change from base class to derived class, see
[covariance and contravariance in Wikipedia](<https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)>).

In a generics context, we are specifically interested in the subtyping
relationships between [type-types](#type-type). In particular, a type-type
encompasses a set of [type constraints](#type-constraints), and you can convert
a type from a more-restrictive type-type to another type-type whose constraints
are implied by the first. C++ concepts terminology uses the term
["subsumes"](https://en.cppreference.com/w/cpp/language/constraints#Partial_ordering_of_constraints)
to talk about this partial ordering of constraints, but we avoid that term since
it is at odds with the use of the term in
[object-oriented subtyping terminology](https://en.wikipedia.org/wiki/Subtyping#Subsumption).

Note that subtyping is a bit like
[coercion](https://en.wikipedia.org/wiki/Type_conversion), except we want to
make it clear that the data representation of the value is not changing, just
its type as reflected in the API available to manipulate the value.

Casting is indicated explicitly by way of some syntax in the source code. You
might use a cast to switch between [type adaptations](#adapting-a-type), or to
be explicit where an implicit cast would otherwise occur. For now, we are saying
"`x as y`" is the provisional syntax in Carbon for casting the value `x` to the
type `y`. Note that outside of generics, the term "casting" includes any
explicit type change, including those that change the data representation.

## Adapting a type

A type can be adapted by creating a new type that is
[compatible](#compatible-types) with an existing type, but has a different API.
In particular, the new type might implement different interfaces or provide
different implementations of the same interfaces.

Unlike extending a type (as with C++ class inheritance), you are not allowed to
add new data fields onto the end of the representation -- you may only change
the API. This means that it is safe to [cast](#subtyping-and-casting) a value
between those two types without any dynamic checks or danger of
[object slicing](https://en.wikipedia.org/wiki/Object_slicing).

This is called "newtype" in Rust, and is used for capturing additional
information in types to improve type safety of move some checking to compile
time ([1](https://doc.rust-lang.org/rust-by-example/generics/new_types.html),
[2](https://doc.rust-lang.org/book/ch19-04-advanced-types.html#using-the-newtype-pattern-for-type-safety-and-abstraction),
[3](https://www.worthe-it.co.za/blog/2020-10-31-newtype-pattern-in-rust.html))
and as a workaround for Rust's orphan rules for coherence.

## Type erasure

"Type erasure" is where a type's API is replaced by a subset. Everything outside
of the preserved subset is said to have been "erased". This can happen in a
variety of contexts including both generics and runtime polymorphism. For
generics, type erasure restricts a type to just the API required by the
constraints on a generic function.

An example of type erasure in runtime polymorphism in C++ is casting from a
pointer of a derived type to a pointer to an abstract base type. Only the API of
the base type is available on the result, even though the implementation of
those methods still come from the derived type.

The term "type erasure" can also refer to
[the specific strategy used by Java to implement generics](https://en.wikipedia.org/wiki/Generics_in_Java).
which includes erasing the identity of type parameters. This is not the meaning
of "type erasure" used in Carbon.

## Facet type

A facet type is a [compatible type](#compatible-types) of some original type
written by the user, that has a specific API. This API might correspond to a
specific [interface](#interface), or the API required by particular
[type constraints](#type-constraints). In either case, the API can be specified
using a [type-type](#type-type). Casting a type to a type-type results in a
facet type, with data representation matching the original type and API matching
the type-type.

Casting to a facet type is one way of modeling compile-time
[type erasure](#type-erasure) when calling a generic function. It is also a way
of accessing APIs for a type that would otherwise be hidden, possibly to avoid a
name conflict or because the implementation of that API was external to the
definition of the type.

A facet type associated with a specific interface, corresponds to the
[impl](#impls-implementations-of-interfaces) of that interface for the type.
Using such a facet type removes ambiguity about where to find the declaration
and definition of any accessed methods.

## Extending/refining an interface

An interface can be extended by defining an interface that includes the full API
of another interface, plus some additional API. Types implementing the extended
interface should automatically be considered to have implemented the narrower
interface.

## Witness tables

For witness tables, values passed to a generic parameter are compiled into a
table of required functionality. That table is then filled in for a given
passed-in type with references to the implementation on the original type. The
generic is implemented using calls into entries in the witness table, which turn
into calls to the original type. This doesn't necessarily imply a runtime
indirection: it may be a purely compile-time separation of concerns. However, it
insists on a full abstraction boundary between the generic user of a type and
the concrete implementation.

A simple way to imagine a witness table is as a struct of function pointers, one
per method in the interface. However, in practice, it's more complex because it
must model things like associated types and interfaces.

Witness tables are called "dictionary passing" in Haskell. Outside of generics,
a [vtable](https://en.wikipedia.org/wiki/Virtual_method_table) is a witness
table that witnesses that a class is a descendant of an abstract base class, and
is passed as part of the object instead of separately.

### Dynamic-dispatch witness table

For dynamic-dispatch witness tables, actual function pointers are formed and
used as a dynamic, runtime indirection. As a result, the generic code **will
not** be duplicated for different witness tables.

### Static-dispatch witness table

For static-dispatch witness tables, the implementation is required to collapse
the table indirections at compile time. As a result, the generic code **will**
be duplicated for different witness tables.

Static-dispatch may be implemented as a performance optimization for
dynamic-dispatch that increases generated code size. The final compiled output
may not retain the witness table.

## Instantiation

Instantiation is the implementation strategy for templates in both C++ and
Carbon. Instantiation explicitly creates a copy of the template code and
replaces the template components with the concrete type and its implementation
operations. It allows duck typing and lazy binding. Instantiation implies
template code **will** be duplicated.

Unlike [static-dispatch witness tables](#static-dispatch-witness-table) and
[monomorphization (as in Rust)](https://doc.rust-lang.org/book/ch10-01-syntax.html#performance-of-code-using-generics),
this is done **before** type checking completes. Only when the template is used
with a concrete type is the template fully type checked, and it type checks
against the actual concrete type after substituting it into the template. This
means that different instantiations may interpret the same construct in
different ways, and that templates can include constructs that are not valid for
some possible instantiations. However, it also means that some errors in the
template implementation may not produce errors until the instantiation occurs,
and other errors may only happen for **some** instantiations.

## Specialization

### Template specialization

Specialization in C++ is essentially overloading in the context of a template.
The template is overloaded to have a different definition for some subset of the
possible template argument values. For example, the C++ type `std::vector<T>`
might have a specialization `std::vector<T*>` that is implemented in terms of
`std::vector<void*>` to reduce code size. In C++, even the interface of a
templated type can be changed in a specialization, as happens for
`std::vector<bool>`.

### Generic specialization

Specialization of generics, or types used by generics, is restricted to changing
the implementation _without_ affecting the interface. This restriction is needed
to preserve the ability to perform type checking of generic definitions that
reference a type that can be specialized, without statically knowing which
specialization will be used.

While there is nothing fundamentally incompatible about specialization with
generics, even when implemented using witness tables, the result may be
surprising because the selection of the specialized generic happens outside of
the witness-table-based indirection between the generic code and the concrete
implementation. Provided all selection relies exclusively on interfaces, this
still satisfies the fundamental constraints of generics.

## Conditional conformance

Conditional conformance is when you have a parameterized type that has one API
that it always supports, but satisfies additional interfaces under some
conditions on the type argument. For example: `Array(T)` might implement
`Comparable` if `T` itself implements `Comparable`, using lexicographical order.

## Interface type parameters versus associated types

Let's say you have an interface defining a container. Different containers will
contain different types of values, and the container API will have to refer to
that "element type" when defining the signature of methods like "insert" or
"find". If that element type is a parameter (input) to the interface type, we
say it is a type parameter; if it is an output, we say it is an associated type.

Type parameter example:

```
interface Stack(ElementType:$ Type)
  fn Push(this: Self*, value: ElementType);
  fn Pop(this: Self*) -> ElementType;
}
```

Associated type example:

```
interface Stack {
  var ElementType:$ Type;
  fn Push(this: Self*, value: ElementType);
  fn Pop(this: Self*) -> ElementType;
}
```

Associated types are particularly called for when the implementation controls
the type, not the caller. For example, the iterator type for a container is
specific to the container and not something you would expect a user of the
interface to specify.

```
interface Iterator { ... }
interface Container {
  // This does not make sense as an parameter to the container interface,
  // since this type is determined from the container type.
  var IteratorType:$ Iterator;
  ...
  fn Insert(this: Self*, position: IteratorType, value: ElementType);
}
struct ListIterator(ElementType:$ Type) {
  ...
  impl Iterator;
}
struct List(ElementType:$ Type) {
  // Iterator type is determined by the container type.
  var IteratorType:$ Iterator = ListIterator(ElementType);
  fn Insert(this: Self*, position: IteratorType, value: ElementType) {
    ...
  }
  impl Container;
}
```

Since type parameters are directly under the user's control, it is easier to
express things like "this type parameter is the same for all these interfaces",
and other type constraints.

If you have an interface with type parameters, there is a question of whether a
type can have multiple impls for different combinations of type parameters, or
if you can only have a single impl (in which case you can directly infer the
type parameters given just a type implementing the interface). You can always
infer associated types.

## Type constraints

Type constraints restrict which types are legal for template or generic
parameters or associated types. They help define semantics under which they
should be called, and prevent incorrect calls.

In general there are a number of different type relationships we would like to
express, for example:

-   This function accepts two containers. The container types may be different,
    but the element types need to match.
-   For this container interface we have associated types for iterators and
    elements. The iterator type's element type needs to match the container's
    element type.
-   An interface may define an associated type that needs to be constrained to
    implement some interfaces.
-   This type parameter must be [compatible](#compatible-types) with another
    type. You might use this to define alternate implementations of a single
    interfaces, such as sorting order, for a single type.

Note that type constraints can be a restriction on one type parameter, or can
define a relationship between multiple type parameters.

## Type-type

A type-type is the type used when declaring some type parameter. It foremost
determines which types are legal arguments for that type parameter, also known
as [type constraints](#type-constraints). For template parameters, that is all a
type-type does. For generic parameters, it also determines the API that is
available in the body of the function. Calling a function with a type `T` passed
to a generic type parameter `U` with type-type `I`, ends up setting `U` to the
facet type `T as I`. This has the API determined by `I`, with the implementation
of that API coming from `T`.