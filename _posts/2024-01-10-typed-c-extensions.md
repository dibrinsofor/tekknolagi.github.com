---
title: "Mining type information for faster Python C extensions"
layout: post
date: 2024-01-10
---

## what

pypy's c-api has some performance trouble. i am working on a way to make pypy's
c-api interactions much faster. it's looking very promising. here's a sketch of
how it works.

## intro

Python is pretty widely-used. For years, CPython was the only implementation,
and CPython was not designed to be fast. The Python community needed some
programs to go faster and determined that the best path forward was to write
some modules in C and interact with them from Python. Worked just fine.

Then other Python runtimes like PyPy came along. PyPy includes a JIT compiler
and its execution of normal Python code is *very* fast, at least until it hits
a call from Python to a C function. Then things go a little bit sideways. First
and foremost because PyPy can't "see into" the native code; it's generated
outside of the JIT and opaque. And second of all because the binding API for
the aforementioned C modules ("The C API") uses a totally different
representation than PyPy.

PyPy has its own object model, runtime, and moving garbage collector. This is
all to get better performance. Unfortunately, this means that whenever you call
a C API function from PyPy, it has to stop what it's doing, set up some C API
scaffolding, do the C API call, and then take down the scaffolding.

For example, the C API is centered around `PyObject` pointers. PyPy does not
normally use `PyObject`s. It has to allocate a `PyObject`, make it point into
the PyPy heap, call the C API function, and then (potentially) free the
`PyObject`.

<figure style="display: block; margin: 0 auto;">
    <object class="svg" type="image/svg+xml" data="https://mermaid.ink/svg/pako:eNptUMtOhTAQ_ZVmlgYJyKPQxU3Uu9GNJldjYthUGB5KWywlVyT8uwUfQWNX0zPnnDkzE-SqQGDQ4-uAMsd9wyvNRSaJfVIZJC2WhqiSXF_dMXJQAsntaGolyaL85NnW6W53SR407zrUjJy3rcq5Wag3T8-Ym5ONoW6qenXcCPaK9Nba1I2syLEx9V_lD9cOWpPcy6MF_h3wK_EFz1-IUdvQ4IBALXhT2MWnRZiBqVFgBsyWBZZ8aE0GmZwtlQ9GHUaZAzN6QAeGrrCbfd0JWMnb3qIdl49KiW-S_QKb4A1YQH3XDwPq0TMaBakXOzACo57rJwGlcRJFYZzQaHbgfdV7bpymYRB66QInaUznD-SXhKM">
    </object>
  <figcaption>Fig. 1 - something</figcaption>
</figure>
<!--
sequenceDiagram
    note left of JIT: Some Python code
    JIT->>C Wrapper: Allocate PyObject*
    note right of C Wrapper: Do something with PyObject*
    C Wrapper->>JIT: Unwrap PyObject*
    note left of JIT: Back to Python code
-->
<!-- https://mermaid.live/edit#pako:eNptUMtOhTAQ_ZVmlgYJyKPQxU3Uu9GNJldjYthUGB5KWywlVyT8uwUfQWNX0zPnnDkzE-SqQGDQ4-uAMsd9wyvNRSaJfVIZJC2WhqiSXF_dMXJQAsntaGolyaL85NnW6W53SR407zrUjJy3rcq5Wag3T8-Ym5ONoW6qenXcCPaK9Nba1I2syLEx9V_lD9cOWpPcy6MF_h3wK_EFz1-IUdvQ4IBALXhT2MWnRZiBqVFgBsyWBZZ8aE0GmZwtlQ9GHUaZAzN6QAeGrrCbfd0JWMnb3qIdl49KiW-S_QKb4A1YQH3XDwPq0TMaBakXOzACo57rJwGlcRJFYZzQaHbgfdV7bpymYRB66QInaUznD-SXhKM -->

That's a lot of overhead. (And there's more, too. See Antonio Cuni's [excellent
blog
post](https://www.pypy.org/posts/2018/09/inside-cpyext-why-emulating-cpython-c-8083064623681286567.html))[^capi-problem].

[^capi-problem]: This C API problem has bitten pretty much every alternative
    runtime to CPython. They all try changing something about the object model
    for performance and then they all inevitably hit the problem of pervasive C
    API modules in the wild. This is part of what hurt the
    [Skybison](https://github.com/tekknolagi/skybison) project.

Worse, the C API function may not even *need* the `PyObject` to exist in the
first place. For example, a lot of C API functions are structured like this:

```c
long foo_impl(long num) {
    return num * 2;
}

PyObject* foo(PyObject* obj) {
    long num = PyLong_AsLong(obj);
    if (num < 0 && PyErr_Occurred()) {
        return NULL;
    }
    long result = foo_impl(num);
    return PyLong_FromLong(result);
}
```

In this example, the `PyObject*` code is only a wrapper around another function
that works directly on C integers.

<figure style="display: block; margin: 0 auto;">
  <object class="svg" type="image/svg+xml" data="https://mermaid.ink/svg/pako:eNqVkV1LwzAUhv9KOJdSR2s_0uZioO5m3ihMEaQ3sT3d6pqcmqbMOfbfTTsnTkEwV0nO83KenOygoBJBQIevPeoCZ7VcGqlyzdzSZJE1WFlGFbuZ3wu2IIXsbmtXpNmQPHCudD6dXrNHI9sWjWCXTUOFtAN6-_yChT07gF_IiM9V26BCbaWtSQv2oDeu-jMzSph6uRotfqdmxLrBakNmfexyyvzfbXzrHzonM7mSxZpZ-j4W8EChUbIu3Wh3QzAHu3JGOQi3LbGSfWNzyPXeobK3tNjqAoQ1PXrQt6Xz-_wJEJVsOnfbSv1EpI6QO4LYwRuIkAeTIAq5zy94HGZ-4sEWBPcnQRpynqRxHCUpj_cevI95f5JkWRRGfhDGQRSlPNh_AK9DpdY">
  </object>
  <figcaption>Fig. 2 - something</figcaption>
</figure>
<!--
https://mermaid.live/edit#pako:eNqVkV1LwzAUhv9KOJdSR2s_0uZioO5m3ihMEaQ3sT3d6pqcmqbMOfbfTTsnTkEwV0nO83KenOygoBJBQIevPeoCZ7VcGqlyzdzSZJE1WFlGFbuZ3wu2IIXsbmtXpNmQPHCudD6dXrNHI9sWjWCXTUOFtAN6-_yChT07gF_IiM9V26BCbaWtSQv2oDeu-jMzSph6uRotfqdmxLrBakNmfexyyvzfbXzrHzonM7mSxZpZ-j4W8EChUbIu3Wh3QzAHu3JGOQi3LbGSfWNzyPXeobK3tNjqAoQ1PXrQt6Xz-_wJEJVsOnfbSv1EpI6QO4LYwRuIkAeTIAq5zy94HGZ-4sEWBPcnQRpynqRxHCUpj_cevI95f5JkWRRGfhDGQRSlPNh_AK9DpdY
-->
<!--
sequenceDiagram
    note left of JIT: Some Python code
    JIT->>C Wrapper: Allocate PyObject*
    C Wrapper->>C Implementation: Unwrap PyObject*
    note right of C Implementation: Do some work
    C Implementation->>C Wrapper: Allocate PyObject*
    C Wrapper->>JIT: Unwrap PyObject*
    note left of JIT: Back to Python code
-->

All of the bits in the middle between the JIT and the C implementation are
"wasted work" because it's not needed for the actual execution of the user's
program.

So even if the PyPy JIT is doing great work and has eliminated memory
allocation in Python code---PyPy could have unboxed some heap allocated int
into a C int---it still has to heap allocate a `PyObject*` ... only to throw
both away soon after.

If there was a way to communicate that `foo` expects an `int` and is going to
unbox it into a C `int` (and will also reutrn a C `int`) to PyPy, it wouldn't
need to do any of these shenanigans.

proposal: <!-- TODO -->

diagram: <!-- TODO -->

And yes, ideally there wouldn't be a C API call at all. But sometimes you have
to (perhaps because you have no control over the code), and you might as well
speed up that call. Let's see if it can be done.

## Potsdam

This is where I come into this. I was at the ECOOP conference in 2022 where
[Carl Friedrich](https://cfbolz.de/) introduced me to some other implementors
of alternative Python runtimes. I got to talk to the authors of PyPy and ZipPy
and GraalPython over some coffee. They're really nice.

They've been working on a project called HPy. HPy is a new design for a C API
Python that takes alternative runtimes into account. As part of this design,
they were investigating a way to pipe type information from the C module
through the C API and into a place where the host runtime can read it.

It's a tricky problem because not only is there a C API, but also a C ABI (note
the "B"). While an API is an abstract contract between caller and callee, an
ABI is more concrete. In the case of the C ABI, it means not changing structs,
adding function parameters, things like that. This is kind of a tight
constraint and it wasn't clear what the best way forward was.

https://github.com/hpyproject/hpy/issues/129

I decided to take a stab at implementing it for Cinder

https://github.com/faster-cpython/ideas/issues/546

## Sketchy C things

This is the kind of type metadata we want to add to each typed method.

```c
struct PyPyTypedMethodMetadata {
  int arg_type;
  int ret_type;
  void* underlying_func;
};
typedef struct PyPyTypedMethodMetadata PyPyTypedMethodMetadata;
```

In this artificially limited example, we store the type information for one
argument (but more could be added in the future), the type information for the
return value, and the underlying (non-`PyObject*`) C function.

But it's not clear where to put that.

The existing `PyMethodDef` struct looks like this. It contains a little bit of
metadata and a C function pointer (the `PyObject*` one). In an ideal world, we
would "just" add the type metadata to this struct and be done with it. But we
can't change its size for ABI reasons.

```c
struct PyMethodDef {
    const char  *ml_name;   /* The name of the built-in function/method */
    PyCFunction  ml_meth;   /* The C function that implements it */
    int          ml_flags;  /* Combination of METH_xxx flags, which mostly
                               describe the args expected by the C func */
    const char  *ml_doc;    /* The __doc__ attribute, or NULL */
};
typedef struct PyMethodDef PyMethodDef;
```

What to do? Well, I decided to get a little weird with it and see if we could
sneak in a pointer to the metadata somehow. My original idea was to put the
entire `PyPyTypedMethodMetadata` struct *behind* the `PyMethodDef` struct, but
that wouldn't work so well: `PyMethodDef`s are commonly statically allocated in
arrays, and we can't change the layout of those arrays. But what we can do is
point the `ml_name` field to a buffer inside another struct[^linux-trick].

[^linux-trick]: later learned that this is common in the Linux kernel

Then, when we notice that a method is marked as typed (with a new `METH_TYPED`
flag we can add to the `ml_flags` bitset), we can read backwards to find the
`PyPyTypedMethodMetadata` struct. Here's how you might do that in C:

```c
struct PyPyTypedMethodMetadata {
  int arg_type;
  int ret_type;
  void* underlying_func;
  const char ml_name[100];  // New field!
};
typedef struct PyPyTypedMethodMetadata PyPyTypedMethodMetadata;

PyPyTypedMethodMetadata*
GetTypedSignature(PyMethodDef* def)
{
  assert(def->ml_flags & METH_TYPED);  // A new type of flag!
  return (PyPyTypedMethodMetadata*)(def->ml_name - offsetof(PyPyTypedMethodMetadata, ml_name));
}
```

And here's a diagram to illustrate this because it's really weird and
confusing.

<svg version="1.1" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 595.5 280.69167393781254" width="595.5" height="280.69167393781254">
  <!-- svg-source:excalidraw -->
  
  <defs>
    <style class="style-fonts">
      @font-face {
        font-family: "Virgil";
        src: url("https://excalidraw.com/Virgil.woff2");
      }
      @font-face {
        font-family: "Cascadia";
        src: url("https://excalidraw.com/Cascadia.woff2");
      }
      @font-face {
        font-family: "Assistant";
        src: url("https://excalidraw.com/Assistant-Regular.woff2");
      }
    </style>
    
  </defs>
  <rect x="0" y="0" width="595.5" height="280.69167393781254" fill="#ffffff"></rect><g stroke-linecap="round" transform="translate(301.5 172.8591812698021) rotate(0 142 48.5)"><path d="M24.25 0 C96.98 0.07, 170.08 1.31, 259.75 0 M24.25 0 C91.64 1.01, 159.28 0.63, 259.75 0 M259.75 0 C277.18 -1.81, 282.61 8.34, 284 24.25 M259.75 0 C277.04 0.75, 286.19 8.14, 284 24.25 M284 24.25 C283.22 35.88, 284.64 52.64, 284 72.75 M284 24.25 C285.26 35.92, 284.01 49.77, 284 72.75 M284 72.75 C283.04 89.11, 277.03 98.59, 259.75 97 M284 72.75 C282.59 88.86, 276.09 98.31, 259.75 97 M259.75 97 C202.55 96.59, 145.75 96, 24.25 97 M259.75 97 C194.6 98.64, 130.77 99.23, 24.25 97 M24.25 97 C7.55 96.82, -0.08 87.92, 0 72.75 M24.25 97 C8.82 95.01, 0.75 90.45, 0 72.75 M0 72.75 C0.42 59.62, -0.5 44.56, 0 24.25 M0 72.75 C-0.31 62, 0.39 50.58, 0 24.25 M0 24.25 C0.6 7.93, 7.62 0.36, 24.25 0 M0 24.25 C-1.19 5.9, 7.1 1.56, 24.25 0" stroke="#000000" stroke-width="1" fill="none"></path></g><g transform="translate(12 10) rotate(0 136.49166870117188 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">PyPyTypedMethodMetadata</text></g><g transform="translate(320 145.75) rotate(0 64.375 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">PyMethodDef</text></g><g transform="translate(323 209.48459595774125) rotate(0 40.358333587646484 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">ml_name</text></g><g stroke-linecap="round"><g transform="translate(413 174.625) rotate(0 -0.5 47.5)"><path d="M-0.24 0.21 C-0.58 16.29, -0.7 80.09, -0.86 96.07 M-1.83 -0.72 C-2.41 15.08, -1.7 78.66, -1.73 94.5" stroke="#000000" stroke-width="1" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(493.1505265640949 172.95269669828917) rotate(0 -0.5 47.5)"><path d="M-0.4 -0.76 C-0.62 15.2, -2.05 79.08, -2.04 94.86 M1.59 1.45 C1.79 17.69, 0.51 80.78, 0.13 96.33" stroke="#000000" stroke-width="1" fill="none"></path></g></g><mask></mask><g transform="translate(447 208.54647392583252) rotate(0 8.225000381469727 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">...</text></g><g transform="translate(526.7749996185303 209.17188861377167) rotate(0 8.225000381469727 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">...</text></g><g stroke-linecap="round" transform="translate(10 44.04372656020746) rotate(0 142 48.5)"><path d="M24.25 0 C81.4 0.59, 139.51 0.48, 259.75 0 M24.25 0 C76.66 2.05, 129.56 2.09, 259.75 0 M259.75 0 C277.04 -1.2, 285.14 8.12, 284 24.25 M259.75 0 C273.68 2.24, 284.4 8.52, 284 24.25 M284 24.25 C283.59 42.07, 286.16 59.02, 284 72.75 M284 24.25 C282.68 42.25, 282.83 60.97, 284 72.75 M284 72.75 C282.47 88.38, 276.34 95.42, 259.75 97 M284 72.75 C285.66 90.19, 275.08 97.5, 259.75 97 M259.75 97 C187.17 95.96, 117.77 97.99, 24.25 97 M259.75 97 C188.95 94.81, 118.66 94.88, 24.25 97 M24.25 97 C6.43 97.47, 0.64 89.78, 0 72.75 M24.25 97 C8.65 99.11, -2.04 90.05, 0 72.75 M0 72.75 C2.2 55.17, 2.17 40.39, 0 24.25 M0 72.75 C0.06 56.12, -0.19 39.47, 0 24.25 M0 24.25 C-1.99 8.14, 8.97 1.8, 24.25 0 M0 24.25 C0.69 9.54, 7.32 -1.75, 24.25 0" stroke="#000000" stroke-width="1" fill="none"></path></g><g transform="translate(111.5 80.11448964279407) rotate(0 40.358333587646484 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">ml_name</text></g><g stroke-linecap="round"><g transform="translate(101.5 45.80954529040514) rotate(0 -0.5 47.5)"><path d="M0.43 -0.92 C0.3 14.64, 0.02 77.99, -0.26 94.12 M-0.81 1.21 C-1.07 16.84, -0.59 79.74, -0.82 95.19" stroke="#000000" stroke-width="1" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(201.6505265640949 44.13724198869454) rotate(0 -0.5 47.5)"><path d="M0.74 -0.88 C0.46 15.25, -1 79.72, -1.27 95.85 M-0.33 1.27 C-0.83 17.15, -2.16 78.71, -2.36 94.17" stroke="#000000" stroke-width="1" fill="none"></path></g></g><mask></mask><g transform="translate(55.5 79.45168744587136) rotate(0 8.225000381469727 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">...</text></g><g transform="translate(235.27499961853027 79.78308854433271) rotate(0 8.225000381469727 12.5)"><text x="0" y="0" font-family="Virgil, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="start" style="white-space: pre;" direction="ltr" dominant-baseline="text-before-edge">...</text></g><g stroke-linecap="round"><g transform="translate(314.88948191038185 202.0536137670765) rotate(0 -70.98591049402219 -42.714306883538256)"><path d="M-0.15 -0.22 C-23.58 -14.53, -117.3 -71.12, -140.82 -85.14 M-1.69 -1.38 C-25.15 -16.14, -117.84 -72.99, -141.16 -87.12" stroke="#000000" stroke-width="1" fill="none"></path></g><g transform="translate(314.88948191038185 202.0536137670765) rotate(0 -70.98591049402219 -42.714306883538256)"><path d="M-116.65 -82.17 C-123.91 -81.67, -129.93 -83.74, -141.16 -87.12 M-116.65 -82.17 C-126.68 -84.38, -135.51 -86.11, -141.16 -87.12" stroke="#000000" stroke-width="1" fill="none"></path></g><g transform="translate(314.88948191038185 202.0536137670765) rotate(0 -70.98591049402219 -42.714306883538256)"><path d="M-125.57 -67.58 C-130.42 -71.14, -133.94 -77.31, -141.16 -87.12 M-125.57 -67.58 C-132.19 -75.38, -137.56 -82.77, -141.16 -87.12" stroke="#000000" stroke-width="1" fill="none"></path></g></g><mask></mask></svg>

I started off with a mock implementation of this in C (no Python C API, just
fake structures to sketch it out) and it worked. So I implemented a hacky
version of it in Cinder, but never shipped it because my integration with
Cinder was a little too hacky.

A year later, I decided to poke Carl Friedrich and see if we could implement it
in PyPy.

## Implementing in PyPy

<!-- TODO(max): A sketch of the implementation inside PyPy, or at least a link
to the commit(s) -->

## Where do all the C extensions come from?

None in PyPy standard library since they are all implemented in Python

### Cython

In this snippet of Cython code, we make a function that adds two machine
integers.

```cython
cpdef int add(int a, int b):
    return a + b
```

Cython will generate a very fast C function that adds two machine integers.
Calls to this from Cython are type checked at compile time and will be as fast
as your C compiler allows:

```c
static int add(int __pyx_v_a, int __pyx_v_b) {
  return __pyx_v_a + __pyx_v_b;
}
```

Since we used `cpdef` instead of `cdef`, Cython will also generate a wrapper C
extension function so that this function can be called from Python.

This means that the generated Cython code looks like (a much uglier version of)
below. **You don't need to understand or really even read the big blob** of
cleaned-up generated code below. You just need to say "ooh" and "aah" and "wow,
so many if-statements and so much allocation and so many function calls."


<!-- NOTE: this is worse, even, since it's unwrapping fastcall too -->

```c
static PyObject *add_and_box(CYTHON_UNUSED PyObject *__pyx_self,
                             int __pyx_v_a,
                             int __pyx_v_b) {
  int result = add(__pyx_v_a, __pyx_v_b, 0);
  // Check if an error occurred (unnecessary in this case)
  if (result == ((int)-1) && PyErr_Occurred()) {
    return NULL;
  }
  // Wrap result in PyObject*
  return PyLong_FromLong(result);
}

static PyObject *add_python(PyObject *__pyx_self,
                            PyObject *const *__pyx_args,
                            Py_ssize_t __pyx_nargs,
                            PyObject *__pyx_kwds) {
  // Check how many arguments were given
  PyObject* values[2] = {0,0};
  if (__pyx_nargs == 2) {
    values[1] = __pyx_args[1];
    values[0] = __pyx_args[0];
  } else if (__pyx_nargs == 1) {
    values[0] = __pyx_args[0];
  }
  // Check if any keyword arguments were given
  Py_ssize_t kw_args = __Pyx_NumKwargs_FASTCALL(__pyx_kwds);
  // Match up mix of position/keyword args to parameters
  if (__pyx_nargs == 0) {
    // ...
  } else if (__pyx_nargs == 1) {
    // ...
  } else if (__pyx_nargs == 2) {
    // ...
  } else {
    // ...
  }
  // Unwrap PyObject* args into C int
  int __pyx_v_a = PyLong_AsLong(values[0]);
  // Check for error (unnecessary if we know it's an int)
  if ((__pyx_v_a == (int)-1) && PyErr_Occurred()) {
    return NULL;
  }
  int __pyx_v_b = PyLong_AsLong(values[1]);
  // Check for error (unnecessary if we know it's an int)
  if ((__pyx_v_b == (int)-1) && PyErr_Occurred()) {
    return NULL;
  }
  // Call generated C implementation of add
  return add_and_box(__pyx_self, __pyx_v_a, __pyx_v_b);
}
```

Now, to be clear: this is probably the fastest thing possible for interfacing
with CPython. Cython has been worked on for years and years and it's *very*
fast. But we have some other runtimes that have different performance
characeteristics

### Other binding generators

pybind11, nanobind, ...

Even Argument Clinic in CPython

### Hand-written

## Small useless benchmark

Now let's try benchmarking the interpreter interaction with the native module
with a silly benchmark. It's a little silly because it's not super common (in
use cases I am familiar with anyway) to call C code in a hot loop like this
without writing the loop in C as well. But it'll be a good reference for the
maximum amount of performance we can win back.

```python
# bench.py
import mytypedmod


def main():
    i = 0
    while i < 10_000_000:
        i = mytypedmod.inc(i)
    return i


if __name__ == "__main__":
    print(main())
```

We'll try running it with CPython first because CPython doesn't have this
problem making `PyObject*`---that is just the default object representation in
the runtime.

```console
$ python3.10 setup.py build
$ time python3.10 bench.py
10000000
846.6ms
$
```

Okay so the output is a little fudged since I actually measured this with
`hyperfine`, but you get the idea. CPython takes a very respectable 850ms to go
back and forth with C *10 million times*.

Now let's see how PyPy does on time, since it's doing a lot more work at the
boundary.

```console
$ pypy3.10 setup.py build
$ time pypy3.10 bench.py
10000000
2.269s
$
```

Yeah, okay, so PyPy (before our changes) has to do a bunch of work with every
call to `inc` and this difference in timing makes it clear. But this post is
all about adding types. What if we add types to the C module?

```c
// TODO(max): Turn this into a diff?
#include <Python.h>

long inc_impl(long arg) {
  return arg+1;
}

PyObject* inc(PyObject* module, PyObject* obj) {
  (void)module;
  long obj_int = PyLong_AsLong(obj);
  if (obj_int == -1 && PyErr_Occurred()) {
    return NULL;
  }
  long result = inc_impl(obj_int);
  return PyLong_FromLong(result);
}

PyPyTypedMethodMetadata inc_sig = {
  .arg_type = T_C_LONG,
  .ret_type = T_C_LONG,
  .underlying_func = inc_impl,
  .ml_name = "inc",
};

static PyMethodDef mytypedmod_methods[] = {
    // TODO(max): Add METH_FASTCALL | METH_TYPED
    {inc_sig.ml_name, inc, METH_O | METH_TYPED, "Add one to an int"},
    {NULL, NULL, 0, NULL}};

static struct PyModuleDef mytypedmod_definition = {
    PyModuleDef_HEAD_INIT, "mytypedmod",
    "A C extension module with type information exposed.", -1,
    mytypedmod_methods,
    NULL,
    NULL,
    NULL,
    NULL};

PyMODINIT_FUNC PyInit_mytypedmod(void) {
  PyObject* m = PyState_FindModule(&mytypedmod_definition);
  if (m != NULL) {
    Py_INCREF(m);
    return m;
  }
  return PyModule_Create(&mytypedmod_definition);
}
```

And now let's run it with our new patched PyPy.

```console
$ pypy3.10-patched setup.py build
$ time pypy3.10-patched bench.py
10000000
168.1ms
$
```

<!-- TODO(max): Compare with Skybison -->

I honestly did not believe my eyes when I saw this number. It's a greater than
10x performance improvement *and I think there is still room for more* (such as
calling that C function to get the metadata instead of doing that inside the
JIT).

This is extraordinarily promising.

As I said before, most applications don't consist of a Python program calling a
C function and only a C function in a tight loop. It would be important to
profile how this change affects a *representative* workload. That would help
motivate the inclusion of these type signatures in a binding generator such as
Cython.

## PyPy internals

PyPy is comprised of two main parts:

* A Python interpreter
* A tool to transform interpreters into JIT compilers

This means that instead of writing fancy JIT compiler changes to get this to
work, I wrote an interpreter change. Their `cpyext` (C API) handling code
already contains a little "interpreter" of sorts to make calls to C extensions.
It looks at `ml_flags` to distinguish between `METH_O` and `METH_FASTCALL`, for
example.

So I added a new case that looks like this pseudocode:

```diff
diff --git a/tmp/interp.py b/tmp/typed-interp.py
index 900fa9c..b973f13 100644
--- a/tmp/interp.py
+++ b/tmp/typed-interp.py
@@ -1,7 +1,17 @@
 def make_c_call(meth, args):
     if meth.ml_flags & METH_O:
         assert len(args) == 1
+        if meth.ml_flags & METH_TYPED:
+            return handle_meth_typed(meth, args[0])
         return handle_meth_o(meth, args[0])
     if meth.ml_flags & METH_FASTCALL:
         return handle_meth_fastcall(meth, args)
     # ...
+
+def handle_meth_typed(meth, arg):
+    sig = call_scary_c_function_to_find_sig(meth)
+    if isinstance(arg, int) and sig.arg_type == int and sig.ret_type == int:
+        unboxed_arg = convert_to_unboxed(arg)
+        unboxed_result = call_f_func(sig.underlying_func, unboxed_arg)
+        return convert_to_boxed(unboxed_result)
+    # ...
```

Since the JIT probably already knows about the types of the arguments to the C
function (and probably has also unboxed them), all of the intermediate work can
be elided. This makes for much less work!

To checkout the changes to PyPy, look at [this stack of
commits](https://github.com/pypy/pypy/compare/3a06bbe5755a1ee05879b29e1717dcbe230fdbf8...branches/py3.10-mb-typed-sig-experiments).

To see what this looks like from the JIT's perspective, we can peek at the
generated IR before and after my change.

<!-- TODO(max): Add PyPy traces before/after -->

## Next steps

* Get other native types (other int types, double) working
* Get multiple parameters working
* Hack a proof of concept of this idea into Cython
* Make the signature more expressive
  * Perhaps we should have a mini language kind of like CPython's Argument
    Clinic