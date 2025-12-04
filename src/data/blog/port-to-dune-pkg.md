---
author: Sudha Parimala
pubDatetime: 2025-11-23T14:25:00Z
title: Porting an OxCaml project to Dune Package Management
slug: oxcaml-dune-pkg
featured: true
draft: false
tags:
  - functional-programming
description:
  Migrating an OxCaml project to build with Dune pkg.
---

As part of the Dune development team, we've been working to strengthen Dune’s
package management and smooth out the remaining rough edges. For the
uninitiated, [Dune Package
Management](https://dune.readthedocs.io/en/stable/explanation/package-management.html)
adds the capability to manage OCaml project dependencies to OCaml's build tool
[Dune](https://github.com/ocaml/dune/). As we move towards making it more
robust, we are also in the process of battle-testing it in various projects.

[OxCaml](https://oxcaml.org/) is a compiler branch developed by Jane Street with
fast-moving compiler extensions that they would eventually like to upstream to the
mainstream compiler. Jane Street uses this compiler branch internally, and it is
also being used in external projects. Recently at ICFP 2025, there was a
[tutorial on OxCaml](https://github.com/oxcaml/tutorial-icfp25). This was
suggested as a potential testing project for OxCaml with Dune package management.

I took up the task of porting this tutorial to build with Dune Package
Management (`dune pkg`). Before I started, I was of the opinion that this was
going to be a straightforward change of a few lines, and ta da, everything would
work (famous last words...). But this task ended up requiring a number of
patches, and I decided to write this blog post to summarise our findings.


### Hello OxCaml

Some of my co-workers had tried out a `hello-oxcaml` project, trying to build a
minimal OxCaml project with `dune pkg`:
https://github.com/gridbugs/hello-oxcaml. I thought it made sense to first
validate this still works. I had to make some changes as both OxCaml and `dune
pkg` are fast-moving projects: https://github.com/gridbugs/hello-oxcaml/pull/2.
This worked, however there was an issue. The compiler build was super slow to a
point I started wondering whether my machine had frozen. Looking at running
processes, it was still running, albeit on a single core, which was strange. I
discussed this with [@art-w](https://github.com/art-w), and he mentioned having
come across the same issue in the past. This was happening due to nested
invocations of Dune, and Dune failing to pick up concurrency. More details are
[here](https://github.com/ocaml/dune/issues/12737).

It did finish building, and the `hello-oxcaml` project worked as expected. Next
step was to try to build the OxCaml tutorial.

### Building the project

I found the dependencies of the project from its opam file and added them to the
`dune-project`. While Dune Package Management can infer dependencies from the
opam package, having the dependencies in `dune-project` proves to be an easier
way to tweak dependencies as needed.

```dune
(package
 (name oxcaml-tutorial)
 (depends
   (ocaml-variants (= 5.2.0+ox))
   base
   parallel))
```

Next, I tried to build it. And I hit the next roadblock: there was a build
failure when building the `ocamlbuild` package.


```
File "dune.lock/ocamlbuild.pkg", line 23, characters 7-74:
23 |   (url https://github.com/ocaml/ocamlbuild/archive/refs/tags/0.15.0.tar.gz)
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Error: Error trying to read targets after a rule was run:
- checksum/sha512=c8311a9a78491bf759eb27153d6ba4692d27cd935759a145f96a8ba8f3c2e97cef54e7d654ed1c2c07c74f60482a4fef5224e26d0f04450e69cdcb9418c762d3/dir/examples/07-dependent-projects/libdemo: Unexpected file kind "S_DIR" (directory)
```

Looking at the error message, one thing was clear to me: it was not picking up
the Dune version of `ocamlbuild`, which is needed by Dune package managemet.
There are some patches on top of `ocamlbuild` in the Dune overlays. But it was
picking up the OxCaml version of `ocamlbuild`, presumably also having its own
set of patches. Now, this means we have two different forks of `ocamlbuild`. We
need to combine the patches into a single branch in order for this to work with
both OxCaml and Dune package managemet. Furthermore, I pretended this to be the
OxCaml version, as there were hard dependencies on the specific version of the
package. But this pretend version does build. It's still to be seen what happens
upstream.

Now the project was generating lockfiles, which means we had the dependencies
set. We could go ahead and install them and build the project. Then came the next
roadblock: we had a build failure in one of the dependencies.

```
File "dune.lock/odoc-parser.pkg", line 15, characters 3-82:
15 |    git+https://github.com/oxcaml/odoc.git#97e1daecb432d33a7137d525f7a554f203073a95)))
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Error: Error trying to read targets after a rule was run:
- url/ebca61d809da743073794a02009c3602/dir/test/generators/html/fonts: Unexpected file kind "S_DIR" (directory)
```

Okay, we already know this failure! This shows there are symlinks in
`odoc-parser`. Dune Package Management doesn't support symlinks yet, and they
need to be removed. Checking the source code, there were indeed symlinks, but
somehow this didn't cause a problem in the mainline `odoc` and friends. Anyway,
I created a branch after removing the symlinks and copying the files, and added
a pin to this branch. This worked! Talking to the `odoc` developers, most of the
symlinks can be removed, and that's in progress.

The project builds!

### Developer tools

Next, we wanted to install developer tools for the project built with OxCaml.
There are forks of some developer tools to make them work with the OxCaml
compiler. This has to be made known to `dune pkg`. I ordered the opam
repositories `oxcaml`, `overlays`, `upstream` in that order, with `oxcaml`
having the highest priority. I assumed that with this ordering the solver would
pick up the versions of developer tools present in `oxcaml` by default, but that
turned out not to be the case. It picked up the latest versions from upstream,
as the project constraints in `dune-project` don't apply by default to developer
tools. For this, we need to specify constraints for the developer tools in
`dune-workspace`. That would look something like:

```dune
(lock_dir
 (path "dev-tools.locks/ocaml-lsp-server")
 (pins ocamlbuild odoc-parser)
 (constraints
   (ocaml-lsp-server (= 1.19.0+ox))
   (ocamlformat (= 0.26.2+ox))
   (ocaml-variants (= 5.2.0+ox)))
 (repositories overlay oxcaml upstream))
```

With this introduced, one also needs to add a path entry to the main
`dune.lock/` lock directory.

```dune
(lock_dir
 (path "dune.lock")
 (repositories :standard oxcaml))
```

And that gets the dev tools to build and work! Full changes can be found here:
https://github.com/Sudha247/tutorial-icfp25/pull/1/files. While all of this is
true at the time of writing this post, it might have changed if you're reading
it in the future.

Working through these issues gave me a much clearer sense of how OxCaml and Dune
Package Management fit together in practice. The process surfaced unexpected
interactions, but also showed how quickly things improve with the right patches
and discussions. Hopefully this post helps others navigating the same path, and
I look forward to seeing these workflows become even smoother in the future. If
you have any questions, do poke us at https://github.com/ocaml/dune/issues.
