
# Analysis of Pattern Matching for Python

This document attempts an objective analysis of the syntactic, semantic,
and implementation issues related to the possible addition of pattern matching to Python.

## Applicability of pattern matching to Python

This is an inherently subjective issue.
Adding pattern matching does seem to be quite popular on the mailing lists.

[An analysis of the standard library](.stdlib.md) shows 
that adding pattern matching does not change expressiveness significantly.

That does not imply, however, that there is no change in comprehension or clarity.
To determine that objectively would require a user survey of some sort.

## What is pattern matching?

A pattern matching construct, whether expression or statement, takes a value and matches it against a list
of patterns. Once a matching pattern is found, elements of the matched value are bound, or assigned, to variables.
In a pattern matching expression, the bound values can be used only in the sub-expression for that pattern.
In a pattern matching statement, assigned variables can have the same restricted scope as an expression, or the variables
can have the same scope as normal variables.

### Mutability

Python is a dynamically typed, procedural language.
Pattern matching has historically been used in functional languages,
especially statically typed languages.

This means that there are issues of side-effects and ambiguity that can occur in patterns for Python, that do not occur in Erlang or Haskell.
Consequently designs for pattern matching from these languages do not seem to map well to Python.

## Design choices

Any pattern matching proposal needs to address the following syntactic issues:

* Should a match be an expression or statement?
* Scope of variables. Should the scope of variables be restricted to the case, or to follow normal scoping rules?
* How, if at all, should variable assignment within a pattern be denoted?
    * This seems to be the most controversial issue. It is explored in more depth below.

### Differentiating between values and assignment in pattern matching

This seems to be the main point of contention in competing pattern matching designs.

In the broadest of terms there are three possible options.
In no particular order they are:

1. Mark all assignments targets, all expressions are evaluated as normal. Either a special token can be used, or some existing token used.
2. Mark all values, only expressions that are marked are evaluated as normal. Other expressions are assignment targets.
3. Chose a limited set of expressions to be assignment targets and a different set to be values.
    * The sets may not overlap, but may be context dependent.

### Pros and cons of the approaches

The discussion below focuses only on the objective pros and cons.
Factors like ease of use and aesthetics are not discussed here, even though they are important.
Nor does this imply that all advantages and disadvantages have equal weight.
The relative weight of the points below will likely differ from user to user.

#### Mark all assignments targets

This has two advantages:

* It makes explicit, without context, whether an expression is evaluated, or is an assignment target
* All expressions are evaluated the same way as the expressions outside of a pattern.

And one disadvantage:
    
* All assignment targets must be marked, which adds clutter and may be considered unattractive or "unpythonic". The markers may be especially intrusive in simple patterns.

#### Mark all evaluated expressions, leaving assignments targets unmarked.

This has one advantage:

* It makes explicit, without context, whether an expression is evaluated, or is an assignment target

And two disadvantages:

* All expressions in patterns  must be marked, which adds clutter and may be considered unattractive or "unpythonic", although this may less intrusive than for option 1.
* Expressions have different syntax to that outside of a pattern. This means that there are two different syntaxes for equivalent expressions.

#### No explicit markers. Rely on context to distinguish.

This has one advantage:

* There is no need for markers; common patterns require less syntax.

And two disadvantages:

* Whether a syntactic element is an assignment target or an expression is not clear without examining the larger context. In complex patterns this might be confusing.
* Some expressions become impossible to express, as the syntax is taken for assignments targets.


### Existing proposals

PEP 634 chooses option 3.

PEP 642 chooses option 2.

Option 1 has been suggested on the mailing list, but there are no formal proposals.

## Semantics

Some semantic options to consider are:

* Can assignments be made for unsuccessful matches?
* Is it possible to assign to some but not all of the targets?
* Should the kind of patterns that can be matched be determined explicitly by the object,
by some properties of the object, or by some external registry?
* Does being eligible to match one kind of pattern preclude matching any or all of the other kinds?
* Is partial deconstruction of an object allowed or must it be complete?
* How does pattern matching interact with the rest of the language? Does it need additional special methods? Does it need certain modules to be present?


## Performance

What are the performance implications of a particular design?

## Other issues

These issues apply to the broader language as well, but have been brought up in the discussion of pattern matching

### Equality of integers 0 and 1 to boolean values.

In Python

```python
1 == True
0 == False
```
This can cause confusion when `1` matches `True` or vice versa.

Should this be handled as part of pattern matching, or would a broader solution applicable to the whole language be better?

### Strings are sequences

```python
a,b,c = "abc"
```

causes the following to be true
```python
a == "a"
b == "b"
c == "c"
```

This can cause confusion as strings are commonly thought of as scalar values.

Should strings be treated as sequences in patterns?
If not, should their behavior change for iteration or unpacking?

