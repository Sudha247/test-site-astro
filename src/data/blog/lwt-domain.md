---
author: Sudha Parimala
pubDatetime: 2020-10-05T15:22:00Z
title: Introducing parallelism in Lwt with Multicore OCaml
slug: introducing-parallelism-in-lwt
featured: true
draft: false
tags:
  - ocaml
description:
  Lwt meets Multicore OCaml.
---


[Lwt](https://ocsigen.org/lwt/) is OCaml's widely used concurrency
library. It offers powerful primitives for concurrent programming which have
been used in many systems for effective I/O parallelism.
Recently, we have been doing some experiments to add support for CPU
parallelism in Lwt with [Multicore OCaml](https://github.com/ocaml-multicore/ocaml-multicore).
This post will showcase some ways to speed up Lwt applications with Multicore
OCaml.

### Lwt_preemptive

`Lwt_preemptive` module has the facility for preemptive scheduling, unlike rest
of Lwt which operates in a cooperative manner.
[`Lwt_preemptive`](https://ocsigen.org/lwt/5.2.0/api/Lwt_preemptive) runs every task in a new [systhread](https://caml.inria.fr/pub/docs/manual-ocaml/libref/Thread.html).
All the systhreads run concurrently, but need to obtain the master runtime lock in order to execute OCaml code. Hence, the OCaml parts of the program do not run any faster.

We bring in parallelism via the `Lwt_preemptive` module.
In the Multicore version illustrated below, every new task will run on a
separate [domain](https://github.com/ocaml-multicore/ocaml-multicore/blob/parallel_minor_gc/stdlib/domain.mli),
in parallel with all other tasks running at that point. This gives us an easy
way to run tasks in multiple cores. For best results, number of
domains spawned should be equal to the number of cores available. Number of
domains can be controlled via the `Lwt_preemptive.set_bounds` method. The
[API docs](https://github.com/Sudha247/lwt-multicore/blob/master/src/unix/lwt_preemptive.mli) have more details.

We shall go through an example to see how we could use `Lwt_preemptive`
to speedup an application.

### Installation

To start with, Multicore OCaml needs to be installed. This can be done with
multicore opam. Installation instructions can be found [here](https://github.com/ocaml-multicore/multicore-opam#install-multicore-ocaml). If your
code uses `ppx`, it is recommended to use the
`4.10.0+multicore+no-effect-syntax` compiler variant to maintain compatibility.

After installing the compiler, Lwt with multicore support can be installed by
pinning the repository -

```
opam pin add lwt https://github.com/Sudha247/lwt-multicore.git
```

### Example

Let us take an example of a simple Lwt server that accepts integer requests and
returns its fibonacci number. A sequential version of this server is
[here](https://github.com/Sudha247/code-samples/blob/master/lwt-server/fib.ml).

With the help of `Lwt_preemptive`, we could process multiple requests at a time.
To do this, the parent domain keeps accepting requests. Once a request is
received, the actual computation and writing response is delegated to a
detached task that gets executed on another domain.

```ocaml
let detached oc = (* run computation in detached domain *)
  Lwt_preemptive.detach (fun msg -> compute msg |> send_res oc)

let rec main oc ic =
  match%lwt recv ic with (* accept request *)
  | Some msg ->
    incr counter;
    if !counter mod num_domains = 0 then begin
      compute msg |> send_res oc; (* run in parent domain *)
      main oc ic
      end
    else
      detached oc msg >>= (* delegate to detached domain *)
      fun () -> main oc ic
  | None -> return_unit
```

We also occassionally perform the computation in the parent domain. This is to
ensure that computations are (almost) equally distributed amongst available
cores. Full code of parallel server is available [here](https://github.com/Sudha247/code-samples/blob/master/lwt-server/fibp.ml).

### Performance

The parallel fibonacci server was benchmarked on a Intel Xeon Gold 5120, with 
10 clients and 10 requests per client. Each request was the value `45` and 
server had to run a non-tail recursive `fib 45` for every request. 
Performance numbers are below, the client's code is available
[here](https://github.com/Sudha247/code-samples/blob/master/lwt-server/client.ml).

**Systhreads version** - `4.10.0` compiler

Time taken by systhreads version: `15m1.059s`

**Multicore version** - `4.10.0+multicore+no-effect-syntax` compiler

| Cores | Time       |
|-------|------------|
| 1     | 14m52.014s |
| 2     | 7m26.023s  |
| 3     | 5m21.133s  |
| 4     | 4m9.774s   |
| 5     | 4m9.769s   |
| 6     | 4m0.854s   |
| 7     | 3m51.936s  |
| 8     | 3m43.021s  |

We can observe quite some speedup as the number of cores increase.

---

This is an attempt to introduce parallelism in Lwt in a backwards compatible
manner. It is still an experimental feature. If you find any bugs or have any
questions, feel free to use the [issue tracker](https://github.com/Sudha247/lwt-multicore/issues) or get in touch. In case any existing code breaks with the
multicore version of Lwt, please let us know.

If you managed to speed up any of your applications with the help of
`Lwt_preemptive` (or Multicore OCaml in general), we'd be happy to have it as a
parallel benchmark in our [benchmarking suite Sandmark](https://github.com/ocaml-bench/sandmark).
Consider submitting a PR or open an issue if you need any help with integrating
your benchmark.  