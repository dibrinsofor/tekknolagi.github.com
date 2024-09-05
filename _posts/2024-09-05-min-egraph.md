---
title: "What's in an e-graph?"
layout: post
date: 2024-09-05
---

*This post follows from several conversations with [CF
Bolz-Tereick](https://cfbolz.de/), [Philip
Zucker](https://www.philipzucker.com/), and various e-graphs people.*

I love union-find. It enables fast, easy, in-place IR rewrites for compiler
authors. Its API has two main functions: `union` and `find`. The minimal
implementation is about 15 lines of code and is embeddable directly in your IR.
See below an adaptation of CF's implementation from the toy optimizer
series[^also-phil]:

[^also-phil]: See also this tidy little union-find implementation by Phil from
    [his blog post](https://www.philipzucker.com/compile_constraints/):

    ```python
    uf = {}
    def find(x):
      while x in uf:
        x = uf[x]
      return x

    def union(x,y):
      x = find(x)
      y = find(y)
      if x != y:
        uf[x] = y
      return y
    ```

    I really enjoy that it reads like a margin note. The only downside, IMO,
    is that it requires the IR operations to be both hashable and comparable
    (`__hash__` and `__eq__`).

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Optional

@dataclass
class Expr:
    forwarded: Optional[Expr] = dataclasses.field(
        default=None,
        init=False,
        compare=False,
        hash=False,
    )

    def find(self) -> Expr:
        """Return the representative of the set containing `self`."""
        expr = self
        while expr is not None:
            next = expr.forwarded
            if next is None:
                return expr
            expr = next
        return expr

    def make_equal_to(self, other) -> None:
        """Union the set containing `self` with the set containing `other`."""
        found = self.find()
        if found is not other:
            found.forwarded = other
```

Union-find can be so fast because it is limited in its expressiveness:

* There's no built-in way to enumerate the elements of a set
* Each set has a single representative element
* We only care about the representative of a set

This is really great for some compiler optimizations. Consider the following
made-up IR snippet:

```
v0 = Const 1
v1 = Const 2
v2 = v0 + v1
```

A constant-folding pass can easily determine that `v2` is equivalent to `Const
3` and run `v2.make_equal_to(Const 3)`. This is an unalloyed good: making `v2`
constant might unlock some further optimization opportunities elsewhere and
there's probably no world in which we care to keep the addition representation
of `v2` around.

But not all compiler rewrites are so straightforward and unidirectional.
Consider the expression `(a * 2) / 2`, which is the example from the [e-graphs
good](https://egraphs-good.github.io/) website and paper. A strength reduction
pass might rewrite the `a * 2` subexpression to `a << 1` because left shifts
are often faster than multiplications. That's great; we got a small speedup.

Unfortunately, it stops another hypothetical pass from recognizing that
expressions of the form `(a * b) / b` are equivalent to `a * (b / b)` and
therefore equivalent to `a`. This is because rewrites that use union-find are
eager and destructive; we've gotten rid of the multiplication. How might we
find it again?

## Enumerating the equivalence classes

Let's make this more concrete and conjure a little math IR. We'll base it on
the `Expr` base class because it's rewritable using union-find.

```python
@dataclass
class Const(Expr):
    value: int

@dataclass
class Var(Expr):
    name: str

@dataclass
class BinaryOp(Expr):
    left: Expr
    right: Expr

@dataclass
class Add(BinaryOp):
    pass

@dataclass
class Mul(BinaryOp):
    pass

@dataclass
class Div(BinaryOp):
    pass

@dataclass
class LeftShift(BinaryOp):
    pass
```

It's just constants and variables and binary operations but it'll do for our
demo.

Let's also write a little optimization pass that can do limited constant
folding, simplification, and strength reduction. We have a function
`optimize_one` that looks at an individual operation and tries to simplify it
and a function `optimize` that applies `optimize_one` to a list of
operations---a basic block, if you will.

```python
def is_const(op: Expr, value: int) -> bool:
    return isinstance(op, Const) and op.value == value

def optimize_one(op: Expr) -> None:
    if isinstance(op, BinaryOp):
        left = op.left.find()
        right = op.right.find()
        if isinstance(op, Add):
            if isinstance(left, Const) and isinstance(right, Const):
                op.make_equal_to(Const(left.value + right.value))
            elif is_const(left, 0):
                op.make_equal_to(right)
            elif is_const(right, 0):
                op.make_equal_to(left)
        elif isinstance(op, Mul):
            if is_const(left, 1):
                op.make_equal_to(right)
            elif is_const(right, 1):
                op.make_equal_to(left)
            elif is_const(right, 2):
                op.make_equal_to(Add(left, left))
                op.make_equal_to(LeftShift(left, Const(1)))

def optimize(ops: list[Expr]):
    for op in ops:
        optimize_one(op.find())
```

Let's give it a go and see what it does to our initial smaller IR snippet that
added two constants[^printing-niceties]:

[^printing-niceties]: I sneakily added some printing niceties to the `Expr`
    class that I didn't show here. They're not important for the point I'm
    making and appear in the full code listing.

```python
ops = [
    a := Const(1),
    b := Const(2),
    c := Add(a, b),
]
print("BEFORE:")
for op in ops:
    print(f"v{op.id} =", op.find())
optimize(ops)
print("AFTER:")
for op in ops:
    print(f"v{op.id} =", op.find())
# BEFORE:
# v0 = Const<1>
# v1 = Const<2>
# v2 = Add v0 v1
# AFTER:
# v0 = Const<1>
# v1 = Const<2>
# v2 = Const<3>
```

Alright, it works. We can fold `1+2` to `3`. Hurrah. But the point of this
section of the post is to discover the equivalence classes implicitly
constructed by the union-find structure. Let's write a function to do that.

To build such a function, we'll need to iterate over all operations created. I
chose to explicitly keep track of every operation in a list, but you could also
write a function to walk the `forwarded` chains of all reachable operations.

```python
every_op = []

@dataclass
class Expr:
    # ...
    def __post_init__(self) -> None:
        every_op.append(self)

# ...

def discover_eclasses(ops: list[Expr]) -> dict[Expr, set[Expr]]:
    eclasses: dict[Expr, set[Expr]] = {}
    for op in ops:
        found = op.find()
        if found not in eclasses:
            # Key by the representative
            eclasses[found] = set()
        eclasses[found].add(op)
        if op is not found:
            # Alias the entries so that looking up non-representatives also
            # finds equivalent operations
            eclasses[op] = eclasses[found]
    return eclasses

# ...
print("ECLASSES:")
eclasses = discover_eclasses(every_op.copy())
for op in ops:
    print(f"v{op.id} =", eclasses[op])
# BEFORE:
# v0 = Const<1>
# v1 = Const<2>
# v2 = Add v0 v1
# AFTER:
# v0 = Const<1>
# v1 = Const<2>
# v2 = Const<3>
# ECLASSES:
# v0 = {Const<1>}
# v1 = {Const<2>}
# v2 = {Const<3>, Add v0 v1}
```

Let's go back to our more complicated IR example from the egg website, this
time expressed in our little IR:

```python
ops = [
    a := Var("a"),
    b := Const(2),
    c := Mul(a, b),
    d := Div(c, b),
]
```

If we run our optimizer on it right now, we'll eagerly rewrite the
multiplication into a left-shift, but then rediscover the multiply in the
equivalence classes (now I've added little `*` to indicate the union-find
representatives of each equivalence class):

```
BEFORE:
v0 = Var<a>
v1 = Const<2>
v2 = Mul v0 v1
v3 = Div v2 v1
AFTER:
v0 = Var<a>
v1 = Const<2>
v2 = LeftShift v0 v5
v3 = Div v6 v1
ECLASSES:
v0 = * {Var<a>}
v1 = * {Const<2>}
v2 =   {LeftShift v0 v5, Add v0 v0, Mul v0 v1}
v3 = * {Div v6 v1}
v4 =   {LeftShift v0 v5, Add v0 v0, Mul v0 v1}
v5 = * {Const<1>}
v6 = * {LeftShift v0 v5, Add v0 v0, Mul v0 v1}
```

That solves one problem: at any point, we can enumerate the equivalence classes
stored in the union-find structure. But, like all data structures, the
union-find representation we've chosen has a trade-off: fast to rewrite, slow
to enumerate. We'll accept that for now.

TODO parallel worlds of graphs

This enumeration feature on its own does not comprise one of the APIs of an
e-graph. To graft on e-matching to union-find, we'll need to do one more step:
a search. Some would call it `match`.

## Matching

So we can rediscover the multiplication even after reducing it to a left shift.
That's nice. But how can we do pattern matching on this data representation?

Let's return to `(a * b) / b`. This corresponds to the IR-land Python
expression of `Div(Mul(a, b), b)` for any expressions `a` and `b` (and keeping
the `b`s equal, which is not the default in a Python `match` pattern).

For a given operation, we can see if there is a `Div` in its equivalence class
by looping over the entire equivalence class:

```python
def optimize_match(op: Expr, eclasses: dict[Expr, set[Expr]]):
    # Find cases of the form a / b
    for e0 in eclasses[op]:
        if isinstance(e0, Div):
            # ...
```

That's all well and good, but how do we find if it's a `Div` of a `Mul`? We
loop again!

```python
def optimize_match(op: Expr, eclasses: dict[Expr, set[Expr]]):
    # Find cases of the form (a * b) / c
    for e0 in eclasses[op]:
        if isinstance(e0, Div):
            div_left = e0.left
            div_right = e0.right
            for e1 in eclasses[div_left]:
                if isinstance(e1, Mul):
                    # ...
```

Note how we don't need to call `.find()` on anything because we've already
aliased the set in the equivalence classes dictionary for convenience.

And how do we hold the `b`s equal? Well, we can check if they match:

```python
def optimize_match(op: Expr, eclasses: dict[Expr, set[Expr]]):
    # Find cases of the form (a * b) / b
    for e0 in eclasses[op]:
        if isinstance(e0, Div):
            div_left = e0.left
            div_right = e0.right
            for e1 in eclasses[div_left]:
                if isinstance(e1, Mul):
                    mul_left = e1.left
                    mul_right = e1.right
                    if mul_right == div_right:
                        # ...
```

And then we can rewrite the `Div` to the `Mul`'s left child:

```python
def optimize_match(op: Expr, eclasses: dict[Expr, set[Expr]]):
    # Find cases of the form (a * b) / b and rewrite to a
    for e0 in eclasses[op]:
        if isinstance(e0, Div):
            div_left = e0.left
            div_right = e0.right
            for e1 in eclasses[div_left]:
                if isinstance(e1, Mul):
                    mul_left = e1.left
                    mul_right = e1.right
                    if mul_right == div_right:
                        op.make_equal_to(mul_left)
                        return
```

If we run this optimization function for every node in our basic block, we end
up with:

```
AFTER:
v0 = Var<a>
v1 = Const<2>
v2 = LeftShift v0 v5
v3 = Var<a>
```

where `v3` corresponds to our original big expression. Congratulations, you've
successfully implemented a time-traveling compiler pass!

Unfortunately, it's very specific: our match conditions are hard-coded into the
loop structure and the loop structure (how many levels of nesting) is
hard-coded into the function. This is the sort of thing that our programming
giants invented SQL to solve[^egg-relational].

[^egg-relational]: This is a bit of a head-nod to the good folks working on egg
    and egglog, a team comprised of compilers people and database people. They
    have realized that the e-graph and the relational database are very similar
    and are building tools that do a neat domain crossover. Read [the
    paper](https://dl.acm.org/doi/10.1145/3591239) (open access) if you are
    interested in learning more!

We don't have time or brainpower to implement a full query language, so in this
post will implement a small pattern-matching DSL that kind of vaguely
maybe-if-you-squint looks like something relational.

TODO: a matching DSL

One thing to note: after every write with `make_equal_to`, we need to
rediscover the eclasses. I think this is what the egg people call a "rebuild"
and part of what made their paper interesting was finding a way to do this less
often or faster.

Now what we have is a bunch of parallel worlds for our basic block where each
operation is actually a set of equivalent operations. But which element of the
set should we pick? One approach, the one we were taking before, is to just
pick the representative as the desired final form of each operation. This is a
very union-find style approach. It's straightforward, it's fast, and it works
well in a situation where we only ever do strength reduction type rewrites.

But e-graphs popped into the world because people wanted to explore a bigger
state space. It's possible that the representative of an equivalence class is
locally optimal but not globally optimal. What if we want to find a program
with a cost function that takes into account the entire program?

## Extracting

## Further reading

* PyPy eager union-find
* egg(log)