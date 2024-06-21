---
title: Watching now
layout: page
---

Here are some projects that are occupying some brain space and that I am keeping an eye on, in no particular order (to demonstrate that, I shuffled before publishing). Some are because they do a lot with a small amount of code/people, some are because they feel like they are poking at something very new and cool, and some are excellent learning resources.

* [Porffor](https://github.com/CanadaHonk/porffor), a from-scratch AOT JS engine
* [Natalie](https://github.com/natalie-lang/natalie), an AOT Ruby compiler
* [HVM](https://github.com/HigherOrderCO/HVM), a massively parallel interaction combinator VM for the GPU
* [Cosmopolitan](https://github.com/jart/cosmopolitan), a C library that can build very portable self-contained binaries
* [Taskflow](https://github.com/taskflow/taskflow), a graph-based parallel task system
* [AtomVM](https://github.com/atomvm/AtomVM), a small BEAM (Erlang) VM for small devices
* [Pyret](https://github.com/brownplt/pyret-lang), a functional programming language with some excellent ideas and a great pedagogical focus
* [YJIT](https://github.com/Shopify/ruby), a JIT for Ruby based on Maxime's research on basic block versioning
* [Roc](https://github.com/roc-lang/roc), a fast, friendly programming language
* [Bun](https://github.com/oven-sh/bun/), a fast JS runtime based on JavaScriptCore
* [Alan](https://github.com/alantech/alan), an autoscalable programming language
* [Iroh](https://github.com/n0-computer/iroh), a toolkit for building distributed applications
* [Toit](https://github.com/toitlang/toit), a VM from the Strongtalk lineage and management software for fleets of ESP32s
* [monoruby](https://github.com/sisshiki1969/monoruby), a full Ruby VM and JIT by one person
* [Pydrofoil](https://github.com/pydrofoil/pydrofoil), a fast RISC-V emulator based on RPython, the PyPy internals
* [ir](https://github.com/dstogov/ir), the JIT internals for PHP
* [MatMul-free LLM](https://github.com/ridgerchu/matmulfreellm), an implementation of transformer-based large language models without matrix multiplication
* [bifrost](https://github.com/aperturerobotics/bifrost), a p2p library that supports multiple transports
* [gf](https://github.com/nakst/gf), a very usable GUI GDB frontend
* [LPython](https://github.com/lcompilers/lpython), a very early stages optimizing Python compiler
* [micrograd](https://github.com/karpathy/micrograd/), [ccml](https://github.com/t4minka/ccml), and [cccc](https://github.com/skeeto/cccc), small, educational autodiff libraries
* [arcan](https://github.com/letoram/arcan), a new display server and window system
* [MaPLe compiper](https://github.com/MPLLang/mpl), a compiler for automatic fork/join parallelism in SML
* [dulwich](https://github.com/jelmer/dulwich), a pure Python Git implementation
* [elfconv](https://github.com/yomaytk/elfconv), an ELF to Wasm AOT compiler
* [mold](https://github.com/rui314/mold/), a fast and parallel linker
* [wild](https://github.com/davidlattimore/wild), an incremental linker
* [Cake](https://github.com/thradams/cake), a C23 frontend and compiler to older C versions
* [plzoo](https://github.com/andrejbauer/plzoo), which has multiple different small PL implementations with different semantics
* [onramp](https://github.com/ludocode/onramp), a bootstrapping C compiler
* [simple-abstract-interpreter](https://github.com/sree314/simple-abstract-interpreter), just what it says on the tin
* [bril](https://github.com/sampsyo/bril), an educational compiler IR
* [weval](https://github.com/cfallin/weval), a WebAssembly partial evaluator
* [tinygrad](https://github.com/tinygrad/tinygrad), a small autodiff library with big plans
* [Riker](https://github.com/curtsinger-lab/riker), correct and fast incremental builds using system call tracing
* [try](https://github.com/binpash/try), to inspect a command's effects before modifying your live system
* [MicroHs](https://github.com/augustss/MicroHs), Haskell implemented with combinators in a small C++ core
* [container2wasm](https://github.com/ktock/container2wasm), to run containers in WebAssembly

Not quite code but presenting very cool ideas:

* Verifying your whole register allocator too hard? No problem, just [write a verifier for a given allocation](https://cfallin.org/blog/2021/03/15/cranelift-isel-3/) and abort if it fails. This also lends itself nicely to fuzzing for automatically exploring large program state spaces.
* [Copy and Patch](https://fredrikbk.com/publications/copy-and-patch.pdf) (PDF) compilation, which generates pretty fast code very quickly
* [Egg](https://egraphs-good.github.io/), and more broadly egraphs, for program IRs
* (To-be-written) Using Z3 to prove your static analyzer correct