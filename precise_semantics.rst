PEP: XXX
Title: Precise Semantics for Pattern Matching
Version: $Revision$
Last-Modified: $Date$
Author: Mark Shannon <mark@hotpy.org>
BDFL-Delegate:
Discussions-To: Python-Dev <python-dev@python.org>
Status: Draft
Type: Informational
Content-Type: text/x-rst
Created: 5-Nov-2020
Post-History:
Resolution:


Abstract
========

This document outlines a well defined semantics for pattern matching in Python.

It is not meant as a full proposal for pattern matching for Python, as the syntax is designed for clarity, 
not elegance or ease of use.
However, it is a necessary proposal, as the other proposals for pattern matching are imprecise, fragile and not implemented efficiently.
This propsal is precise, robust, and describes how it can be implemented efficiently.

This proposal shows that it is feasible to have a specification for pattern matching that has well defined semantics, 
can be implemented efficiently and builds on existing concepts in Python, such as special methods.

We believe that a pattern matching implementation for Python must have the following three properties:

* The semantics of pattern matching must be precisely defined.
* It must be implemented efficiently. That is, it should perform at least as well as a well written series of ``if``, ``elif`` statements.
* It must be robust in the face of changes to the environment. It should be defined in terms of the behaviour of objects, not global state.

Syntax
======

This syntax is probably not what you would want as langauge level syntax.
It is chosen to be unambiguous and to allow the semantics to be clear.
It is not going to win any beauty contests.

This syntax is suitable as a desugaring target for more aesthetically pleasing syntaxes,
to help clarify the semantics of other proposals.

Match statement
---------------

::

  match_stmt: "match" subject_expr ':' NEWLINE INDENT case_block+ DEDENT
  subject_expr:
      | star_named_expression ',' star_named_expressions?
      | named_expression
  case_spec: "case" pattern ['if' pattern_guard] [ '->' star_targets ] ':'
  case_block: case_spec (NEWLINE case_spec)* INDENT block DEDENT

Note that this differs from PEP 634 as multiple cases can be grouped together.
This is to avoid the need for an "or" pattern, leaving ``|`` and ``or`` to have their normal meaning.
A match on pattern ``A`` *or* pattern ``B`` can be written as:

::

  match obj:
      case A:
      case B:
          body_for_A_or_B

Patterns
--------

::

  pattern:
    | CAPTURE_VARIABLE
    | mapping_pattern
    | sequence_pattern
    | deconstruct_pattern
    | expression

A ``pattern_guard`` has the same syntax as a ``named_expression``, but allows a CAPTURE_VARIABLE as an additional atom.

The ``CAPTURE_VARIABLE`` token is a dollar followed by a decimal number. E.g. ``$2``

This design requires that capture variables are lexically distinct from normal variables.
The "$" prefix is arbitrary; any non-identifier character that is not already used in Python would work.

::

  mapping_pattern: '{' mapping_pattern_item (',' mapping_pattern_item)* [','] '}'
  mapping_pattern_item: '**' pattern | NAME ':' pattern

::

  sequence_pattern: '[' item_pattern (',' item_pattern)* [','] ']'
  item_pattern: ['*'] pattern

::

  deconstruct_pattern:
    | class_name '(' named_pattern (',' named_pattern)* [','] ')'
    | class_name '(' [ pattern (',' pattern)* [','] ] ')'
  class_name: NAME ('.' NAME)*
  named_pattern: NAME '=' pattern

Desugaring
----------

The above syntax is designed to make the semantics clear, by making the test to match a pattern distinct from
the assignment to variables.

Any syntactic element in a pattern that is the same as an expression is evaluated exactly as that expression.
Any values captured are clearly marked as a ``CAPTURE_VARIABLE``. 

Other proposals for pattern matching either prohibit certain syntaxes within patterns or use a "sigil" (a special character)
to either mark binding variables, or to mark non-binding variables.
Regardless of their exact syntaxes, the syntax of other proposals can be desugared to this syntax.

Example desugarings
'''''''''''''''''''

PEP 634:

::

    case [a,b]:
        ...
    case BinOp(l, "+", r):
        ...

becomes

::

    case [$1, $2] -> a, b:
        ...
    case BinOp($1, "+", $2) -> l, r:
        ...

PEP 642:

::

    case [a, ?CONST]:
        ...

becomes

::

    case [$1, CONST] -> a:
        ...

Further desugaring
''''''''''''''''''

Further desugaring steps are needed to help define the semantics.

First, all case blocks containing multiple case specs are simplified by splitting into
mutiple case blocks, duplicating the body.

For example:

::

      case A:
      case B:
          body

becomes

::


      case A:
          body
      case B:
          body
          
Second, the capture variables are renumbered so that they start from zero.

Semantics
=========

Additions to the object model
-----------------------------

A ``__match_kind__()`` method will be added to ``object``.
It should be overridden by classes to describe the kind of match that class of objects supports.
It must return one of:

::

  MATCH_VALUE
  MATCH_SEQUENCE
  MATCH_MAPPING
  MATCH_CLASS

.. note::

    For the purposes of this semantics, it does not matter what the actual values are.
    We will refer to them by name only. In practice, they will most likely be small integers.

``object.__match_kind__()`` will return ``MATCH_VALUE``.

Classes that return ``MATCH_CLASS`` need to implement two additional special methods:

* ``__attributes__()``: must return a tuple of strings indicating the names of attributes that are to be considered for matching.
* ``__deconstruct__()``: must return a sequence of the same length as the tuple returned from ``__attributes__`` which contains the values corresponding to the attribute names.

.. note::

    ``__attributes__`` and ``__deconstruct__`` will be automatically
    generated for dataclasses and named tuples.

The pattern matching implementation is *not* required to check that the values returned by ``__attributes__`` or ``__deconstruct__`` are as specified.
If the result of ``__attributes__()`` or ``__deconstruct__`` is not as specified, then
the implementation may raise any exception, or match the wrong pattern.

Matching
--------

Match scope
'''''''''''

Each match statement introduces its own scope with three variables,
``$kind``, ``$cls``, ``$attrs`` and ``$values``.
Nested match statements introduce their own scope and cannot see the scope of enclosing of match statments.
These match scopes do not change normal variable scopes in any way.

Case scope
''''''''''

Each case has, after desugaring, its own scope with the variables ``$0``, ``$1``, etc.
Cases can see the scope of the directly enclosing match statement, but no further enclosing cases or match statments.
These match scopes do not change normal variable scopes in any way.


Matching process
''''''''''''''''

The object being matched will be compared against each pattern in turn, until a match is found.
Once a match has been found, and only once a match has been found, will values be assigned.

Before any patterns are examined, ``__match_kind__()`` is called and the result stored in ``$kind``.
The matching process procedes as follows, for each pattern in the order specified until a match is found:

1. Fail to match if the pattern does not apply to ``$kind``.

   * ``capture_pattern`` applies to all kinds
   * ``expression`` applies to all kinds
   * ``mapping_pattern`` applies to ``MATCH_MAPPING``
   * ``sequence_pattern`` applies to ``MATCH_SEQUENCE``
   * ``deconstruct_pattern`` applies to ``MATCH_CLASS``

2. Match against the pattern:

   * A ``capture_pattern`` is always a match.
   * For an ``expression`` perform an equality test with the object. If equal, then the pattern matches.
    
     .. note::

        We are sidestepping the issue of whether ``True`` should match ``1`` or not.
        The issues involved are exactly the same as for PEP 634.
        Whatever is chosen, it should be symmetric. That is, if ``True`` does not match ``1``, then ``1`` must not match ``True``.

   * For a ``sequence_pattern``:

     a. If this is the first ``sequence_pattern``: iterate over the object forming a list. Store that list in ``$values``.
    
        .. note::

          Implementations are allowed to treat iteration steps are side-effect free, but not the process of creating an iterator.
          Consequently, objects that match sequences should not rely on iterators being exhausted, but can rely on an iterator being created.
     
     b. If the length of ``$values`` is not within the range of lengths for the pattern, then proceed to the next case
     c. Check each sub-pattern for a match in depth-first, left-to-right order. If any sub-pattern  match fails, then the whole match fails.

   * For a ``mapping_pattern``:

     a. Evaluate ``m = bool(all(key in obj for key in keys))`` where ``keys`` is the list of keys in the pattern. If ``m`` is false then the match fails.
     b. Check each sub-pattern for a match in depth-first, left-to-right order. If any sub-pattern  match fails, then the whole match fails.

   * For a ``deconstruct_pattern``:

     a. If this is the first ``deconstruct_pattern``:
    
        * Store ``type(obj)`` in ``$cls``
        * If this is the first ``deconstruct_pattern`` to contain named attributes, then call ``__attributes__`` and store it to ``$attrs``.
        * Call ``__deconstruct__`` and store it in ``$values``.

     b. If ``not issubclass($cls, pcls)`` where ``pcls`` is the result of evaluating the expression defining the class in the pattern, then the match fails.
     c. If the pattern contains named patterns, then pattern variables are assigned with ``$n = $values[$attrs.index(name)]``.
     d. If the pattern does not contain named patterns, then pattern variables are assigned with ``$n = $values[n]``.
     
        .. note::
          If ``$attrs.index(name)`` raises a ``ValueError``, the implementation must convert it to an ``AttributeError``.
          If ``$values[n]`` raises an ``IndexError``, the implementation must convert it to a ``TypeError``.
          Implementations are encouraged to provide meaningful error messages in these cases.

     c. Check each sub-pattern for a match in depth-first, left-to-right order. If any sub-pattern match fails, then the whole match fails.

3. If the pattern matches, then check the guard:

   * Fail to match if the the guard evaluates false.

Once a match has been found, then perform the assignment and execute the case body.

Additional desugaring for complex patterns
''''''''''''''''''''''''''''''''''''''''''

The above specification does not state how values are assigned to the capture variables.

For capture patterns, it is explicit. However the more complex patterns need an addtional step.

Inner patterns are desugared in the same way as outer patterns, taking advantage of the scoping rules to avoid name clashes.
Note that the scope of the assignment is the enclosing scope.

For example:

::

  case Cls($1, Cls(0, $2, $3), $4) -> a,b,c,d:

desugars by renumbering the inner pattern and assigning the inner pattern variables to the outer pattern variables.

::

  case Cls($0, (Cls(0, $0, $1)->$1, $2), $3) -> a,b,c,d

Implementation
==============

Implementation Rules
--------------------

Implementations should obey the "as if" rule. That is, they should behave as if the above sequence of operations occurs.
Implementations are thus free to reorder operations that have no observable side effects.

They are also free to consider that loading a global or class-local variable has no side-effects,
even though in some obscure circumstances it might have.

Implementations are also free to terminate iteration over a sequence early,
if further iteration is not needed to determine which case to execute.

Implementation stategy
----------------------

The following is not part of the specification,
but guidelines to help developers create an efficient implementation.

Splitting evaluation into lanes
'''''''''''''''''''''''''''''''

Since the first step in matching each pattern is check to against the kind, it is possible to move the check against kind to the beginning 
of the match. The list of cases can then be duplicated into several "lanes" each corresponding to one kind of pattern.
It is then trivial to remove unmatchable cases from each lane.
Depending on the kind, different optimization strategies are possible for each lane.

Capture and value patterns
''''''''''''''''''''''''''

Both of these forms trivially decompose into a series of tests, and should be compiled as the equivalent ``if``, ``elif`` statement.

Sequence patterns
'''''''''''''''''

This is probably the most complex to optimize and the most profitable in terms of performance.
Since each pattern can only match a range of lengths, often only a single length,
the sequence of tests can be rewitten in as an explicit iteration over the sequence,
attempting to match only those patterns that apply to that sequence length.

For example:

::

    case []:
        A
    case [$0] -> x:
        B
    case [$0, $1] -> x, y:
        C
    case $0:
        D

Can be compiled roughly as:

::

    # Choose lane
    __tmp = iter(obj)
    for $0 in __tmp:
        break
    else:
        A
        goto done
    for $1 in __tmp:
        break
    else:
        x = $0
        B
        goto done
    for $2 in __tmp:
        break
    else:
        x = $0
        y = $1
        C
        goto done
    D
  done:


For variable length matches, rather than attempt to slice the list, it is probably more efficient to store
the values before the ``*`` as discrete values, then pop the values are the ``*`` leaving the resulting list.

For example:

::

    case [$0, *$1, $2] -> a,b,c:

Can be compiled roughly as:

::

  # Choose lane
  __tmp = iter(obj)
  $0 = next(__tmp) # If this fails jump to next case
  $1 = list(__tmp)
  $2 = $1.pop() # If this fails, del $0, $1 and jump to next case
  a = $0
  b = $1
  c = $2


Mapping patterns
''''''''''''''''

The best stategy here is probably to form a decision tree based on which keys are present.
There is no point repeatedly testing for the presence of an attribute.
For example:

::

  match obj:
      case {a=$0,b=$1}:
          X # includes assignment and cleanup of $0, $1
      case {a=$0,c=$1}:
          Y
      case $0:
          Z

If the key ``"a"`` is not present when checking for case X, there is no need to check it again for Y.

The mapping lane can be implemented, roughly as:

::

  # Choose lane
  __tmp = obj
  if "a" in __tmp:
      if "b" in __tmp:
          $0 = __tmp["a"]
          $1 = __tmp["b"]
          goto X
      if "c" in __tmp:
          $0 = __tmp["a"]
          $1 = __tmp["c"]
          goto Y
  $0 = __tmp
  goto Z


Desconstruct patterns
'''''''''''''''''''''

This offers the least opportunity for optimisation. If there are multiple cases for the same class,
then a similar optimisation as for mapping might be used to avoid some tests.


Handling temporary variables
''''''''''''''''''''''''''''

For a stack machine based implementation, like CPython and PyPy,
keeping temporary variables on the stack seems like the obvious strategy.
It avoid any leakage of local variables, and is efficient.

Example
-------

Consider the match statement

::

    match x:
        case [$1] -> a:
            # Match a single element sequence
            A
        case [$1, $2] if $1 > 3 -> a, b:
            # Match a two element sequence if the first element is greater than 3
            B
        case [$1, a] -> x:
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
        case $1 -> f:
            # Match anything else
            F

Any implementation of pattern matching will need several new instructions.
For this example, we introduce two new instructions.

* ``MATCH_KIND`` which pushes ``type(tos).__match_kind__()`` where ``tos`` is the value on top of the stack.
* ``PEEK n`` which pushes a copy of the nth value on the stack.

Several other new bytecodes would be needed for a full implementation.

This example can be compiled reasonably efficiently, using the above stategy and assuming that the compiler
performs dead code elimination, jump fusion and peephole optimizations.

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
    POP_TOP
    POP_TOP
    POP_TOP
    JUMP case_E_or_F
  case_A:
    STORE_FAST a
    POP_TOP
    # Code for A
    JUMP end
  case_B_or_C_or_E_or_F:
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
    CALL_FUNCTION 1
    COMPARE_OP ==
    POP_JUMP_IF_FALSE case_F
    POP_TOP
    # Code for E
    JUMP end
  case_F:
    STORE_FAST f
    # Code for F
  end:

The above code is efficient as it checks against patterns as the sequence is iterated over.
It does this while obeying the specified semantics as it acts *as if* each pattern were matched in turn.


Conclusion
==========

It is possible to have precise semantics for pattern matching that performs well and works with the object model,
regardless of the syntax chosen.

Having precise semantics helps, not hinders, optimization
---------------------------------------------------------

Having precise semantics means that the range of possible implementations is well defined.
It is possible to determine what is a legal transformation and what is not.

Without precise semantics, any changes to the implementation have to verified in an ad-hoc way,
relying on test suites capturing the full range of behavior that is relied upon by users.
The implementation, whatever it is, may become the de facto specification.


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
