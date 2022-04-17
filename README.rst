PEP: <REQUIRED: pep number>
Title: Adding a sealed qualifier to typing
Author: John Hagen <johnthagen@gmail.com>
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
in Python. Consider this example::

  from dataclasses import dataclass
  from typing import sealed

  @sealed
  class Node:
      ...

  class Expression(Node):
      ...

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
``Union[Name, Operation, Assignment, Print]``, and also safely treat e.g.
``Expression`` as ``Union[Name, Operation]``. So this will result in a type
checking error in the below snippet, because ``Name`` is not handled (and type
checker can give a useful error message)::

  def dump(node: Node) -> str:
      match node:
          case Assignment(target, value):
              return f"{target} = {dump(value)}"
          case Print(value):
              return f"print({dump(value)})"
          case Operation(left, op, right):
              return f"({dump(left)} {op} {dump(right)})"


Rationale
=========

[Describe why particular design decisions were made.]


Specification
=============

[Describe the syntax and semantics of any new language feature.]


Backwards Compatibility
=======================

[Describe potential impact and severity on pre-existing code.]


Security Implications
=====================

[How could a malicious user take advantage of this new feature?]


How to Teach This
=================

[How to teach users, new and experienced, how to apply the PEP to their work.]


Reference Implementation
========================

[Link to any existing implementation and details about its state, e.g. proof-of-concept.]


Rejected Ideas
==============

[Why certain ideas that were brought while discussing this PEP were not ultimately pursued.]


Open Issues
===========

[Any points that are still being decided/discussed.]


Footnotes
=========

.. [1]
   https://en.wikipedia.org/wiki/Pattern_matching

.. [2]
   https://en.wikipedia.org/wiki/Algebraic_data_type

.. [3]
   https://docs.python.org/3/library/typing.html

Copyright
=========

This document is placed in the public domain.