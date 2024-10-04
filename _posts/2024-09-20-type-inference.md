---
title: Little implementations of the various Damas-Hindley-Milner inference algorithms
layout: post
co_authors: River Dillon Keefer
---

## What is Damas-Hindley-Milner?

A Damas-Hindley-Milner (HM) type system is a type system for the lambda
calculus (later adapted for Standard ML and the ML-family languages) with
parametric polymorphism, aka generic functions. It sits at a sweet spot in PL
design: the type system is quite expressive, and there are well known type
inference algorithms that require absolutely no annotations from the
programmer. 

TODO: link to papers from Damas, Hindley, and Milner

The type system is limited, but by virtue of being limited, it confers these
advantages:

* Inference algorithms tend to be fast (roughly O(size of code))
* Something something principal types
* ??? [It's a good type system, reader!](https://stackoverflow.com/a/399392)

<!--As a general note: the more constrained your language/system is, the more you
can optimize-->

In this post, we implement HM two ways, and then extend it a little bit. We'll
do this in the context of [scrapscript](/blog/scrapscript), but the goal is to
get a better understanding of HM in general.

We'll start with Algorithm W because it's the OG, and doesn't depend on side
effects like Algorithm J does. It *is* definitely more visually confusing than
Algorithm J, though, so if you get discouraged, you might want to skip ahead to
Algorithm J.

## The data structures

Every expression has exactly one type, called a monotype (we'll get to
generalization later). For this type checker, instead of working with values,
such as `5`, we're evaluating a program composed of types (`int`) and
operations on types (`list`). For our purposes, a monotype is either a type
variable like `a` or a type constructor like `->`, `list`, etc.

We represent that divide in python with classes:

```python
@dataclasses.dataclass
class MonoType:
    pass


@dataclasses.dataclass
class TyVar(MonoType):
    name: str


@dataclasses.dataclass
class TyCon(MonoType):
    name: str
    args: list[MonoType]
```

A lot of people make HM type inference implementations by hard-coding functions
and other type constructors like `list` as the only type constructors but we
instead model them all in terms of `TyCon`:

```python
IntType = TyCon("int", [])
BoolType = TyCon("bool", [])

def list_type(ty: MonoType) -> MonoType:
    return TyCon("list", [ty])

def func_type(arg: MonoType, ret: MonoType) -> MonoType:
    return TyCon("->", [arg, ret])
```

With these, we model the world.

## Algorithm W

Let's start with Algorithm W. It's probably the most famous one (citation
needed) because it was presented in the paper as the easiest to prove correct.
It's also purely functional, which probably appeals to Haskell nerds.

The idea is that you have a function `infer_w` that takes an expression and an
environment (a "context") and returns a substitution and a type. The
substitution is a mapping from type variables to monotypes and the type is the
type of the expression that you passed in. In Python syntax, that's:

```python
Subst = typing.Mapping[str, MonoType]  # type variable -> monotype
Context = typing.Mapping[str, Forall]  # program variable -> type scheme
def infer_w(expr: Object, ctx: Context) -> tuple[Subst, MonoType]: ...
```

Note that while substitutions map type variables to monotypes,
contexts map source-level variable names to type schemes.

The rules of inference are as follows:

* if you see a function `e`,
  * invent a new type variable for the parameter `t` and add it
    to the environment while type checking the body `b`
  * constrain the type of `e` to be a function from `t` to `type(b)`
  * return `type(e)`
* if you see function application `e`,
  * infer the type of callee `f`
  * infer the type of the argument `a`
  * constrain `type(f)` to be a function from `type(a)` to `type(e)`
  * return `type(e)`
* if you see a variable `e`,
  * look up the scheme of `e` in the environment
  * instantiate the scheme and return it
* if you see a let binding `let n = v in b` (called "where" in scrapscript) `e`,
  * infer the type of the value `v`
  * generalize `type(v)` to get a scheme `s`
  * add `n: s` to the environment while type checking the body `b`
  * return `type(b)`

Generalize is kind of like the opposite of instantiate. It takes a type and
turns it into a scheme using its free variables:

```python
def generalize(ty: MonoType, ctx: Context) -> Forall: ...
```

In order to keep the constraints (substitutions) flowing after each recursive
call to `infer_w`, we need to be able to compose substitutions. It's not just a
union of two dictionaries, but instead more like function composition

```python
def compose(s1: Subst, s2: Subst) -> Subst: ...

def apply_ty(ty: MonoType, subst: Subst) -> MonoType: ...

def apply_ctx(ctx: Context, subst: Subst) -> Context: ...

def unify_w(ty1: MonoType, ty2: MonoType) -> Subst:
    if isinstance(ty1, TyVar):
        return bind_var(ty2, ty1.name)
    if isinstance(ty2, TyVar):  # Mirror
        return unify_w(ty2, ty1)
    if isinstance(ty1, TyCon) and isinstance(ty2, TyCon):
        if ty1.name != ty2.name:
            unify_fail(ty1, ty2)
        if len(ty1.args) != len(ty2.args):
            unify_fail(ty1, ty2)
        result: Subst = {}
        for l, r in zip(ty1.args, ty2.args):
            result = compose(
                unify_w(apply_ty(l, result), apply_ty(r, result)),
                result,
            )
        return result
    raise TypeError(f"ICE: Unexpected type {type(ty1)}")


def infer_w(expr: Object, ctx: Context) -> tuple[Subst, MonoType]:
    if isinstance(expr, Var):
        scheme = ctx.get(expr.name)
        if scheme is None:
            raise TypeError(f"Unbound variable {expr.name}")
        return {}, instantiate(scheme)
    if isinstance(expr, Int):
        return {}, IntType
    if isinstance(expr, Function):
        arg_tyvar = fresh_tyvar()
        assert isinstance(expr.arg, Var)
        body_ctx = {**ctx, expr.arg.name: Forall([], arg_tyvar)}
        body_subst, body_ty = infer_w(expr.body, body_ctx)
        return body_subst, TyCon("->", [apply_ty(arg_tyvar, body_subst), body_ty])
    if isinstance(expr, Apply):
        s1, ty = infer_w(expr.func, ctx)
        s2, p = infer_w(expr.arg, apply_ctx(ctx, s1))
        r = fresh_tyvar()
        s3 = unify_w(apply_ty(ty, s2), TyCon("->", [p, r]))
        return compose(compose(s3, s2), s1), apply_ty(r, s3)
    if isinstance(expr, Where):
        name, value, body = expr.binding.name.name, expr.binding.value, expr.body
        s1, ty1 = infer_w(value, ctx)
        ctx1 = dict(ctx)  # copy
        ctx1.pop(name, None)
        scheme = generalize(ty1, apply_ctx(ctx1, s1))
        ctx2 = {**ctx, name: scheme}
        s2, ty2 = infer_w(body, apply_ctx(ctx2, s1))
        return compose(s2, s1), ty2
    raise TypeError(f"Unexpected type {type(expr)}")
```

## Algorithm M

https://www.classes.cs.uchicago.edu/archive/2007/spring/32001-1/papers/p707-lee.pdf

Pass in a type variable? Annotate the AST?

Top-down (upside-down W, ha ha)

## Algorithm J

Union-find

Instead of explicitly threading through and composing substitutions, we
implicitly modify the type variables as a way to keep track of the environment.

```python
def unify_j(ty1: MonoType, ty2: MonoType) -> None:
    ty1 = ty1.find()
    ty2 = ty2.find()
    if isinstance(ty1, TyVar):
        ty1.make_equal_to(ty2)
        return
    if isinstance(ty2, TyVar):  # Mirror
        return unify_j(ty2, ty1)
    if isinstance(ty1, TyCon) and isinstance(ty2, TyCon):
        if ty1.name != ty2.name:
            unify_fail(ty1, ty2)
            return
        if len(ty1.args) != len(ty2.args):
            unify_fail(ty1, ty2)
            return
        for l, r in zip(ty1.args, ty2.args):
            unify_j(l, r)
        return
    raise TypeError(f"Unexpected type {type(ty1)}")


def infer_j(expr: Object, ctx: Context) -> TyVar:
    result = fresh_tyvar()
    if isinstance(expr, Var):
        scheme = ctx.get(expr.name)
        if scheme is None:
            raise TypeError(f"Unbound variable {expr.name}")
        unify_j(result, instantiate(scheme))
        return result
    if isinstance(expr, Int):
        unify_j(result, IntType)
        return result
    if isinstance(expr, Function):
        arg_tyvar = fresh_tyvar("a")
        assert isinstance(expr.arg, Var)
        body_ctx = {**ctx, expr.arg.name: Forall([], arg_tyvar)}
        body_ty = infer_j(expr.body, body_ctx)
        unify_j(result, TyCon("->", [arg_tyvar, body_ty]))
        return result
    if isinstance(expr, Apply):
        func_ty = infer_j(expr.func, ctx)
        arg_ty = infer_j(expr.arg, ctx)
        unify_j(func_ty, TyCon("->", [arg_ty, result]))
        return result
    if isinstance(expr, Where):
        name, value, body = expr.binding.name.name, expr.binding.value, expr.body
        value_ty = infer_j(value, ctx)
        body_ty = infer_j(body, {**ctx, name: Forall([], value_ty)})
        unify_j(result, body_ty)
        return result
    raise TypeError(f"Unexpected type {type(expr)}")
```

## Extensions for Scrapscript

Adding type system features that go beyond HM in terms of expressivity often
requires some type annotations from the programmer, or adding significant
complexity to the inference algorithm.
### Recursion

Limited recursion: if typing the pattern `f = FUNCTION` or `f =
MATCH_FUNCTION`, then bind `f` to some new type variable to "tie the knot" in
the context

```python
def infer_j(expr: Object, ctx: Context) -> TyVar:
    # ...
    if isinstance(expr, Where):
        name, value, body = expr.binding.name.name, expr.binding.value, expr.body
        if isinstance(value, (Function, MatchFunction)):
            # Letrec
            func_ty = fresh_tyvar()
            value_ty = infer_j(value, {**ctx, name: Forall([], func_ty)})
        else:
            # Let
            value_ty = infer_j(value, ctx)
        # ...
```

In an ideal world, we would have a way to type mutual recursion. I think this
involves identifying call graphs and strongly connected components within
thiose graphs

### Let polymorphism

Hindley Milner types also include a `forall` quantifier that allows for some
amount of polymorphism. Consider the function `id = x -> x`. The type of `id`
is `forall 'a. 'a -> 'a`. This is kind of like a lambda for type variables. The
`forall` construct binds type variables like normal lambdas bind normal
variables. Some of the literature calls these *type schemes*.

```python
@dataclasses.dataclass
class Forall:
    tyvars: list[TyVar]
    ty: MonoType

    def __str__(self) -> str:
        return f"(forall {', '.join(map(str, self.tyvars))}. {self.ty})"
```

You can't directly use a `Forall` in a type expression. Instead, you have to
*instantiate* ("call", "apply") the `Forall`. This replaces the bound variables
with new variables in the right hand side---in the type. For example,
instantiating `forall 'a. 'a -> 'a` might give you `'t123 -> 't123`.

```python
def instantiate(scheme: Forall) -> MonoType: ...
```

If you want to have polymorphic functions, you need to find a cut point for
where to generalize. Maybe this means adding a `generic` keyword or maybe this
means making all `let`-bound variables polymorphic.

We chose to do the latter. This means that we need to take the type of the
function (say, `a -> b -> c`) and generalize it before adding it to the
environment:

Before: `('t10->('t11->('t12->'t10)))`

After: `(forall 't10, 't11, 't12. ('t10->('t11->('t12->'t10))))`

```python
def infer_j(expr: Object, ctx: Context) -> TyVar:
    # ...
    if isinstance(expr, Where):
        name, value, body = expr.binding.name.name, expr.binding.value, expr.body
        value_ty = infer_j(value, ctx)
        value_scheme = generalize(recursive_find(value_ty), ctx)  # changed!
        body_ty = infer_j(body, {**ctx, name: value_scheme})
        unify_j(result, body_ty)
        return result
    # ...
```

Due to our union-find implementation, we also need to do this "recursive find"
thing that calls `.find()` recursively to discover all of the type variables in
the type. Otherwise we might just see `'t0` or something.

### Pattern matching

What's the type of a match case pattern? Until a couple of days ago, I didn't
know. Turns out, it's the type that it looks like it should be, as long as you
bind all the variables in the pattern to fresh type variables.

```python
def infer_j(expr: Object, ctx: Context) -> TyVar:
    # ...
    if isinstance(expr, MatchCase):
        pattern_ctx = collect_vars_in_pattern(expr.pattern)
        body_ctx = {**ctx, **pattern_ctx}
        pattern_ty = infer_j(expr.pattern, body_ctx)
        body_ty = infer_j(expr.body, body_ctx)
        unify_j(result, func_type(pattern_ty, body_ty))
        return result
```

Then we unify all of the case functions to make the pattern types line up and
the return types line up.

```python
def infer_j(expr: Object, ctx: Context) -> TyVar:
    # ...
    if isinstance(expr, MatchFunction):
        for case in expr.cases:
            case_ty = infer_j(case, ctx)
            unify_j(result, case_ty)
        return result
```

### Row polymorphism

Scrapscript has records (kind of like structs) and run-time row polymorphism.
This means that you can have a function that pulls out a field from a record
and any record with that field is a legal argument to the function.

See for example two different looking records (2D point and 3D point):

```
get_x left + get_x right
. left  = { x = 1, y = 2 }
. right = { x = 1, y = 2, z = 3 }
. get_x = | { x = x, ... } -> x
```

Hindley Milner doesn't come with support for this right out of the box. If you
add support for records, then you end up with a more rigid system: the records
have to have the same number of fields and same names of fields and same types
of fields. This is safe but overly restrictive.

I think it's possible to "easily" add row polymorphism but we haven't done it
yet. Finding a simple, distilled version of the ideas in the papers has so far
been elusive.

<!--
RowSelect, RowExtend, RowRestrict
-->

* http://www.cs.cmu.edu/~aldrich/courses/819/papers/row-poly.pdf
* https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/scopedlabels.pdf
* https://cs.ioc.ee/tfp-icfp-gpce05/tfp-proc/21num.pdf
* https://github.com/zrho/libra-types/blob/1443ee7e31625a1b9278c29cefb4da044be1c90b/book/src/type_system.md

### Defer-dynamic

Unify doesn't fail but leaves `dyn` and/or run-time check

### Variants

* https://caml.inria.fr/pub/papers/garrigue-polymorphic_variants-ml98.pdf
* https://drops.dagstuhl.de/storage/00lipics/lipics-vol263-ecoop2023/LIPIcs.ECOOP.2023.17/LIPIcs.ECOOP.2023.17.pdf

### Canonicalization or minification of type variables

### Type-carrying code

Can we make hashes of types?

## See also

* Biunification (like CubiML)
* Static Basic Block Versioning
* CFA / Lambda set defunctionalization
  * River's STLC
* https://www.reddit.com/r/ProgrammingLanguages/comments/ijij9o/beyond_hindleymilner_but_keeping_principal_types/
* https://okmij.org/ftp/ML/generalization.html