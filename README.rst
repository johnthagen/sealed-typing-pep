PEP: <REQUIRED: pep number>
Title: Sealed Decorator for Static Typing
Author: John Hagen <johnthagen@gmail.com>, David Hagen <david@drhagen.com>
Sponsor:
PEP-Delegate: <PEP delegate's real name>
Discussions-To: https://discuss.python.org/t/draft-pep-sealed-decorator-for-static-typing/49206
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 22-Mar-2024
Python-Version: 3.13
Post-History:
Resolution: <url>


Abstract
========

This PEP proposes a ``@sealed`` decorator be added to the ``typing`` module to
support creating versatile algebraic data types (ADTs) which type checkers can
exhaustively pattern match against.


Motivation
==========

Quite often it is desirable to apply exhaustiveness to a set of classes without
defining ad-hoc union types, which is itself fragile if a class is missing in
the union definition. A design pattern where a group of record-like classes is
combined into a union is popular in other languages that support pattern
matching [1]_ and is known as a nominal sum type, a key instantiation of
algebraic data types [2]_.

We propose adding a special decorator class ``@sealed`` to the ``typing``
module [3]_, that will have no effect at runtime, but will indicate to static
type checkers that all direct subclasses of this class should be defined in the
same module as the base class.

The idea is that, since all subclasses are known, the type checker can treat
the sealed base class as a union of all its subclasses. Together with
dataclasses this allows a clean and safe support of algebraic data types
in Python. Consider this example,

.. code-block:: python

    from dataclasses import dataclass
    from typing import sealed

    @sealed
    class Node:
        ...

    @sealed
    class Expression(Node):
        ...

    @sealed
    class Statement(Node):
        ...

    @dataclass
    class Name(Expression):
        name: str

    @dataclass
    class Operation(Expression):
        left: Expression
        op: str
        right: Expression

    @dataclass
    class Assignment(Statement):
        target: str
        value: Expression

    @dataclass
    class Print(Statement):
        value: Expression

With such a definition, a type checker can safely treat ``Node`` as
``Union[Expression, Statement]``, and also safely treat ``Expression`` as
``Union[Name, Operation]`` and ``Statement`` as ``Union[Assignment, Print]``.
With these declarations, a type checking error will occur in the below snippet,
because ``Name`` is not handled (and the type checker can give a useful error
message).

.. code-block:: python

    def dump(node: Node) -> str:
        match node:
            case Assignment(target, value):
                return f"{target} = {dump(value)}"
            case Print(value):
                return f"print({dump(value)})"
            case Operation(left, op, right):
                return f"({dump(left)} {op} {dump(right)})"

Note: This section was largely derived from PEP 622 [4]_.


Rationale
=========

Kotlin [5]_, Scala 2 [6]_, and Java 17 [7]_ all support a ``sealed`` keyword
that is used to declare algebraic data types. By using the same terminology,
the ``@sealed`` decorator will be familiar to developers familiar with those
languages.


Specification
=============

The ``typing.sealed`` decorator can be applied to the declaration of any class.
This decoration indicates to type checkers that all immediate subclasses of the
decorated class are defined in the current file.

The exhaustiveness checking features of type checkers should assume that there
are no subclasses outside the current file, treating the decorated class as a
``Union`` of all its same-file subclasses.

Type checkers should raise an error if a sealed class is inherited in a file
different from where the sealed class is declared.

A sealed class is automatically declared to be abstract. Whatever actions a
type checker normally takes with abstract classes should be taken with sealed
classes as well. What exactly these behaviors are (e.g. disallowing
instantiation) is outside the scope of this PEP.

Similar to the ``typing.final`` decorator [8]_, the only runtime behavior of
this decorator is to set the ``__sealed__`` attribute of class to ``True`` so
that the sealed property of the class can be introspected. There is no runtime
enforcement of sealed class inheritance.


Reference Implementation
========================

[Link to any existing implementation and details about its state, e.g.
proof-of-concept.]


Rejected Ideas
==============

``Union`` of independent variants
---------------------------------

Some of the behavior of ``sealed`` can be emulated with ``Union`` today.

.. code-block:: python

    class Leaf: ...
    class Branch: ...

    Node = Leaf | Branch

The main problem with this is that the ADT loses all the features of
inheritance, which is rather featureful in Python, to put it mildly. There can
be no abstract methods, private methods to be reused by the subclasses, public
methods to be exposed on all subclasses, class methods of any kind,
``__init_subclass__``, etc. Even if a specific method is implemented on each
subclass, then rename, jump-to-definition, find-usage, and other IDE features
are difficult to make work reliably.

Adding a base class in addition to the union type alleviates some of these
issues:

.. code-block:: python

    class BaseNode: ...

    class Leaf(BaseNode): ...
    class Branch(BaseNode): ...

    Node = Leaf | Branch

Despite being possible today, this is quite unergonomic. The base class and the
union type are conceptually the same thing, but have to be defined as two
separate objects. If this became standard, it seems Python would be first
language to separate the definition of an ADT into two different objects.

This duplication causes a serious don't-repeat-yourself problem. A new subclass
must be added to both the base class and the union type. Failure to do so will
not result in an immediate error but in inconsistent behavior between the two
representations.

The base class is not merely passive, either. There are a number of operations
that will only work when using the base class instead of the union type and
vice verse. For example, matching only works on the base class, not the union
type:

.. code-block:: python

    maybe_node: Node | None = ...  # must be Node to enforce exhaustiveness

    match maybe_node:
        case Node():  # TypeError: called match pattern must be a type
            ...
        case None:
            ...

    match maybe_node:
        case BaseNode():  # no error
            ...
        case None:
            ...

Having to remember whether to use the base class or the union type in each
situation is particularly unfriendly to the user of a sealed class.

Generalize ``Enum``
-------------------

Rust [9]_, Scala 3 [10]_, and Swift [11]_ support algebraic data types using a
generalized ``enum`` mechanism.

.. code-block:: rust

    enum Message {
        Quit,
        Move { x: i32, y: i32 },
        Write(String),
        ChangeColor(i32, i32, i32),
    }

One could imagine a generalization of the Python ``Enum`` [12]_ to support
variants of different shapes. Valueless variants could use ``enum.auto`` to
keep themselves terse.

.. code-block:: python

    from dataclasses import dataclass
    from enum import auto, Enum

    class Message(Enum):
        Quit = auto()

        @dataclass
        class Move:
            x: int
            y: int

        @dataclass
        class Write:
            message: str

        @dataclass
        class ChangeColor:
            r: int
            g: int
            b: int

This solution allows attaching methods directly to the base ADT type,
something a ``Union`` type lacks, but does not support the full
power of inheritance that ``@sealed`` would provide.

This would be a substantial addition to the implementation and
semantics of ``Enum``.

Explicitly list subclasses
--------------------------

Java requires that subclasses be explicitly listed with the base class.

.. code-block:: java

    public sealed interface Node
        permits Leaf, Branch {}

    public final class Leaf {}
    public final class Branch {}

The advantage of this requirement is that subclasses can be defined anywhere,
not just in the same file, eliminating the somewhat weird file dependence of
this feature. The disadvantage is that it requires all subclasses to be
written twice: once when defined and once in the enumerated list on the base
class.

There is also an inherent circular reference when explicitly enumerating the
subclasses. The subclass refers to the base class in order to inherit from it,
and the base class refers to the subclasses in order to enumerate them. In
statically typed languages, these kinds of circular references in the types can
be managed, but in Python, it is much harder.

For example, this ``Sealed`` base class that behaves like ``Generic``:

.. code-block:: python

    from typing import Sealed

    class Node(Sealed[Leaf, Branch]): ...

    class Leaf(Node): ...
    class Branch(Node): ...

This cannot work because ``Leaf`` must be defined before ``Node`` and ``Node``
must be defined before ``Leaf``. This is a not an annotation, so lazy
annotations cannot save it. Perhaps, the subclasses in the enumerated list could
be strings, but that severely hurts the ergonomics of this feature.

If the enumerated list was in an annotation, it could be made to work, but there
is no natural place for the annotation to live. Here is one possibility:

.. code-block:: python

    class Node:
        __sealed__: Leaf | Branch

    class Leaf(Node): ...
    class Branch(Node): ...

Footnotes
=========

.. [1]
   https://en.wikipedia.org/wiki/Pattern_matching

.. [2]
   https://en.wikipedia.org/wiki/Algebraic_data_type

.. [3]
   https://docs.python.org/3/library/typing.html

.. [4]
   https://peps.python.org/pep-0622/#sealed-classes-as-algebraic-data-types

.. [5]
   https://kotlinlang.org/docs/sealed-classes.html

.. [6]
   https://docs.scala-lang.org/tour/pattern-matching.html

.. [7]
   https://openjdk.java.net/jeps/409

.. [8]
   https://peps.python.org/pep-0591/

.. [9]
   https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html

.. [10]
   https://docs.scala-lang.org/scala3/reference/enums/adts.html

.. [11]
   https://docs.swift.org/swift-book/LanguageGuide/Enumerations.html

.. [12]
   https://docs.python.org/3/library/enum.html



Copyright
=========

This document is placed in the public domain.
