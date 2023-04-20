# Rust library

Now that we have hello world let's do something more real by creating a library (crate), unit testing it, and making our binary call it.

## Creating the library
We're going to use multiple files for our trivial library to demonstrate how that is handled in Bazel.

One thing we'll need to think about is what we want the crate name to be. In the C++ code above you can see that the header file location is based on the path to the
file in the repo. If we had Java files, we'd also see this there, where package names are nested and based on their
path.

To make something that resembles this for Rust crate names in the monorepo I've decided to name my crates based
on the path to them. I also have them all have the same top-level prefix to ensure I avoid classes with third-party
crates. This also makes it easy for me to see a crate name in a source file (like in a use statement) and know
where to find it in the monorepo.

Having settled the naming delima let's add the library to `$HOME/repo/src/summation/BUILD` by adding these lines to the file (the name attribute is what the crate name defaults to):
```python
load("@rules_rust//rust:defs.bzl", "rust_library")
rust_library(
    name = "src_summation",
    srcs = [
        "lib.rs",
        "f64.rs",
        "u32.rs",
    ],
    deps = [],
)
```

We have to list all the files we want to compile against here. Otherwise Bazel won't copy them into the sandbox where our library is compiled.
I like listing all the files explictly, but if you want to include all "*.rs" files Bazel provides a [glob()](https://bazel.build/reference/be/functions#glob) to do this.

Now let's make `$HOME/repo/src/summation/lib.rs`:
```rust
pub mod f64;
pub mod u32;
```

If we don't have the `mod` lines, when bazel runs rustc it'll ignore the f64.rs and u32.rs files since rustc uses the crate root source file to figure out what to compile.
Including them in the `BUILD` file gets them copied over to the sandbox `rustc` is run in, adding them to `lib.rs` gets rustc to compile them.

And lets make `$HOME/repo/src/summation/f64.rs`:
```rust
pub fn summation_f64(values: &[f64]) -> f64 {
    values.iter().sum()
}
```

Wow, that's a boring function. Clearly I picked a simple example. Let's make a boring `$HOME/repo/src/summation/u32.rs` file as well:
```rust
pub fn summation_u32(values: &[u32]) -> u32 {
    values.iter().sum()
}
```

Now let's build it. From anywhere in the workspace we can build this using it's full path:
```shell
bazel build //src/summation:src_summation
```

The `//` maps to the root of the workspace which is `$HOME/repo` in our example, `//src/summation` says we are talking about that path from the workspace root,
and then `src_summation` is the target inside the build file that we are trying to build. If we're already in `$HOME/repo/src/summation` we can omit the path
and just use `bazel build :src_summation` for short.

We can also run `bazel build :all` to build all the targets in the directory we are in. This should be a no-op if you manually built the executable and lib already
since none of the source files have changed so bazel just uses the cached build and doesn't need to remake them.
