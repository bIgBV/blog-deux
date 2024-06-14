+++
title = "My journey through different languages"
date = 2018-03-23

[taxonomies]
categories = ["programming","career"]
+++

I've been programming since close to six years now and over that time,
I learned and tried to learn a lot of languages. This is a brief history of the
languages which I stuck with over time.

I started learning python in my undergrad course and built some toy projects to
better understand it. Eventually it lead my to my first job as a full stack
developer. Since python was my first real language I learned, I didn't really
get statically typed languages. I naively thought the productivity I had in
languages such as python and javascript were not worth the benefits. But then
I came across typescript. It completely changed front end development for me.

At that time I was working heavily with the react+redux architecture and
typescript prevented a whole class of bugs. Things like not having to step
through deeply nested component trees to see where the wrong prop was passed
down saved a huge amount of development time. It also invariably made me
organize my projects better by putting some thought into the API's exposed.
I finally understood how amazing strongly typed dynamic languages were.

I wanted the same experience in the backend and I naturally turned to Go
because it seemed vastly more approachable than Rust. I had been following the
Rust project since before it hit 1.0. I read up, did some tutorials but never
a non-trivial project, Plus the Rust web development story wasn't the best at
that time. So I started building web services in Go, but quickly hit a wall
with tooling. There wasn't an idiomatic way to manage packages and to my
knowledge, the community is still split between different tools. Plus,
I personally felt the way to implement interfaces was not good. There was no
way to know if an interface was implemented for a struct just by looking at the
method implementations. You *have* to know what methods are declared in the
interface. After feeling like I was fighting the languages, I just wasn't
interested anymore. Sure Go has a really shallow learning curve, but it just
isn't that ergonomic a language.

I was still looking at Rust from the sidelines, but as a part of my grad
school, I took an OS class which heavily used C and assembly. Here I really
learned about managing memory and how awesome C is. It's a tiny language which
can be learned in the same or even lesser time as compared to Go. But it is far
more flexible. Sure it has its many many warts, but to me, C will always hold
a special place since it makes reasoning out what your code is doing so easy
(Plus it was my first experience with systems programming). You look at a piece
of code and you know what it does.

That's when I really started looking at Rust seriously, since it is just as low
level as C (for my use cases) plus, it has a ton of modern features. I've tried
picking up C++ many times, but it was very difficult to pick up. The lack of
a default build system and having a no package manager meant I had no idea how
to get started on a non-trivial C++ project. Rust with cargo made getting
started insanely simple and approachable to newcomers.

I am taking a distributed computing systems class and had to build a simulation
of a distributed system using vector clocks and Rust seemed to be the perfect
candidate for such a [project](https://github.com/bIgBV/rust-evc). And even though I was new to Rust, I had
a heavily multi-threaded project with a lot of inter-thread communication up
and running within a few hours! It was an amazing feeling when I the first time
I ran the program, it ran perfectly. Without a single issue. And this was with
niceties like config files and logging to different backends. As far as writing
my first multi-threaded program went, it couldn't have been simpler.

The idea that if it compiles, it runs, is awesome and I plan on really diving
into the deep end with Rust. I'm looking at the [ redox-os
](https://github.com/redox-os/redox) project lately to get myself acquainted
with the codebase to start contributing small things to it.  Here's to many
more years of building systems.
