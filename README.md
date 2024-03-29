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

Caveat: being short does not mean that you can read faster than usual - in fact,
exactly the opposite. The text is very dense, and every sentence is packed with
a lot of information: be sure to take your time.

The code itself is in [Clojure](https://clojure.org/), a popular modern Lisp
dialect - the only thing that should not be instantly comprehensible for Lispers
is the following built-in macro: `#(... %)` is an alternative syntax for
denoting a lambda, the same as `(fn [x] (... x))` - Clojure's equivalent of
`(lambda (x) (... x))` -, with `%` representing the place of the first or only
function argument in the body.  I will use that form to denote those lambdas
that are really just redundant wrappers, serving no other purpose than to delay
the evaluation of their wrapped expression.

Problem
---
We have an anonymous function `f` that we'd like to be able to call recursively,
without the means of explicit self-reference.

```clojure
(def f (fn [x] (
         ;; point of recursion somewhere in body
       )))
```

Solution
---
The **Y combinator** is a clever function that can create a modified version of
`f` that is able to recurse without knowing its own name. I will first show the
so-called strict version of the Y combinator, commonly called the **Z
combinator** - the subtle difference will be addressed later. 

As a preliminary step, we need to move the self-reference out from the function
`f`. That is, the Z combinator does not work on the original function, but
expects a wrapped version of `f` (a maker for `f`, if you like), that returns an
`f` that calls the maker's _argument_ - a bound variable in the enclosing scope,
that's perfectly OK - at the point of recursion.

```clojure
(def f-maker (fn [f-self]
               ;; Spoiler: the function returned here will be a clever,
               ;; "self-replicating" version of our original `f`, if
               ;; provided with the right form of `f-self` by the maker.
               (fn [x] (
                 ;; call the provided `f-self` at the point of recursion
               ))))
```

Now the trick waiting to be performed by us is passing this new kind of `f`
_itself_ as argument to `f-maker` somehow. On how that can be achieved in an
indefinite recursive situation, without manually feeding `f-maker` with an `f`
that is created by an `f-maker` taking an `f` that is created by... and so on,
that is, writing out an unfinishable chain of nested calls, see the rest.

Without further ado, the magic function:

```clojure
(def Z (fn [f-maker]
         ((fn [self] (f-maker #((self self) %)))
          (fn [self] (f-maker #((self self) %))))))
```

A helper function that might make it easier to read: `self-apply` (alias U
combinator) is simply `(fn [x] (x x))`.

```clojure
(def Z (fn [f-maker]
         (self-apply
           (fn [self] (f-maker #((self-apply self) %))))))
```

The important thing to note above is that the `(self-apply self)` form inside,
when called, will expand to the body of `Z` itself, _the very form with which we
have started_. That is, `Z` calls `f-maker` with a box containing `Z`'s own
body, and thus _loads it into the resulting function_. This is a - probably the - 
key step to understand here, the rest follows pretty trivially. If the "why"
is not clear yet, just walk through the substitutions step by step; it is a
crucial exercise.

In the end, this results in getting a version of our original `f` (let it be
`self-replicating-f`) that has the same body as `f`, except that it has the
means to recreate itself on demand: at the point of recursion, it has the body
of `Z`_, with_ `f-maker` _already bound_, boxed in, ready to be evaluated.

Once that happens, i.e., when a recursive call is initiated in our supercharged
`self-replicating-f`, the box opens, and the whole machinery inside starts
moving again, with the body of `Z` - via calling `f-maker` - ultimately reducing
itself to the same `self-replicating-f`, that is finally ready to take its
argument (the one being passed in the recursive call) and execute.

```clojure
;; self-replicating-f
(fn [x] (
  ;; call `#((self-apply self) %)` i.e. `#(Z-body-with-f-maker-enclosed %)`
  ;; i.e. `#(self-replicating-f %)` at the point of recursion
))
```

And that's pretty much it.

```clojure
(def self-replicating-f (Z f-maker))
```

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
         (self-apply
           (fn [self] (f-maker (self-apply self))))))
```

TODO
---
- [ ] Go through and summarize [McAdam's
  paper](http://www.lfcs.inf.ed.ac.uk/reports/97/ECS-LFCS-97-375/) on the
  practical applications of the concept.

