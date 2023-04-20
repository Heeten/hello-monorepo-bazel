# Using Bazel with Rust, gRPC, protobuf, and Docker

> **NOTE**: Buck2 was recently released and part of the motivation for this doc
> is providing a simple example of using Bazel and then replicating it with Buck2
> to learn how the two compare.

This doc walks through creating a rust library (crate), rust binary, unit test, docker image, and Google Cloud run service
using Bazel. When I started using Bazel for Rust that I ran into less roadblocks
than I expected, but there were lots of speedbumps and I hope this walk-through helps with that.

Bazel provides lots of flexability and in this doc I make certain choices on conventions for directory structure and
crate naming that have worked for me, but they aren't the only way to do things and encourage you to choose what
feels best to you.

This doc won't cover when or why to use monorepos. There's a lot written on that. If cargo is working great for you,
you probably don't need Bazel or Buck2. If you think you might want to explore that, this doc shows you how to get
a functional setup in place.

If you find any issues with this book please open a bug report at https://github.com/Heeten/hello-monorepo-bazel/issues