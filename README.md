
# Analysis of Pattern Matching for Python

This document attempts an objective analysis of the syntactic, semantic,
and implementation issues related to the possible addition of pattern matching to Python.

## Applicability of pattern matching to Python

This is an inherently subjective issue.

[An analysis of the standard library](.stdlib.md) shows 
that adding pattern matching does not change expressiveness.

That does not imply, however, that there is no change in comprehension or clarity.
To determine that would require a user server.

## What is pattern matching?

A pattern matching construct, whether expression or statement, takes a value and matches it against a list
of patterns. Once a matching pattern is found, elements of the matched value are bound, or assigned, to variables.
In a pattern matching expression, the bound values can be used only in the sub-expression for that pattern.
In a pattern matching statement, assigned variables can have the same restricted scope as an expression, or the variables
can have the same scope as normal variables.


## Syntactic issues

Python is a dynamically typed, procedural language.
Pattern matching has historically been used in functional languages,
especially statically typed languages.

This means that there are issues of side-effects and ambiguity that can occur in patterns for Python, that do not occur in Erlang or Haskell.

### Mutability

In a pure functional language, there are no mutable variable.
Instead of varaibles, "let bindings" are provided which allow the result of evaluating an expression to 
be named for convenience.

In functional languages, expressions are much more common that statements. Functional languages have pattern matching functions
rather than statements.

The combination of expressions and let bindings means that the bindings in a functional expression have restricted scope
