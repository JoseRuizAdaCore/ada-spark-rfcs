- Feature Name: Structural generic instantiation
- Start Date: 2023-03-03
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This RFC proposes to allow "structural" instantiation of generics, that is to
be able to reference an implicit instance of a generic, that is denoted only by
its actual parameters, rather than by its name.

Motivation
==========

The expected benefits of this feature are:

1. Expressivity. Combined with other features that can be found in the [meta
   RFC](../meta/rfc-improved-generic-instantiations.md), we hope to make
   generic subprograms much more usable, and unblock potential use cases that
   would otherwise require language support to be expressive (Ada 2022's
   `Reduce` comes to mind).

2. Be able to refer to a "unique", structural instance of a generic. For
   example there will be a unique instance of `Ada.Containers.Vectors
   (Positive, Positive)`, and all people refering to it will refer to the same
   instance, which solves a long standing problem in generics, which is the
   ability to structurally reference unique entities.

See the high level RFC for examples.

Guide-level explanation
=======================

You can structurally refer to an implicit instantiation of a generic by naming
it. The (tentative) syntax for naming it is the following:

```ada
Ada.Unchecked_Deallocation [Integer, Integer_Access] (My_Int_Access);
```

By naming the generic, it will be implicitly instantiated, a key point being
that there is only one generic corresponding to `Ada.Unchecked_Deallocation
[Integer, Integer_Access]` at a high level, and every reference to it
references the same entity.

> *Note*
>
> It's not clear that we can actually guarantee that it will be compiled only
> once with a separate compilation model, which is why it is not mentioned
> above, but the goal is clearly to ensure that when possible, and when it's
> not, to minimize the number of instances actually generated.

This syntax does also allow naming parameters:

```ada
Ada.Unchecked_Deallocation [Object => Integer, Name => Integer_Access] (My_Int_Access);

Ada.Unchecked_Deallocation [Name => Integer_Access] (My_Int_Access);
--  NOTE: This relies on parameter inference
```

and empty parameter lists:

```ada
generic procedure Foo (A : Integer) is null;

Foo [] (12);

Ada.Unchecked_Deallocation [] (My_Int_Access);
--  NOTE: This relies on inference from name & type resolution context
```

> *Note*
>
> Do we want to allow `Ada.Unchecked_Deallocation (My_Int_Access)` - so,
> without any explicit syntactic instantiation indication ? Seems nifty and
> possible, but maybe too implicit.

Any generic can be instantiated, be it a package, procedure or function:

```ada
A : Ada.Containers.Vectors [Positive, Positive].Vector;
```

This allows generalized structural typing in Ada, and fixes a long standing
problem regarding generic types and modularity:

```ada
generic
    type Element_Type is private;
package Consume_Elements is
    package Elements_Vectors is new Ada.Containers.Vectors (Positive, Element_Type);

    procedure Consume_Elements (Elements : Elements_Vectors.Vector);
end Consume_Elements;

--  In another package/library

generic
    type Element_Type is private;
package Produce_Elements is
    package Elements_Vectors is Ada.Containers.Vectors (Positive, Element_Type);

    function Produce_Elements return Elements_Vectors.Vector;
end Produce_Elements;

--  No solution to use vectors produced by Produce_Elements.Produce_Elements in
--  Consume_Elements.Consume_Elements (appart from unchecked conversion).
```

There is a convoluted solution using generic formal packages, that is far from
ideal:

```ada
generic
    type Element_Type is private;
    with package Elements_Vectors is new Ada.Containers.Vectors (Positive, Element_Type);
procedure Consume_Elements (Elements : Elements_Vectors.Vector);

--  In another package/library

generic
    type Element_Type is private;
    with package Elements_Vectors is new Ada.Containers.Vectors (Positive, Element_Type);
function Produce_Elements return Elements_Vectors.Vector;

package Positive_Vectors is new Ada.Containers.Vectors (Positive, Positive);
function Produce_Positives is new Produce_Elements (Positive, Positive_Vectors);
procedure Consume_Positives is new Consume_Elements (Positive, Positive_Vectors);

Consume_Positives (Produce_Positives);
```

This solution is far from ideal, mainly because of its verbosity. It forces
instantiators to instantiate the generic themselves even in the (probable)
majority of cases where this modularity isn't needed. The consequence is that,
in practice, most generic code in Ada is not made to be modular.

Consider the solution with structural instantiations:

```ada
generic
    type Element_Type is private;
procedure Consume_Elements (Elements : Ada.Containers.Vectors [Positive, Element_Type].Vector);

--  In another package/library

generic
    type Element_Type is private;
function Produce_Elements return Ada.Containers.Vectors [Positive, Element_Type].Vector;

Consume_Elements [Positive] (Produce_Elements [Positive]);
```

Reference-level explanation
===========================

This is clearly not complete, we expect this draft to be completed during
prototyping.

### Syntax changes

Add the following syntax rule:

```
structural_generic_instantiation_reference ::=
    name [generic_actual_part]
```

And alter the `name` rule to include `structural_generic_instantiation_reference`

### Semantic changes

* Each `structural_generic_instantiation_reference` references a structural
  generic instantiation.

* This structural generic instantiation is semantically unique accross all
  units of the closure, so all references refer to the same instantiation.

* As soon as there exists one reference to a given structural instantiation,
  then it will be instantiated.

* Any generic can be instantiated, be it a package, procedure or function. A
  `structural_generic_instantiation_reference` will be syntactically valid in
  any context where a name is valid, and semantically valid in any context
  where a reference to the instantiated entity (subprogram or package) is
  valid.

Rationale and alternatives
==========================

The rationale is contained in the high level RFC on generics.

Drawbacks
=========

N/A

Prior art
=========

Most languages with generics also have by default structural instantiation of
them. In fact it is pretty much the default paradigm for generics in most
languages (C++, C#, Java, Rust, Haskell, OCaml, etc), which makes it difficult
to identify the feature with such a specific name, because it is usually just
called "generics".

TODO: Try to fill out this section nonetheless
