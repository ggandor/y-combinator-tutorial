Y combinator: A very short explanation
===

The following aims to be the most distilled explanation of the Y combinator I
could possibly come up with, that - according to my hopes - still remains
comprehensible.

This is not a bottom-up derivation, but a _top-down_ explanation: I will just
show the solution to the problem, and then explain _why_ it works, and what is
the idea behind it.

You might want to read
[one](http://blog.tomtung.com/2012/10/yet-another-y-combinator-tutorial/) or
[two](https://www.cs.toronto.edu/~david/courses/csc324_w15/extra/ycomb.html)
other, more verbose tutorials in the former category before reading this one. If
you're just on the verge of grokking, however, this might be the final push that
clicks everything into place. Or you can do it the other way around, and read
this as a warm-up - however you like it. I expect you to already have at least
some understanding of lambda calculus and functional programming in general
though.

The code uses Clojure syntax - the only thing that should not be instantly
comprehensible for Lispers is the following built-in macro: `#(foo %)`, is the
same as `(fn [x] (foo x))`, `%` representing the (first or only) function
argument.

So here it comes:

Problem
---
We have an (anonymous) function `f` that we'd like to be able to call
recursively, without explicit self-reference.

```clojure
(def f (fn [x] (... point-of-recursion-somewhere-in-body ...)))
```

Solution
---
With the Y combinator, we can _create_ a modified version of `f` that is able to
recurse without knowing its own name. I will first show the so-called strict
version of the Y combinator, commonly called the Z combinator - the subtle
difference will be addressed later. 

`Z` will take `f-maker`, a wrapped version of `f` (or constructor for `f`, if
you like), that uses its _argument_ at the point of recursion. So ultimately it
expects _itself_ as argument - on how that can be done in a recursive situation
without "manually" feeding the function, writing out endless chains of nested
calls, see the rest.

```clojure
(def f-maker (fn [self]
               (fn [x] (... will call the provided self argument
                            at the point of recursion ...))))
```

A helper function making the Z combinator code easier to read: `self-apply` is
simply `(fn [x] (x x))`. This is the so-called U combinator by the way.

And now, the magic function:

```clojure
(def Z (fn [f-maker]
         (self-apply (fn [self]
                       (f-maker #((self-apply self) %))))))
```

The important thing to note here is that the `(self-apply self)` call inside
will expand to the body of `Z` itself, _the very form with which we have
started_. That is, `Z` calls `f-maker` with its - `Z`'s - own body, loading it
into the resulting function. This is a - probably _the_ - key step to understand
here - the rest follows pretty trivially. If the "why" is not clear yet, just
walk through the substitutions step by step; it is a crucial exercise.

In the end, this results in returning a version of `f` (let it be
`self-replicating-f`) that has the same body as `f`, except that at the point of
recursion, it has _the body of_ `Z`_, with_ `f-maker` _already bound_,
burnt-in. It's easy to see that calling that expression at the point of
recursion will ultimately return `self-replicating-f` again, that otherwise
works just like `f`, but can get itself returned at the point of recursion if
needed – and so on...

And that's pretty much it.

The "real" Y combinator
---

The Y combinator is the same as the Z combinator, except that it does not wrap
the `(self-apply self)` call (whose result is then passed to `f-maker`) in a
lambda. This only works in lazy languages though, like Haskell, where functions
evaluate their arguments only when needed, and not before executing the body.
Otherwise we'd be stuck in an infinite recursion - `(self-apply self)` would
expand forever after the first call.

```clojure
(def Y (fn [f-maker]
         (self-apply (fn [self]
                       (f-maker (self-apply self))))))
```

A biological allegory
---
The Y and Z combinators work a bit like loading the "genetic code" of a
function into the function itself. The following analogy might sound weird, but
I find it extremely illuminating: `Z` is like an ovum, `f-maker` is the sperm,
while the child, `self-replicating-f`, is a curious organism that has the
genetic information of both parents, but can reproduce itself without needing
another individual. This is _not_ asexual reproduction, however: the child and
its subsequent children are able to reproduce by _making an insemination happen
inside themselves_. A bit of a mindfuck, yeah.

