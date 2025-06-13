---
author: Sudha Parimala
pubDatetime: 2023-01-12T10:25:00Z
title: My Experience at Lambda Retreat
slug: lambda-retreat
featured: true
draft: false
tags:
  - functional-programming
description:
  Working thorough SICP with a bunch of programmers, at a forest.
---

I spent a week in the woods with fellow programmers at the [Lambda Retreat](https://anandology.com/lambda-retreat/). It was a wonderful way to explore the nature of computations, abrstractions, and paradigms. Although I mostly work in OCaml, it was fun and challenging to code in Scheme, another functional programming language.

_Structure and Interpretation of Computer Programs (SICP)_ is many programmers' favourite programming textbook. It teaches programming constructs like recursion, modularity, abstractions, etc. For a long time, it was used as the textbook for an introduction to programming course. Here's what [Peter Norvig](https://www.amazon.com/review/R403HR4VL71K8) and [Eli Bendersky](https://eli.thegreenplace.net/2008/05/28/book-review-structure-and-interpretation-of-computer-programs-by-harold-abelson-gerald-jay-sussman/) have to say about SICP.

Having seen a lot of people highly recommend SICP, I grabbed a copy for myself a few years ago and started reading it, but, alas, I never completed the book.

In the latter part of last year, [Anand](https://anandology.com/) decided to host a week-long retreat to gather a bunch of people and go through some interesting parts of SICP. Suffice it to say, I jumped at the opportunity to do nothing but read and write code for a week.

### Getting Ready

Two weeks before the retreat, we had some warm-up sessions to get ourselves ready. During this time, we attended some remote sessions and solved a few exercises from the Chapters 1 & 2 of SICP.

### Arriving

On Day 0, we all arrived at Bangalore from various parts of India, and carpooled to the [Tamarind Valley Collective](https://tvc.farm/) (TVC), located ~80km from the city. Reaching TVC turned out to be an unexpected but enjoyable 1.5km trek, since the roads to the campsite were unusable due to rain.

### At the Retreat

**Functional Geometry**

The retreat began with Functional Geometry from the second chapter of SICP. We started with the basics, like rendering images, and slowly built the primitives needed for generating Escher's woodcut.

![fish](/images/fish.jpg)
![Escher's Woodcut](/images/woodcut.jpg)

It was amazing to see the power of composability! We thoroughly enjoyed building Escher's woodcut from an unassuming image of a fish.

Anand rewarded everyone with an Escher's woodcut T-Shirt for successfully generating the `square-limit` \o/

We then abstracted out the implementation details for the Functional Geometry primitives we had built. The abstraction gives us the freedom to change the implementation at a later point without affecting the higher-level details.

**Mutability and State**

We spent some time understanding mutations, global state, and local state in Scheme, a functional language like OCaml. This led us to building some mutable data structures, like a mutable queue and a mutable hash table in Scheme, and generalising with a dispatcher to perform operations.

**Metacircular Interpreter**

Another exercise before the retreat was to write a parser for Scheme in Python. At the retreat, we started with translating it to Scheme. Going further, we built a metacircular interpreter -- a Scheme interpreter written in Scheme. How cool is that?

We then learnt about lazy evaluation in Scheme and went on to make our metacircular interpreter lazy by default. Another interesting part we looked forward to was targeting WebAssembly (Wasm) from Scheme. It was surprisingly simple to go from Scheme to Wasm, targeting Wasm's Lisp-like syntax.

### Beyond Tech

Living at TVC in the middle of a forest with barely any electricity or cellular network for a week was a humbling experience. Madhav, the resident manager at TVC, and his team made sure our stay was comfortable. The food, made from locally-sourced indgredients featuring local cuising was amazing!

The evening walks and hikes at TVC were memorable. We managed to sight some kingfishers and owls while snacking on some freshly plucked tamarinds. We had so much fun hiking along a stream that runs in the middle of TVC and capturing wisdom about sustainable living from Madhav and Vikrant, who put it into practice by living on farms.

![Group](/images/poco-dog.jpg)

Our days of hacking were followed by board games in evenings and nights. We had so much fun and laughter riots playing games like Skull, Chameleon, and Ticket to Ride Europe. By the end of the retreat, we were surprised by how little internet and social media we had consumed that week!

![Ticket to ride game](/images/ticket-to-ride.jpg)

I'm grateful to have had the opportunity to attend the first ever Lambda Retreat and hope to carry the functional programming spirit forward. It was super nice meeting all the enthusiastic and kind people at the retreat, and I hope to see everyone again at future events.

Thanks to Anand for organising it and to everyone who attended for making it an enjoyable experience. Thanks also to Madhav and his team at TVC for ensuring our stay was comfortable.