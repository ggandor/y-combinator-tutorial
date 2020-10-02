Y combinator: A very short explanation
===

The following is the most distilled top-down explanation of the Y combinator I
could possibly come up with, that - according to my hopes - still remains
comprehensible. You might want to read this either as a warm-up before diving
deeper, or after reading
[one](http://blog.tomtung.com/2012/10/yet-another-y-combinator-tutorial/) or
[two](https://www.cs.toronto.edu/~david/courses/csc324_w15/extra/ycomb.html)
more verbose tutorials with a bottom-up approach, if the concept hasn't quite
clicked yet.

The code itself is in [Clojure](https://clojure.org/), a Lisp dialect - the only
thing that should not be instantly comprehensible for Lispers is the following
built-in macro: `#(foo %)` is an alternative syntax for denoting a lambda, the
same as `(fn [x] (foo x))`, with `%` representing the place of the (first or
only) function argument in the body. I will use that form to denote those
lambdas that are really just redundant wrappers, serving no other purpose than
to delay the evaluation of their wrapped expression.

So here it comes:

Problem
---
We have an anonymous function `f` that we'd like to be able to call recursively,
without explicit self-reference.

```clojure
(def f (fn [x] (
         ;; point of recursion somewhere in body
       )))
```

Solution
---
With the **Y combinator**, we can _create_ a modified version of `f` that is
able to recurse without knowing its own name. I will first show the so-called
strict version of the Y combinator, commonly called the **Z combinator** - the
subtle difference will be addressed later. 

`Z` will take `f-maker`, a wrapped version of `f` (or constructor for `f`, if
you like), that makes `f` call the provided _argument_ - a bound variable,
that's perfectly OK - at the point of recursion. Now the trick waiting to be
performed by us is passing this new kind of `f` _itself_ as argument to
`f-maker` somehow. On how that can be achieved in an indefinite recursive
situation, without manually feeding `f-maker` with an `f` that is created by an
`f-maker` taking an `f` that is created by... and so on, that is, writing out an
unfinishable chain of nested calls, see the rest.

```clojure
(def f-maker (fn [f-self]
               ;; Spoiler: this will be a clever, "self-replicating" version
               ;; of `f`, if provided with the right `f-self` argument.
               (fn [x] (
                 ;; call `f-self` at the point of recursion
               ))))
```

And now, the magic function:

```clojure
(def Z (fn [f-maker]
         ((fn [self] (f-maker #((self self) %)))
          (fn [self] (f-maker #((self self) %))))))
```

A helper function that might make it easier to read: `self-apply` is simply
`(fn [x] (x x))`. (This is the so-called U combinator by the way.)

```clojure
(def Z (fn [f-maker]
         (self-apply (fn [self]
                       (f-maker #((self-apply self) %))))))
```

The important thing to note above is that the `(self-apply self)` form inside,
when called, will expand to the body of `Z` itself, _the very form with which we
have started_. That is, `Z` calls `f-maker` with `Z`'s own body, and _loads it
into the resulting function_. This is a - probably _the_ - key step to
understand here - the rest follows pretty trivially. If the "why" is not clear
yet, just walk through the substitutions step by step; it is a crucial exercise.

In the end, this results in returning a version of our original `f` (let it be
`self-replicating-f`) that has the same body as `f`, except that it has the
means to recreate itself on demand: at the point of recursion, it has the body
of `Z`_, with_ `f-maker` _already bound_, burnt-in. It will look something like
this:

```clojure
(fn [x] (
  ;; call `#((self-apply self) %)` at the point of recursion
))
```

It's easy to see that calling that expression will ultimately return this
`self-replicating-f` again (since that is what `Z` does), that otherwise works
just like `f`, but can get itself returned at the point of recursion if needed –
and so on...

```clojure
(def self-replicating-f (Z f-maker))
```

And that's pretty much it.

The Y combinator
---
The Y combinator is the same as the Z combinator, except that it does not wrap
the `(self-apply self)` call in a lambda, conveniently delaying evaluation. This
only works in lazy languages though, like Haskell, where functions evaluate
their arguments only when needed, and not before executing the body. Otherwise
we'd be stuck in an infinite recursion - `(self-apply self)` would expand
forever after the first call, before it could be passed on to `f-maker`.

```clojure
(def Y (fn [f-maker]
         (self-apply (fn [self]
                       (f-maker (self-apply self))))))
```
