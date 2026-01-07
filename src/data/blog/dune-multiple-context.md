---
author: Sudha Parimala
pubDatetime: 2026-01-07T14:25:00Z
title: Power of Contexts in Dune
slug: context-dune
featured: true
draft: false
tags:
  - functional-programming
  - ocaml
  - dune
description:
  Using Dune contexts to isolate builds workflows.
---

I've been working with Dune for almost a year now, and yet there are new
features I discover from time to time. Owing to the sheer complexity of the
software system, even people who've been working on Dune for a while are not
entirely sure of all parts of it. Likewise, I recently discovered the power of
the context stanza in Dune.

If you have used Dune to build OCaml projects, you would have noticed in your
build directory, `_build/default/...` The `default` here actually refers to the
`default` context. A “context” in Dune is essentially a named build
configuration/environment that Dune uses when compiling (for example, which
compiler/switch/toolchain it should use). If no `context` is explicitly specified
(which is mostly the case), it is assumed to be the default context. However,
you can define your custom context for the build, which comes in handy
sometimes.

Let's take an example of
[ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html). For the
uninitiated, ThreadSanitizer is a tool to detect data races in your code. OCaml
has supported ThreadSanitizer since 5.2. Let's build a project with
ThreadSanitizer.

`tsan_check.ml`:
```ocaml
let x = ref 0
let n = 5_000_000

let inc () =
  for _ = 1 to n do
    x := !x + 1
  done

let () =
  let d = Domain.spawn inc in
  inc ();
  Domain.join d;

  let expected = 2 * n in
  Printf.printf "x=%d expected=%d\n%!" !x expected;

  (* This will typically fail because increments get lost. *)
  assert (!x = expected)
```

The OCaml code is going to be the same — we're going to employ a few different
ways to build this and, in the process, see how contexts are useful.

### Build with `opam`

To build with `opam`, you would create an `opam` switch that supports
ThreadSanitizer. For setting up the switch, see instructions
[here](https://ocaml.org/docs/multicore-transition). Once you have a TSan
switch, you just do

`dune`:
```dune
(executable
 (name tsan_check)
 (modes native))
```

Now if you run:

```bash
$ opam switch 5.3.0+tsan && eval $(opam env)
$ dune exec -- ./tsan_check.exe
```

It shows you something like this:

```bash
WARNING: ThreadSanitizer: data race (pid=1662846)
  Write of size 8 at 0x7fa28ecffa58 by main thread (mutexes: write M78):
    #0 camlDune__exe__Tsan_check.inc_272 <null> (tsan_check.exe+0x483e2)
    #1 camlDune__exe__Tsan_check.entry <null> (tsan_check.exe+0x484df)
    #2 caml_program <null> (tsan_check.exe+0x460a9)
    #3 caml_start_program <null> (tsan_check.exe+0xdca47)
    #4 caml_startup_common runtime/startup_nat.c:132 (tsan_check.exe+0xdc1f0)
    #5 caml_startup_common runtime/startup_nat.c:88 (tsan_check.exe+0xdc1f0)
    #6 caml_startup_exn runtime/startup_nat.c:139 (tsan_check.exe+0xdc29b)
    #7 caml_startup runtime/startup_nat.c:144 (tsan_check.exe+0xdc29b)
    #8 caml_main runtime/startup_nat.c:151 (tsan_check.exe+0xdc29b)
    #9 main runtime/main.c:37 (tsan_check.exe+0x45bf9)

  Previous write of size 8 at 0x7fa28ecffa58 by thread T1 (mutexes: write M83):
    #0 camlDune__exe__Tsan_check.inc_272 <null> (tsan_check.exe+0x483e2)
    #1 camlStdlib__Domain.body_735 <null> (tsan_check.exe+0x7b8ef)
    #2 caml_start_program <null> (tsan_check.exe+0xdca47)
    #3 caml_callback_exn runtime/callback.c:201 (tsan_check.exe+0x9ee53)
    #4 domain_thread_func runtime/domain.c:1215 (tsan_check.exe+0xa32d6)

  Mutex M78 (0x561082d9e728) created at:
    #0 pthread_mutex_init ../../../../src/libsanitizer/tsan/tsan_interceptors_posix.cpp:1227 (libtsan.so.0+0x4bee1)
    #1 caml_plat_mutex_init runtime/platform.c:57 (tsan_check.exe+0xc9e6a)
    #2 caml_init_domains runtime/domain.c:943 (tsan_check.exe+0xa301e)
    #3 caml_init_gc runtime/gc_ctrl.c:353 (tsan_check.exe+0xaff83)
    #4 caml_startup_common runtime/startup_nat.c:111 (tsan_check.exe+0xdc0d7)
    #5 caml_startup_common runtime/startup_nat.c:88 (tsan_check.exe+0xdc0d7)
    #6 caml_startup_exn runtime/startup_nat.c:139 (tsan_check.exe+0xdc29b)
    #7 caml_startup runtime/startup_nat.c:144 (tsan_check.exe+0xdc29b)
    #8 caml_main runtime/startup_nat.c:151 (tsan_check.exe+0xdc29b)
    #9 main runtime/main.c:37 (tsan_check.exe+0x45bf9)

  Mutex M83 (0x561082d9e840) created at:
    #0 pthread_mutex_init ../../../../src/libsanitizer/tsan/tsan_interceptors_posix.cpp:1227 (libtsan.so.0+0x4bee1)
    #1 caml_plat_mutex_init runtime/platform.c:57 (tsan_check.exe+0xc9e6a)
    #2 caml_init_domains runtime/domain.c:943 (tsan_check.exe+0xa301e)
    #3 caml_init_gc runtime/gc_ctrl.c:353 (tsan_check.exe+0xaff83)
    #4 caml_startup_common runtime/startup_nat.c:111 (tsan_check.exe+0xdc0d7)
    #5 caml_startup_common runtime/startup_nat.c:88 (tsan_check.exe+0xdc0d7)
    #6 caml_startup_exn runtime/startup_nat.c:139 (tsan_check.exe+0xdc29b)
    #7 caml_startup runtime/startup_nat.c:144 (tsan_check.exe+0xdc29b)
    #8 caml_main runtime/startup_nat.c:151 (tsan_check.exe+0xdc29b)
    #9 main runtime/main.c:37 (tsan_check.exe+0x45bf9)

  Thread T1 (tid=1662848, running) created by main thread at:
    #0 pthread_create ../../../../src/libsanitizer/tsan/tsan_interceptors_posix.cpp:969 (libtsan.so.0+0x605b8)
    #1 caml_domain_spawn runtime/domain.c:1265 (tsan_check.exe+0xa499a)
    #2 caml_c_call <null> (tsan_check.exe+0xdc92b)
    #3 camlStdlib__Domain.spawn_730 <null> (tsan_check.exe+0x7b806)
    #4 camlDune__exe__Tsan_check.entry <null> (tsan_check.exe+0x484d1)
    #5 caml_program <null> (tsan_check.exe+0x460a9)
    #6 caml_start_program <null> (tsan_check.exe+0xdca47)
    #7 caml_startup_common runtime/startup_nat.c:132 (tsan_check.exe+0xdc1f0)
    #8 caml_startup_common runtime/startup_nat.c:88 (tsan_check.exe+0xdc1f0)
    #9 caml_startup_exn runtime/startup_nat.c:139 (tsan_check.exe+0xdc29b)
    #10 caml_startup runtime/startup_nat.c:144 (tsan_check.exe+0xdc29b)
    #11 caml_main runtime/startup_nat.c:151 (tsan_check.exe+0xdc29b)
    #12 main runtime/main.c:37 (tsan_check.exe+0x45bf9)

SUMMARY: ThreadSanitizer: data race (/home/sudha/ocaml/work/testing/hello-tsan/_build/default/tsan_check.exe+0x483e2) in camlDune__exe__Tsan_check.inc_272
==================
x=5014663 expected=10000000
Fatal error: exception Assert_failure("tsan_check.ml", 19, 2)
ThreadSanitizer: reported 1 warnings
```

Which shows that TSan is reporting the data race it is expected to report. Fair
enough. More often than not, you only require TSan as a testing case, and it is
not something you'd use to deploy your applications. This is where `context`
comes into play. What if we could add a separate context for compiling with TSan
that doesn't affect your normal workflow? We're going to do just that next.

### Context with opam

Now we're adding a new context to build with the `5.3.0+tsan` opam switch.

`dune-workspace`:
```
(lang dune 3.21)

(context default)

(context
 (opam
  (switch 5.3.0+tsan)
  (name tsan)
  (host default)))
```

We can build both contexts separately. If you do `dune build
_build/default/tsan_check.exe`, it will build with your current switch. On the
other hand, if you do `dune build _build/tsan/tsan_check.exe`, it will build
with the TSan switch no matter your current opam switch.

## Dune Package Management

One might wonder whether it is possible to replicate this build with Dune
package management. Turns out it is, and we'll see how. For those unfamiliar,
Dune package management doesn't have a concept of opam switches. The
dependencies are usually listed in the `dune-project` file. The TSan dependency
is a bit out of the ordinary because we need it only for the TSan build and not
the stock compiler build. So, we list it as an optional dependency.

`dune-project`:
```
(lang dune 3.21)

(package
 (name tsan-check)
 (allow_empty)
 (depends
  ocamlfind
  (ocaml
   (= 5.3.0)))
  (depopts ocaml-option-tsan))
```

In order to separate out the build with the stock OCaml compiler and with TSan
enabled, we define a separate `lock_dir` stanza for it. Not only that, we create
a separate context for TSan as we did before for an opam switch, but this time
we attach it to its `lock_dir` to convey what dependencies it needs to build.

`dune-workspace`
```
(lang dune 3.21)

(lock_dir
  (path dune.lock))

(lock_dir
  (path dune-tsan.lock)
  (depopts ocaml-option-tsan))

(context
 (default
  (name default)
  (lock_dir dune.lock)))

(context
  (default
   (name tsan)
   (lock_dir dune-tsan.lock)))
```

Now to build and execute it,

```
$ dune pkg lock dune.lock dune-new.lock
$ dune build _build/tsan/tsan_check.exe _build/default/tsan_check.exe
$ _build/tsan/tsan_check.exe # Run with TSan compiler
$ _build/default/tsan_check.exe # Run with stock compiler
```

With this setup, you get the best of both worlds: a normal, “stock” build that
stays simple, and a dedicated TSan build that you can opt into when you need it.
By isolating the TSan requirements behind a separate `lock_dir` and context, you
avoid pulling it into your everyday dependency graph while still keeping the
workflow entirely within Dune. The end result is a clean, reproducible way to
run race-detection builds on demand, without having to constantly switch
environments or maintain a separate project setup.
