---
author: Sudha Parimala
pubDatetime: 2025-06-16T19:02:00Z
title: Trying out Go by writing a program for Game of Life
slug: game-of-life-go
featured: true
draft: false
tags:
  - programming
  - go
description:
  Trying out go by writing a zero player game
---

There are bunch of languages I've written code in, but Go is not one of them. I decided to try it out. Usually when I want to learn a new language, I go through a tutorial, textbook or examples. For what it's worth, I've heard Go has this very nice bunch of examples to learn the language, called [gobyexample](https://gobyexample.com/). Though, this time, instead of going through any tutorials I decided to try to port a program I'm familiar with to Go. I've contributed to a [parallel OCaml version](https://github.com/ocaml-bench/sandmark/blob/main/benchmarks/multicore-numerical/game_of_life_multicore.ml) of Conway's Game of Life. I decided to take the sequential version in OCaml, and tried to port it to Go.

## Conway's Game of Life

Conway's Game of Life, created by the prolific mathematician John Conway is a zero player game. It starts with a set configuration of live cells, and the live cells grow or shrink through a set of rules. In our version, we start with a random configuration of a board size. Let's say we start with a board size of twenty - there are 20*20 (400) cells, each either dead or alive - and we go from there. Here are the rules, explained from [Wikipedia](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life):

1. Any live cell with fewer than two live neighbours dies, as if by underpopulation.
2. Any live cell with two or three live neighbours lives on to the next generation.
3. Any live cell with more than three live neighbours dies, as if by overpopulation.
4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

Find my Go version here: [code](https://gist.github.com/Sudha247/b8f5d3988cd91373214f347c74dfd248)

```
$ ./gol 20 20

...

. . █ █ █ █ . . . . . . . . █ █ . █ █ .
. █ . . . █ . . . . . . . . █ █ . █ █ .
. █ . █ . █ █ . . . . . . . . . . . . .
. . █ █ . █ █ . . . . . . . . . . . . .
. . . █ █ █ . . . . . . . . . . . . . .
. . . . . . . . . █ █ . . . . . . . . .
. . . . . . . █ █ . . █ . . . . . . . .
. . . . . . . █ █ . █ █ . . . . . █ █ █
. . . . █ █ . . . . █ . . . . . █ █ . █
. . . . █ █ . . . █ . . . . . █ . █ . █
. . . . . █ . █ █ . . . . . . █ . █ . █
. . . . . . . . █ . . . . . . . . . . █
. . . . . █ █ █ . . . . . . . . █ █ █ .
. . . . █ █ . █ . . . . . . . . . . . .
. . . . . █ . . █ █ . . . . . . . . . .
. . . . . . . █ █ █ . . . . . . . . . .
. . . . . . . . █ . . . . . . . . . . .
. . . . █ █ █ █ █ . . . . . . . . . . .
. . . . █ . . . █ . . . . . . . . . . .
. . . . . . . . . . . . . . . . . . . .
```

## Experience writing the Go version

Here, I'm going to try to document my experience of porting the program to Go. First off, the tooling support is really great! I primarily use VSCode as my text editor. I had Go compiler installed on my system already, which I had installed to use tools like `hugo`. I went on to install the official Go VSCode extension, and it just worked out of the box. The extension not only detected errors on the fly, it also had suggestions for more modern and idiomatic syntaxes. Furthermore, it went on to include any packages I was using on my program automatically, and it's quite impressive.

## First impressions

Most of my programming experience is in OCaml, C and Java. Coming more from declative way of thinking, a few things suprised me. I don't mean to put one on a pedastal over the other, end of the day it's just a matter of perspective, and getting things done with your tool of choice. That said, I'll try to list down some of the features of Go which was not something I had encountered. This is by no means an exhaustive list of comparison between Go and OCaml/C etc.

### Dynamically sized array

I tried to declare a variable sized array depending on a command line argument. Something like this --

```go
var board2[board_size][board_size] int
```

But it turns out Arrays in Go are fixed-size at compile time, and hence cannot create arrays whose size is not known at compile time. The error message was `invalid array length board_size`, which was a bit confusing in the beginning but with a little digging I could workaround this by using slices.

### No try-catch

There isn't try-catch blocks in Go, unlike what one could be used to in Java/C or even OCaml. Go uses a different mechanism for error handling, by treating errors as values. For instance, when you want to convert a command-line argument from its default `string` type to an `int`, error handline could be done like below. This pattern in fact, is similar to the [Result](https://ocaml.org/manual/5.3/api/Result.html) mechanism in OCaml.

```go
board_size, err := strconv.Atoi(args[1])

if err != nil {
	fmt.Println("Board size must be an integer:", err)
	return
}
```

### Structure of the code

Like most languages, a Go program begins executing at the `main` function. Functions are decalred with the `func` keyword, and return types are explicitly tagged to a function. Go seems to have type inference for variables, but not usually for functions. With Go generics, the situation seems to have changed but I'm yet to dig into generics part.

### Go Language Server

Like I mentioned earlier, the language server is quite impressive. It started operating after a one click installation. The features that were useful to me in writing this small program are many; For starters, it highlights any compile time error on the fly while you write code. It includes any packages that we use in the code automatically to the import list. Not only that, it also told me a more modern way to use `for` loops. Overall, it gives an IDE experience in VSCode. How it handles debugging is something I'd like explore at a later point, that was not really necessary for this program.

---

My first impression is that Go seems like a pragmatic language that gets things done. It also appears relatively simple to get started with. Even though some of its principles differ from what I'm used to, it makes me want to dig deeper into the design choices behind the language. Next, I want to try writing the parallel version in Go.