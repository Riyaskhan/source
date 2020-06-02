
# Records

This proposal tracks the specification for the C# 9 records feature, as agreed to by the C#
language design team.

The syntax for a record is as follows:

```antlr
record_declaration
    : attributes? class_modifier* 'partial'? 'record' identifier type_parameter_list?
      parameter_list? record_base? type_parameter_constraints_clause* record_body
    ;

record_base
    : ':' class_type argument_list?
    | ':' interface_type_list
    | ':' class_type argument_list? interface_type_list
    ;

record_body
    : '{' class_member_declaration* '}'
    | ';'
    ;
```

Record types are reference types, similar to a class declaration. It is an error for a record to provide
a `record_base` `argument_list` if the `record_declaration` does not contain a `parameter_list`.

## Members of a record type

In addition to the members declared in the record body, a record type has additional synthesized members.
Members are synthesized unless an accessible concrete (non-abstract) member with a "matching" signature is 
either inherited or declared in the record body. Two members are considered matching if they have the same
signature or would be considered "hiding" in an inheritance scenario.

The synthesized members are as follows:

### Equality members

Record types produce synthesized implementations for the following methods, where `T` is the
containing type:

* `object.GetHashCode()` override
* `object.Equals(object)` override
* `T Equals(T)` method, where `T` is the current type
* `Type EqualityContract` get-only property

If either `object.GetHashCode()` or `object.Equals(object)` are sealed, an error is produced.

`EqualityContract` is a virtual instance property which returns `typeof(T)`. If the base type
defines an `EqualityContract` it is overridden in the derived record. If the base `EqualityContract`
is sealed or non-virtual, an error is produced.

`T Equals(T)` is specified to perform value equality such that `Equals` is true if and only if
all accessible instance fields in the receiver are equal to the fields of the parameter
and `this.EqualityContract` equals `other.EqualityContract`.

`object.Equals` performs the equivalent of

```C#
override Equals(object o) => Equals(o as T);
```

### Copy and Clone members

A record type contains two synthesized copying members:

* A protected constructor taking a single argument of the record type.
* A public parameterless virtual instance "clone" method with a compiler-reserved name

The protected constructor is referred to as the "copy constructor" and the synthesized
body copies the values of all accessible instance fields in the input type to the corresponding
fields of `this`.

The "clone" method returns the result of a call to a constructor with the same signature as the
copy constructor. The return type of the clone method is the containing type, unless a virtual
clone method is present in the base class. In that case, the return type is the current containing
type if the "covariant returns" feature is supported and the override return type otherwise. The
synthesized clone method is an override of the base type clone method if one exists. An error is
produced if the base type clone method is sealed.

## Positional record members

In addition to the above members, records with a parameter list ("positional records") synthesize
additional members with the same conditions as the members above.

### Primary Constructor

A record type has a public constructor whose signature corresponds to the value parameters of the
type declaration. This is called the primary constructor for the type, and causes the implicitly
declared default class constructor, if present, to be suppressed. It is an error to have a primary
constructor and a constructor with the same signature already present in the class.

At runtime the primary constructor

1. executes the instance initializers appearing in the class-body

1. invokes the base class constructor with the arguments provided in the `record_base` clause, if present


### Properties

For each record parameter of a record type declaration there is a corresponding public property
member whose name and type are taken from the value parameter declaration.

For a record:

* A public `get` and `init` auto-property is created (see separate `init` accessor specification).
  Each "matching" inherited abstract accessor is overridden. The auto-property is initialized to
  the value of the corresponding primary constructor parameter.

### Deconstruct

A positional record synthesizes a public void-returning method called Deconstruct with an out
parameter declaration for each parameter of the primary constructor declaration. Each parameter
of the Deconstruct method has the same type as the corresponding parameter of the primary
constructor declaration. The body of the method assigns each parameter of the Deconstruct method
to the value from an instance member access to a member of the same name.

## `with` expression

A `with` expression is a new expression using the following syntax.

```antlr
with_expression
    : switch_expression
    | switch_expression 'with' '{' member_initializer_list? '}'
    ;

member_initializer_list
    : member_initializer (',' member_initializer)*
    ;

member_initializer
    : identifier '=' expression
    ;
```

A `with` expression allows for "non-destructive mutation", designed to
produce a copy of the receiver expression with modifications in assignments
in the `member_initializer_list`.

A valid `with` expression has a receiver with a non-void type. The receiver type must contain an
accessible synthesized record "clone" method.

On the right hand side of the `with` expression is a `member_initializer_list` with a sequence
of assignments to *identifier*, which must be an accessible instance field or property of the return
type of the `Clone()` method.

Each `member_initializer` is processed the same way as an assignment to a field or property
access of the return value of the record clone method. The clone method is executed only once
and the assignments are processed in lexical order.
