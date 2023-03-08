- Feature Name: Inference of dependent types in generic instantiations
- Start Date: 2023-03-03
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This RFC proposes to allow inference of types in generic specification, when
there is a way to deduce it from other generic parameters.

Motivation
==========

This RFC is part of the bigger high-level RFC about improving generic
instantiations ([here](../meta/rfc-improved-generic-instantiations.md)), so the
need arises from that context, and it's useful to go read the high level RFC to
understand the bigger picture.

However, even in the reduced context of explicit instantiations, it's easy to
understand the value of this feature with a few simple examples:

```ada
type Integer_Access is access all Integer;

procedure Free is new Unchecked_Deallocation (Name => Integer_Access);
```

or

```ada
type Arr is array (Positive range <>) of Integer;

package Int_Array_Sort
is new Ada.Containers.Generic_Array_Sort (Array_Type => Arr);
```

or

```ada
```

Guide-level explanation
=======================

In every case where a generic formal references other types, those types can
be other formal types. In those case, the user of the generic needs to pass
every type separately.


```ada
generic
    type Element_Type is private;
    type Index_Type is (<>);
    type Array_Type is array (Index_Type range <>) of Element_Type;
package Array_Operations is
    ...
end Array_Operations;

...

type Int_Array is array (Positive range <>) of Integer;

package Int_Array_Operations is new Array_Operations
  (Element_Type => Integer,
   Index_Type   => Positive,
   Array_Type   => Int_Array);
```

This feature allows the programmer to not have to pass type parameters that
could be deduced from other type parameters. In the example above, the language
can automatically deduce the index and element type from the array type that is
passed in:

```ada
package Int_Array_Operations is new Array_Operations (Array_Type => Int_Array);
```

* Generic formal array types (see the first example)

* Generic formal access types, where the accessed type can be deduced from the
  access type.

```ada
type Integer_Access is access all Integer;

procedure Free is new Unchecked_Deallocation (Name => Integer_Access);
```

* Generic formal subprograms, where any type can be deduced from the type of
  the formals of the subprogram

```ada
generic
   type Element_Type is private;
   type Array_Type is array (Positive range <>) of Element_Type;
   type Key_Type is private;
   with function Key (E : Element_Type) return Key_Type;
   with function "<" (L, R : Key_Type) return Boolean is (<>);
function Min_By (Self : Array_Type) return Element_Type;

-- usage:

type Person is record
   Name : Unbounded_String;
   Age  : Natural;
end record;

function Age_Of (P : Person) return Natural is (P.Age);

type Person_Array is array (Positive range <>) of Person;

function Min_By_Age is new Min_By
  (Array_Type => Person_Array, Key => Age_Of);
-- Element_Type inferred from Person_Array, Key_Type inferred from Age_Of

DB      : Person_Array := ...;
Younger : Person := Min_By_Age (DB);
```

> **Note**
> We decided to not include generic formal packages, because the implementer
> already has the option to not require the user to pass in the dependent
> types, via the `<>` syntax:
>
> ```ada
> generic
>    with package P is new Q (<>);
> ```

Reference-level explanation
===========================

To be completed

### Syntax changes

No syntax changes planned

### Semantic changes

When in the presence of a dependent type, as defined above, in a generic
formal, the instantiator of the generic can omit this type, either passing `<>`
explicitly, or just omitting it from the instantiation completely.

The compiler will deduce it from other parameters.

Rationale and alternatives
==========================

The rationale is contained in the high level RFC on generics.

As far as alternatives go, we could imagine a world where developers don't even
have to specify the dependent formals:

```ada
generic
   type Array_Type is array (<>) of <>;
package Array_Operations is
   subtype Index_Type is Array_Type'Index_Type;
   subtype Element_Type is Array_Type'Element_Type;
end Array_Operations;

...

type Int_Array is array (Positive range <>) of Integer;

package Int_Array_Operations is new Array_Operations
  (Array_Type   => Int_Array);
```

However, the current alternative has the advantage of being backwards
compatible with existing generic declarations.


Drawbacks
=========

N/A

Prior art
=========

This is very specific to Ada's generic formals system, but we could consider
that they way generic formal packages' own params can be deduced when
instantiating the generic, is pretty similar to what we propose here, so that
this is the extension of an already existing mechanism.
