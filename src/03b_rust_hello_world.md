# Rust Hello World

First we'll create a "hello world" rust binary. Then we'll create the rust library and switch our rust binary to call it. We'll also create a unit test for our rust library.

# Hello world rust binary
Once we're done this chapter, this binary will print out a random number gint from 0 to a user provided arg. We'll start
with just making sure we can build a rust binary and print hello world!.

We're going to put all the summation related code in the directory `$HOME/repo/src/summation`, let's go ahead and make and cd
that:
```shell
mkdir -p "${HOME}/repo/src/summation"
cd "${HOME}/repo/src/summation"
```

## BUILD files
Bazel uses `BUILD` files to describe all the "targets" (things that get built) in a directory, and the rules for how to
build them. The rules_rust [rust_binary](http://bazelbuild.github.io/rules_rust/defs.html#rust_binary) rule provides options you can set to control how things are built.
> **WARNING**: Bazel aims for [hermetic builds](https://bazel.build/basics/hermeticity); code is built in a sandbox and
> configuration is stored in BUILD files, not environment variables. To pass rustc
> environment variables you set them in the BUILD file since setting environment variables won't pass
> through to the sandbox

Open up `$HOME/repo/src/summation/BUILD` in your favorite text editor and lets setup the build rules for our binary.
```python
# This tells bazel to load the rust_binary rule from the rules_rust package
load("@rules_rust//rust:defs.bzl", "rust_binary")

rust_binary(
    #We are going to call the target/binary summation
    name = "executable",
    #The list of src files it needs (just main.rs)
    srcs = ["main.rs"],
    #Any libraries/crates it depends on, for now we'll leave this blank
    deps = [],
    #The crate_root file, this would default to main.rs but we put it in for clarity
    crate_root = "main.rs",
)
```

Let's also create our `main.rs` file:
```rust
fn main() {
    println!("Hello world");
}
```

Now lets try to build it by running:
```shell
bazel build :executable
```

And it fails! If you get what I got you'll see something like:
```shell
$ bazel build :executable
### UPDATE OUTPUT
```

For most dependencies, you'll tell Bazel where to find them and it'll pull them down for you.
One exception is the C++ toolchain, which rules_rust depends on. To get around this we can install
the build-essential package on debian which includes gcc, g++, and libc.
```shell
sudo apt-get install build-essential
```

Once you've installed that let's try `bazel build :executable` again and see what happens
```shell
$ bazel build :executable
###UPDATE OUTPUT
```

This time it fails again saying we didn't set the edition. We could manually set the edition on the rule, but that's kind of annoying if we want to use the same edition across the repo, so lets open up our `$HOME/repo/WORKSPACE` file and specify
an edition on the `rust_register_toolchains()` call by changing it to:
```python
rust_register_toolchains(edition = "2021")
```

Hopefully the third time is a charm? Let's see what `bazel build :executable` does this time:
```shell
$ bazel build :executable
###UPDATE OUTPUT
```

Success! Now before we get up for a coffee break lets just make sure it actually runs. You can use the `bazel run`
subcommand to run the binary.
```shell
$ bazel run :executable
###UPDATE OUTPUT
```

It worked! One thing to note is this ran it inside bazel, and it output a bunch of bazel log messages. The second
to last line of the output says `INFO: Running command line: bazel-bin/src/summation/executable`.

What is that? Let's go to the repo directory and see:
```shell
$ ls -l $HOME/repo
total 28
-rw-r--r-- 1 parallels parallels  798 Apr 10 18:50 WORKSPACE
-rw-r--r-- 1 parallels parallels  782 Apr 10 18:13 WORKSPACE~
lrwxrwxrwx 1 parallels parallels  123 Apr 10 18:53 bazel-bin -> /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/execroot/__main__/bazel-out/k8-fastbuild/bin
lrwxrwxrwx 1 parallels parallels  106 Apr 10 18:53 bazel-out -> /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/execroot/__main__/bazel-out
lrwxrwxrwx 1 parallels parallels   96 Apr 10 18:53 bazel-repo -> /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/execroot/__main__
lrwxrwxrwx 1 parallels parallels  128 Apr 10 18:53 bazel-testlogs -> /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/execroot/__main__/bazel-out/k8-fastbuild/testlogs
drwxr-xr-x 3 parallels parallels 4096 Apr 10 18:23 src
```

You can see bazel created a bunch of symlinks to a mysterious `.cache/bazel` directory. When you run bazel, it caches build artifacts to avoid rebuilding things that didn't change, and these symlinks give us a way to access the artifacts bazel
produces. If we want, we can run the binary directly by running `$HOME/repo/bazel-bin/src/summation/executable`. Let's try that:
```shell
$ $HOME/repo/bazel-bin/src/summation/executable
Hello world
```
Now we see the output of the binary without any of the bazel messages because we are invoking it directly.

With this we've successfully configured bazel to compile rust and get a binary out. When we're back from our coffee break we'll add in a C++ library, make a rust crate that links to it, and make our Hello world program call the rust crate.
