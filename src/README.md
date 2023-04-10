# Using Bazel with Rust, C++, Docker and gRPC

> **NOTE**: Buck2 was recently released and part of the motivation for this doc
> is providing a simple example of using Bazel and then replicating it with Buck2
> to learn how the two compare.

I was asked for examples, pointers, or warnings from my experience using Bazel
for rust and decided to write up this doc as a response. My experience with
Bazel like build systems comes from my prior work in larger tech
organizations (including using Blaze at Google), and I've found it can feel
harder than it is to use these systems when you're not already building on top
of existing infrastucture.

I was suprised that when I started using Bazel for Rust I ran into less roadblocks
than I expected. While there weren't any roadblocks, there were definetly plenty of
speedbumps I hit and this doc will share how I dealt with them.

Bazel provides a lot of flexability, and I'll share the choices I made but they aren't
the only way or the right way to do things, just the way I did.

There are lots of docs that talk about the pros and cons of monorepos.
This isn't one of them. I avoid talking about why you might want to do this
and just talk about how I do it.

Inspired by the style of [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/), if you walk through the examples in this doc you'll see some of the speed-bumps hit and how to deal with them. If you want to skip this and just see the final working example, its available on GitHub at at todo!()