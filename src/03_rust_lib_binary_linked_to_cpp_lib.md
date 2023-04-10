# Rust and C++ depending on each other
For demonstration purposes, in this chapter we'll make a C++ library that a rust library depends on and wraps,
and then we'll make the rust binary depend on the rust library.

It's also possible to make a C++ binary depend on a rust library, to do that you would use the `rust_static_library` rule instead of `rust_library`, but we don't yet provide an example of doing that.

We'll also use Bazel's [visibility](https://bazel.build/concepts/visibility) concept to restrict other monorepo packages from depending on anything but the Rust library.

# Making the C++ library
We'll make a overly simplistic randint library for example purposes.

Let's make a header file by creating `$HOME/repo/src/randint/cc_lib.h`
```cpp
#ifndef __RANDINT_CC_LIB_H__
#define __RANDINT_CC_LIB_H__

#include <stdint.h>

uint32_t randint(uint32_t i);

#endif
```

Then lets implement the function by `$HOME/repo/src/randint/cc_lib.cc`
```cpp
#include "src/randint/cc_lib.h"
#include <cstdlib>

uint32_t randint(uint32_t i) {
  //Yes we aren't seeding with srand, this is just for example purposes
  return std::rand() * i;
}
```

Finally let's make a build target for it by adding this to `$HOME/repo/src/randint/BUILD`
```python
cc_library(
    name = "cc_lib",
    srcs = ["cc_lib.cc"],
    hdrs = ["cc_lib.h"],
)
```

Now lets build it. Before we ran `bazel build :randint` to specify the specific target we wanted to build.
We're going to switch to running `bazel build //...`, which will build all the targets (that need to be rebuilt)
in the repo. Sometimes this is handy when you want to make sure you didn't break the build for any downstream
consumers of the code you're changing.
```shell
bazel build //...
```

# Making the Rust library
Now we want to make the rust library (crate) that calls our C++ library. One thing we'll need to think about is what we
want the crate name to be. In the C++ code above you can see that the header file location is based on the path to the
file in the repo. If we had Java files, we'd also see this there, where package names are nested and based on their
path.

To make something that resembles this for Rust crate names in the monorepo I've decided to name my crates based
on the path to them. I also have them all have the same top-level prefix to ensure I avoid classes with third-party
crates. This also makes it easy for me to see a crate name in a source file (like in a use statement) and know
where to find it in the monorepo.

So I'll add the following rule to `$HOME/repo/src/randint/BUILD` to build the rust library.
```python
rust_library(
    name = "src_randint",
    srcs = ["lib.rs"],
    deps = [":cc_lib"],
    visibility = ["//visibility:public"],
)
```

The `deps` tag says this library depends on the `cc_lib` target in this package.

If I wanted I could set the `crate_name` setting to override the crate name, but if I don't it'll default to what `name` is, so `src_randint` in this case. I could also have set `crate_root` like I did for the rust_binary rule before, but if I don't it'll default to `lib.rs` for a library target.

We'll also need to tell Bazel to load the `rust_library` rule by updating the top of `BUILD` to have `load("@rules_rust//rust:defs.bzl", "rust_binary", "rust_library")`
The full BUILD file will now look like
```python
# This tells bazel to load the rust_binary and rust_library rule from the rules_rust package
load("@rules_rust//rust:defs.bzl", "rust_binary", "rust_library")

rust_binary(
    #We are going to call the target/binary randint
    name = "randint",
    #The list of src files it needs (just main.rs)
    srcs = ["main.rs"],
    #Any libraries/crates it depends on, for now we'll leave this blank
    deps = [],
    #The crate_root file, this would default to main.rs but we put it in for clarity
    crate_root = "main.rs",
)

cc_library(
    name = "cc_lib",
    srcs = ["cc_lib.cc"],
    hdrs = ["cc_lib.h"],
)

rust_library(
    name = "src_randint",
    srcs = ["lib.rs"],
    deps = [":cc_lib"],
    visibility = ["//visibility:public"],
)

```

Now lets try to build it to make sure we didn't break anything:
```shell
$ bazel build //...
INFO: Analyzed 3 targets (1 packages loaded, 7 targets configured).
INFO: Found 3 targets...
INFO: Elapsed time: 0.188s, Critical Path: 0.01s
INFO: 2 processes: 2 internal.
INFO: Build completed successfully, 2 total actions
```

Looks like it's working!

## Updating the CLI to use the library
In this step we'll change our CLI to use the randint rust library.

First, lets update the `BUILD` to depend on it by adding `:src_randint` to the deps.
```python
rust_binary(
    #We are going to call the target/binary randint
    name = "randint",
    #The list of src files it needs (just main.rs)
    srcs = ["main.rs"],
    #Any libraries/crates this binary depends on to compile
    deps = [":src_randint"],
    #The crate_root file, this would default to main.rs but we put it in for clarity
    crate_root = "main.rs",
)
```

Notice we don't need to specify `:cc_lib`, since that is pulled in transitively, we just need
to specify the dependencies we directly depend on to compile the source files.

Let's run `bazel build //...` now to make sure we didn't break anything
```shell
$ bazel build //...
INFO: Analyzed 3 targets (1 packages loaded, 7 targets configured).
INFO: Found 3 targets...
INFO: Elapsed time: 0.337s, Critical Path: 0.13s
INFO: 4 processes: 3 internal, 1 linux-sandbox.
INFO: Build completed successfully, 4 total actions
