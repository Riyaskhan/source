# Records Work-in-Progress

Unlike the other records proposals, this is not a proposal in itself, but a work-in-progress designed to record consensus design
decisions for the records feature. Specification detail will be added as necessary to resolve questions.

The syntax for a record is proposed to be added as follows:

```antlr
class_declaration
    : attributes? class_modifiers? 'partial'? 'class' identifier type_parameter_list?
      parameter_list? type_parameter_constraints_clauses? class_body
    ;

struct_declaration
    : attributes? struct_modifiers? 'partial'? 'struct' identifier type_parameter_list?
      parameter_list? struct_interfaces? type_parameter_constraints_clauses? struct_body
    ;

class_body
    : '{' class_member_declarations? '}'
    | ';'
    ;

struct_body
    : '{' struct_members_declarations? '}'
    | ';'
    ;
```

The `attributes` non-terminal will also permit a new contextual attribute, `data`.

A class (struct) declared with a parameter list or `data` modifier is called a record class (record struct), either of which is a record type.

It is an error to declare a record type without both a parameter list and the `data` modifier.

## Members of a record type

In addition to the members declared in the class-body, a record type has the following additional members:

### Primary Constructor

A record type has a public constructor whose signature corresponds to the value parameters of the
type declaration. This is called the primary constructor for the type, and causes the implicitly
declared default constructor to be suppressed. It is an error to have a primary constructor and
a constructor with the same signature already present in the class.
At runtime the primary constructor 

1. executes the instance field initializers appearing in the class-body; and then
    invokes the base class constructor with no arguments.

1. initializes compiler-generated backing fields for the properties corresponding to the value parameters (if these properties are compiler-provided; see [Synthesized properties](#Synthesized Properties))


[ ] TODO: add base call syntax and specification about choosing base constructor through overload resolution

### Properties

For each record parameter of a record type declaration there is a corresponding public property member whose name and type are taken from the value parameter declaration. If no concrete (i.e. non-abstract) property with a get accessor and with this name and type is explicitly declared or inherited, it is produced by the compiler as follows:

For a record struct or a record class:

* A public get-only auto-property is created. Its value is initialized during construction with the value of the corresponding primary constructor parameter. Each "matching" inherited abstract property's get accessor is overridden.

### Equality members

Record types produce synthesized implementations for the following methods:

* `object.GetHashCode()` override, unless it is sealed or user provided
* `object.Equals(object)` override, unless it is sealed or user provided
* `T Equals(T)` method, where `T` is the current type

`T Equals(T)` is specified to perform value equality, comparing the property with same name as
each primary constructor parameter to the corresponding property of the other type.
`object.Equals` performs the equivalent of

```C#
override Equals(object o) => Equals(o as T);
```

## `with` expression

A `with` expression is a new expression using the following syntax.

```antlr
with_expression
    : switch_expression
    | switch_expression 'with' anonymous_object_initializer
```

A `with` expression allows for "non-destructive mutation", designed to
produce a copy of the receiver expression with modifications to properties
listed in the `anonymous_object_initializer`.

A valid `with` expression has a receiver with a non-void type. The receiver type must contain an accessible instance method called `With` with
the appropriate parameters and return type. It is an error if there are multiple non-override `With` methods. If there are multiple `With` overrides,
there must be a non-override `With` method, which is the target method. Otherwise, there must be exactly one `With` method.

On the right hand side of the `with` expression is an `anonymous_object_initializer` with a
sequence of assignments with a field or property of the receiver on the left-hand side of the
assignment, and an arbitrary expression on the right-hand side which is implicitly convertible to the type
of the left-hand side.

Given a target `With` method, the return type must be the type of the receiver expression type, or a base type thereof. For each parameter of
the `With` method, there must be an accessible corresponding instance field or readable property on the
receiver type with the same name and the same type. Each property or field in the right-hand side of the With
expression must also correspond to a parameter of the same name in the `With` method.

Given a valid `With` method, the evaluation of a `with` expression is equivalent to calling the `With` method with the expressions in the
`anonymous_object_initializer` substituted for the parameter of the same
name as the property on the left hand side. If there is no matching property
for a given parameter in the `anonymous_object_initializer`, the argument
is the evaluation of the field or property of the same name on the receiver.

The order of evaluation of side effects is as follows, with each expression
evaluated exactly once:

1. Receiver expression

2. Expressions in the `anonymous_object_initializer`, in lexical order

3. The evaluation of any properties matching the `With` method parameters,
in order of definition of the `With` method parameters.