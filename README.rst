PEP: <REQUIRED: pep number>
Title: Adding a sealed qualifier to typing
Author: John Hagen <johnthagen@gmail.com>, David Hagen <david@drhagen.com>
Sponsor: Jelle Zijlstra
PEP-Delegate: <PEP delegate's real name>
Discussions-To: typing-sig@python.org
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 17-Apr-2022
Python-Version: 3.11
Post-History:
Resolution: <url>


Abstract
========

This PEP proposes a ``@sealed`` decorator be added to the ``typing`` module to
support creating algebraic data types (ADTs) which type checkers can
exhaustively pattern match against.


Motivation
==========

Quite often it is desirable to apply exhaustiveness to a set of classes without
defining ad-hoc union types, which is itself fragile if a class is missing in
the union definition. A design pattern where a group of record-like classes is
combined into a union is popular in other languages that support pattern
matching [1]_ and is known under a name of algebraic data types [2]_.

We propose to add a special decorator class ``@sealed`` to the ``typing``
module [3]_, that will have no effect at runtime, but will indicate to static
type checkers that all subclasses (direct and indirect) of this class should
be defined in the same module as the base class.

The idea is that since all subclasses are known, the type checker can treat
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

With such definition, a type checker can safely treat ``Node`` as
``Union[Expression, Statement]``, and also safely treat e.g.
``Expression`` as ``Union[Name, Operation]`` and ``Statement`` as
``Union[Assignment, Print]``. So this will result in a type checking error in
the below snippet, because ``Name`` is not handled (and type checker can give a
useful error message).

.. code-block:: python

    def dump(node: Node) -> str:
        match node:
            case Assignment(target, value):
                return f"{target} = {dump(value)}"
            case Print(value):
                return f"print({dump(value)})"
            case Operation(left, op, right):
                return f"({dump(left)} {op} {dump(right)})"

Note: This section was largely derived from PEP 622 [11]_.


Rationale
=========

Kotlin [4]_, Scala 2 [5]_, and Java 17 [6]_ all support a ``sealed`` keyword
that is used to create algebraic data types. By using the same terminology,
the ``@sealed`` decorator will be familiar to developers familiar with those
languages.


Specification
=============

[Describe the syntax and semantics of any new language feature.]


How to Teach This
=================

[How to teach users, new and experienced, how to apply the PEP to their work.]


Reference Implementation
========================

[Link to any existing implementation and details about its state, e.g. proof-of-concept.]


Rejected Ideas
==============

Generalize ``Enum``
-------------------

Rust [7]_, Scala 3 [8]_, and Swift [9]_ support algebraic data types using a
generalized ``enum`` mechanism.

.. code-block:: rust

    enum Message {
        Quit,
        Move { x: i32, y: i32 },
        Write(String),
        ChangeColor(i32, i32, i32),
    }

One could imagine a generalization of the Python ``Enum`` [10]_ to support
variants of different shapes. But given that the Python ``Enum`` is more or
less a normal class, with some magic internals, this would be a much more
invasive change.

.. code-block:: python

    from dataclasses import dataclass
    from enum import Enum

    class Message(Enum):
        @dataclass
        class Quit:
            ...

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
this feature. Once disadvantage is that requires that all subclasses to be
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


Open Issues
===========

Must subclasses by defined in the same module?
----------------------------------------------

Kotlin, Scala, and Java require subtypes of ``sealed`` classes be defined in
the same file as the ``sealed`` class. This assures that all subtypes are known
to the type checker.

Should Python specify the same restriction?


Footnotes
=========

.. [1]
   https://en.wikipedia.org/wiki/Pattern_matching

.. [2]
   https://en.wikipedia.org/wiki/Algebraic_data_type

.. [3]
   https://docs.python.org/3/library/typing.html

.. [4]
   https://kotlinlang.org/docs/sealed-classes.html

.. [5]
   https://docs.scala-lang.org/tour/pattern-matching.html

.. [6]
   https://openjdk.java.net/jeps/409

.. [7]
   https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html

.. [8]
   https://docs.scala-lang.org/scala3/reference/enums/adts.html

.. [9]
   https://docs.swift.org/swift-book/LanguageGuide/Enumerations.html

.. [10]
   https://docs.python.org/3/library/enum.html

.. [11]
   https://peps.python.org/pep-0622/#sealed-classes-as-algebraic-data-types


Copyright
=========

This document is placed in the public domain.
