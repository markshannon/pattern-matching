
Abstract
========

This document outlines a possible design for pattern matching in Python.

It is not a proposal, merely an existance proof that is feasible to have a specification
for pattern matching that has well defined semantics, can implemented efficiently and
builds on existing concepts in Python, such as special methods.

The goal is to show that precise semantics are possible. PEP 634 demonstrates that precise syntax is possible.
Consequently the syntax below is a bit hand-wavy in places. Hopefully it is clear enough.
Clarification PRs are very welcome.

Syntax
======

At the top level, this roughly follows PEP 634. It is in the syntax for captures that it differs significantly.

Syntax::

  match_stmt: "match" subject_expr ':' NEWLINE INDENT case_block+ DEDENT
  subject_expr:
      | star_named_expression ',' star_named_expressions?
      | named_expression
  case_block: "case" pattern [guard] ':' ("\n" "case" pattern [guard] ':')* block
  guard: 'if' named_expression

Note that this differs from PEP 634 as multiple cases can be grouped together.
This is to avoid the need for an "or" pattern.
A match on pattern ``A`` and pattern ``B`` can be written as:

::

    match obj:
      case A:
      case B:
         body_for_A_or_B

Patterns
--------

::

  pattern:
    | capture_pattern
    | dict_pattern
    | sequence_pattern
    | class_pattern
    | expr

A ``capture_expression`` is defined as:

::

  capture_pattern:
      | CAPTURE_VARIABLE 
      | capture_pattern "." NAME
      | capture_pattern "[" expression "]"

The ``CAPTURE_VARIABLE`` token takes the form ``"$" + NAME``

::

  dict_pattern: '{' [dict_pattern_item (',' dict_pattern_item)* [','] '}' 
  dict_pattern_item: '**' pattern | NAME : pattern

::

  sequence_pattern: '[' sequence_pattern_item (',' sequence_pattern_item)* [','] ']'
  sequence_pattern_item: ['*'] pattern


Semantics
=========

Additions to the object model
-----------------------------

A ``__match_kind__()`` method will be added to ``object``.
It should be overridden by classes to describe the kind of match that class of objects supports.
It must return one of:
``MATCH_VALUE``
``MATCH_SEQUENCE``
``MATCH_MAPPING``
``MATCH_CLASS``

.. note::

    We are sidestepping the issue of whether ``True`` should match ``1`` or vice versa.
    The issues involved are exactly the same as for PEP 634.

Classes that return ``MATCH_CLASS`` need to implement two additional special methods,
``__unpack_attributes__`` and ``__unpack_values__``.
``__unpack_attributes__`` must return a tuple of strings indicating the names of attributes that instances of the class that are considered public
for unpacking. 
``__unpack_values__`` must return a sequence of the same length as the tuple returned from ``__unpack_attributes__``
which contains the values corresponding to the attribute names.

:: note:

    ``__unpack_attributes__`` and ``__unpack_values__`` will be automatically
    generated for dataclasses and named tuples.

Desugaring
----------

To clarify the semantics of the matching operations, all patterns are desugared as follows.
Each ``capture_pattern`` in a pattern is replaced with the special pattern ``$n`` where `n` is its position.
Any variables in the guard whos name matches a ``capture_pattern`` are replaced with the corresponding ``$n`` variable.
Then an assignment is inserted after the guard, but before the body that assigns the pattern variable to normal variables.

For example:

::
    case [$a, $b.x, 1] if $a is None:
        body

Would be desugared to:

::

    case [$1, $2, 1] if $1 is None:
        a, b.x = $1, $2
        body

All special variables ``$n`` have scope limited to the case being matched are invisible to introspection.
A few other special variables are needed ``$cls``, ``$values`` and ``$attrs``. These values scope is the entire match statement.

:: note:
  Each match and pattern have their own set of variables. Nest matches do not use the same variables.
  It is expected that they would be implemented in CPython by keeping them on the stack.


Matching
--------

The object being matched will be compared against each pattern in turn, until a match is found.
Once a match has been found, and only once a match has been found will values be assigned.

Before after patterns are examined, ``__match_kind__()`` is called exactly once.
The matching process procedes as follows:

1. Fail to match if the pattern does not apply to the kind resulting from ``__match_kind__()``

  * ``capture_pattern`` applies to all kinds
  * ``expr`` applies to all kinds
  * ``dict_pattern`` applies to ``MATCH_MAPPING``
  * ``sequence_pattern`` applies to ``MATCH_SEQUENCE``
  * ``class_pattern`` applies to ``MATCH_CLASS``

2. Match against the pattern:

  * For ``capture_pattern``, store the object in the specified variable. It always matches.
  * For ``expr`` perform an equality test with the object. If equal, then the pattern matches.
  * For ``sequence_pattern``:

    * If this is the first ``sequence_pattern``: iterate over the object forming a list. Store that list in ``$values``.
    * If the length of ``$values`` is not within the range of lengths the proceed to the next case
    * Check each sub-pattern for a match in depth-first, left-to-right order. If any fail, then the match fails.

  * For ``dict_pattern``:

    * Evaluate ``m = bool(all(key in obj for key in keys))`` where ``keys`` is the list of keys in the pattern. If ``m`` is false then the match fails.
    * Check each sub-pattern for a match in depth-first, left-to-right order. If any fail, then the match fails.

  * For ``class_pattern``:

    * If this is the first ``class_pattern``:
    
      * Store ``type(obj)`` in ``$cls``
      * Call ``__unpack_attributes__`` and store it to ``$attrs``.
      * Call ``__unpack_values__`` and store it in ``$values``.

    * If ``set(attributes) <= set($attrs)`` is false, where ``attributes`` is the set of attributes specified in the pattern, then the match fails.
    * Check each sub-pattern for a match in depth-first, left-to-right order. If any fail, then the match fails.


4. If the pattern matches, then check the guard:

  * If the guard evaluates false, then proceed to the next case.

Implementation
==============

Implementations should obey the "as if" rule. That is, they should behave as if the above sequence of operations occurs.
Implementations are thus free to reorder operations that have no observable side effects.

They are also free to consider that loading a global or class-local variable have no side-effects,
even though in some obscure circumstances they might have.


Example
-------

Consider the match statement

::

    match x:
        case [$a]:
            # Match a single element sequence
            A
        case [$a, $b] if $a > 3:
            # Match a two element sequence if the first element is greater than 3
            B
        case [$x, a]:
            # Match a two element sequence if the second element is equal to a.
            C
        case []:
            # Match an empty sequence
            D
        case print("testing"):
            # Matches None, and prints "testing" as a side effect.
            # This is obviously silly code, but it is allowed for consistency.
            # `if x == print("testing"):` is legal.
            E
        case $dont_care:
            # Match anything else
            F

Pattern matching we will need several new instructions.
For this example, we introduce two new instructions.

* ``MATCH_KIND`` which pushes ``type(tos).__match_kind__()`` where ``tos`` is the value on top of the stack.
* ``PEEK n`` which pushes a copy of the nth value on the stack.

This example can be compiled reasonably efficiently, and without any further change to the interpreter.

Assuming that all variables are locals, a possible bytecode sequence is:

::

    LOAD_FAST x
    MATCH_KIND
    LOAD_CONST `MATCH_SEQUENCE`
    COMPARE_OP ==
    POP_JUMP_IF_FALSE case_E_or_F
    DUP_TOP
    GET_ITER
    FOR_ITER case_D
    ROT_TWO  # Swap first item and iterator
    FOR_ITER case_A
    ROT_TWO  # Swap second item and iterator
    FOR_ITER case_B_or_C_or_E_or_F
    ROT_TWO
    POP_TOP
    ROT_TWO
    POP_TOP
  exhaust_iter:
    FOR_ITER case_E_or_F
    POP_TOP
    JUMP exhaust_iter
  case_A:
    STORE_FAST a
    POP_TOP
    # Code for A
    JUMP end
  case_B_or_C_or_E:
    PEEK 2
    LOAD_CONST 3
    COMPARE_OP >
    POP_JUMP_IF_FALSE case_C_or_E_or_F
    STORE_FAST b
    STORE_FAST a
    POP_TOP
    # Code for B
    JUMP end
  case_C_or_E_or_F:
    LOAD_FAST a
    COMPARE_OP ==
    POP_JUMP_IF_TRUE case_C
    POP_TOP
    JUMP case_E_or_F
  case_C:
    STORE_FAST x
    POP_TOP
    # Code for C
    JUMP end
  case_D:
    POP_TOP
    # Code for D
    JUMP end
  case_E_or_F:
    DUP_TOP
    LOAD_GLOBAL print
    LOAD_CONST 'testing'
    CALL_FUNCTION            1
    COMPARE_OP ==
    POP_JUMP_IF_FALSE case_F
    POP_TOP
    # Code for E
    JUMP end
  case_F:
    STORE_FAST dont_care
    # Code for F
  end:

The above code obeys the semantics, as it checks the kind of ``x`` and completely iterates over it if it is a sequence.
It is also efficient as we can elide the length checks, by branching on termination of the iteration
to the relevant case(s).

