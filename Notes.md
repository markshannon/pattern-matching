# Notes

These need to be moved into the relevant place


# Broader issues

These issues impinge upon pattern matching, but impact the language as a whole.

## Comparison issues

PEP 634 uses a different notion of equality and sequences than the broader Python language.

Having different semantics may cause confusion, but the existing semantics can cause confusion in common cases.

### The issues

#### Equality of integers 0 and 1 to boolean values.

In Python

```python
1 == True
0 == False
```
This can cause confusion when `1` matches `True` or vice versa.

#### Strings are sequences

```python
a,b,c = "abc"
```

causes means that
```python
a == "a"
b == "b"
c == "c"
```

This can cause confusion as strings are commonly thought of as scalar values.

### Solutions

There are three alternatives here:

1. Pattern matching should follow the same semantics as the rest of the language.
2. Pattern matching should treat these values specially in pattern matching.
3. Add new operations for equality and unpacking, and add them to the language as a whole.


## Distinguishing between sequences and mapping.

At its most abstract a sequence is a subtype of a mapping, that only allows integer keys in the range `range(0, len(seq))`.
However sequences are not generally thought of this way and are regarded as distinct.
Unfortunately, distinguishing between a sequence and a mappping is not a well defined operation.
A subclass test on `collections.abc.Sequence` requires that `collections.abc` is available, that the class in question has registered as a 
subclass of `collections.abc.Sequence`, and that it hasn't been unregistered (e.g. at shutdown).

Using duck-typing is unreliable as well, since mappings and sequences both implement `__getitem__`, `__iter__` and `__len__`.

### Solutions

Here are two possible options (they may be others, but they are not obvious to me):

1. Move `collections.abc.Sequence` into the core of the VM, such that it always exists.
This makes the test robust in the face of shutdown and missing `collections.abc`
2. Add some form of special tag on the class, like `__is_sequence__`,
to classes to allow duck-typing to distinguish between sequences and mapping.

Both approaches have the same weakness, that they require the class being tested to have declared that it is a sequence.

## Loads and stores of variables in patterns.

For a procedural language with mutable variables, like Python, the scope and lifetime of variables impacts the semantics and syntax of pattern matching.

There are a number of issues that must be addressed:

* Can variables be assigned during an attempted match, or only once a match succeeds?
* Does the scope of variables differ when they are used in a pattern?
* Is there any limitation to the scope of varaibles that can used in a pattern?
* How are loads and stores treated?
    * Are only loads allowed?
    * Are only stores allowed?
    * If both are allowed, how are they to be differentiated?

If both are to be allowed, then a marker of some sort is required, the following examples use `$`. `$var` is a single lexical token and is distinct from `var`.

### Internal or external assignment

Pattern matching performs two actions, it matches an object against a pattern and assigns values to variables. Syntactic these can be mixed or distinct.
We will call the mixed form, "internal" assignment and the distinct form "external assignment".

For example consider a pattern that matches a length 2 sequence and assigns the items to `a` and `b`.
* A possible internal assignment syntaxes could be `[a, b]`, or `[$a, $b]`
* A possible external assignment syntax could be `[$, $] -> a, b`, or `[$1, $2] -> a, b`, or `a,b = [$, $]`

Internal assignment is more concise. Which is preferable may depend on the chosen semantics.

### Determining what is a pattern, and what is an expression within a pattern.

Patterns need to select not only a class, or kind of object, they also need to
match values within an object.
To do so, those values must be evaluated from expressions.
The syntax for pattern matching must define what is an expression, to be evaluated normally,
and what is a sub-pattern to be matched.

For example, in PEP 634, in the pattern `[1, a]` `1` is a normal expression, but `a` is a sub-pattern.
So `[1,2]` matches, assigning `2` to `a`, but `[0,2]` does not match as `0 != 1`.

It is not possible to use the full syntax for expressions, and normal variable syntax for patterns as it is not clear in `[1,a]` whether a `a` is a variable to be assigned or an expression to be evaluated.

### Assignment in a pattern, when both load and stores are allowed.

Since a marker of some sort is required, the following examples use `$`. `$var` is a single lexical token and is distinct from `var`. This document does not argue for or against any particualr syntax. `$var` is merely used for the examples.

Possible syntactic approaches are:

* Use the full<sup>[1](#exprsyntax)</sup> syntax for expressions and mark pattern variables with a syntactic marker. 
   * For example, `[a,$a]` matches `[1,2]` if `a == 1` before the match. `a == 2` after the match.
* Use the full<sup>[1](#exprsyntax)</sup> syntax for expressions and mark any assignment with a syntactic marker. This allows more complex assignements.
   * For example, `[a, $self.a]` `[1,2]` if `a == 1` before the match. `self.a == 2` after the match.
* Use the full<sup>[1](#exprsyntax)</sup> syntax for expressions, except for variables which are marked. Pattern variables are unmarked. This disallows the use of more complex assignment targets.
   * For example, `[m.a, a]` matches `[1,2]` if `m.a == 1` before the match. `a == 2` after the match.
* 



## Notes

<a name="exprsyntax">1</a>: Or some restricted set. There are syntactic or implementation reasons for using a restricted set, but it may be chosen for other reasons.


