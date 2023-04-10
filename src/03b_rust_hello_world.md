# Rust Library and Binary Linked to C++ Library

First we'll create "hello world" rust binary. Then we'll create the C++ library, rust library, and flesh out the rust binary. We'll also create a unit test for our rust library.

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
> **WARNING**: [hermetic builds](https://bazel.build/basics/hermeticity) are a key idea in bazel.
> In practice for rust developers this means code is built in a sandbox environment and the expectation is any
> settings used from from the BUILD files, not from the environment. This means if you want to pass rustc
> environment variables they should be set in the BUILD file, setting them in your environment won't pass them
> through to the sandbox

Open up `$HOME/repo/src/summation/BUILD` in your favorit text editor and lets setup the build rules for our binary.
```python
# This tells bazel to load the rust_binary rule from the rules_rust package
load("@rules_rust//rust:defs.bzl", "rust_binary")

rust_binary(
    #We are going to call the target/binary summation
    name = "summation",
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
bazel build :summation
```

And it fails! If you get what I got you'll see something like:
```shell
$ bazel build :summation
Starting local Bazel server and connecting to it...
INFO: Repository local_config_cc instantiated at:
  /DEFAULT.WORKSPACE.SUFFIX:509:13: in <toplevel>
  /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/cc_configure.bzl:184:16: in cc_configure
Repository rule cc_autoconf defined at:
  /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/cc_configure.bzl:143:30: in <toplevel>
ERROR: An error occurred during the fetch of repository 'local_config_cc':
   Traceback (most recent call last):
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/cc_configure.bzl", line 125, column 33, in cc_autoconf_impl
                configure_unix_toolchain(repository_ctx, cpu_value, overriden_tools)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/unix_cc_configure.bzl", line 349, column 17, in configure_unix_toolchain
                cc = find_cc(repository_ctx, overriden_tools)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/unix_cc_configure.bzl", line 314, column 23, in find_cc
                cc = _find_generic(repository_ctx, "gcc", "CC", overriden_tools)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/unix_cc_configure.bzl", line 310, column 32, in _find_generic
                auto_configure_fail(msg)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/lib_cc_configure.bzl", line 112, column 9, in auto_configure_fail
                fail("\n%sAuto-Configuration Error:%s %s\n" % (red, no_color, msg))
Error in fail:
Auto-Configuration Error: Cannot find gcc or CC; either correct your path or set the CC environment variable
ERROR: /DEFAULT.WORKSPACE.SUFFIX:509:13: fetching cc_autoconf rule //external:local_config_cc: Traceback (most recent call last):
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/cc_configure.bzl", line 125, column 33, in cc_autoconf_impl
                configure_unix_toolchain(repository_ctx, cpu_value, overriden_tools)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/unix_cc_configure.bzl", line 349, column 17, in configure_unix_toolchain
                cc = find_cc(repository_ctx, overriden_tools)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/unix_cc_configure.bzl", line 314, column 23, in find_cc
                cc = _find_generic(repository_ctx, "gcc", "CC", overriden_tools)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/unix_cc_configure.bzl", line 310, column 32, in _find_generic
                auto_configure_fail(msg)
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/cpp/lib_cc_configure.bzl", line 112, column 9, in auto_configure_fail
                fail("\n%sAuto-Configuration Error:%s %s\n" % (red, no_color, msg))
Error in fail:
Auto-Configuration Error: Cannot find gcc or CC; either correct your path or set the CC environment variable
INFO: Repository rust_linux_x86_64__x86_64-unknown-linux-gnu__stable_tools instantiated at:
  /home/parallels/repo/WORKSPACE:18:25: in <toplevel>
  /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/rules_rust/rust/repositories.bzl:203:14: in rust_register_toolchains
  /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/bazel_tools/tools/build_defs/repo/utils.bzl:233:18: in maybe
  /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/rules_rust/rust/repositories.bzl:874:65: in rust_repository_set
  /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/rules_rust/rust/repositories.bzl:496:36: in rust_toolchain_repository
Repository rule rust_toolchain_tools_repository defined at:
  /home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/rules_rust/rust/repositories.bzl:333:50: in <toplevel>
ERROR: /home/parallels/repo/src/summation/BUILD:4:12: //src/summation:summation depends on @local_config_cc//:cc-compiler-k8 in repository @local_config_cc which failed to fetch. no such package '@local_config_cc//':
Auto-Configuration Error: Cannot find gcc or CC; either correct your path or set the CC environment variable
ERROR: Analysis of target '//src/summation:summation' failed; build aborted: Analysis failed
INFO: Elapsed time: 11.669s
INFO: 0 processes.
FAILED: Build did NOT complete successfully (92 packages loaded, 187 targets configured)
    Fetching https://static.rust-lang.org/dist/rustc-1.68.1-x86_64-unknown-linux-gnu.tar.gz

```

For most dependencies, you'll tell Bazel where to find them and it'll pull them down for you.
One exception is the C++ toolchain, which rules_rust depends on. To get around this we can install
the build-essential package on debian which includes gcc, g++, and libc.
```shell
sudo apt-get install build-essential
```

Once you've installed that let's try `bazel build :summation` again and see what happens
```shell
$ bazel build :summation
ERROR: /home/parallels/repo/src/summation/BUILD:4:12: in rust_binary rule //src/summation:summation:
Traceback (most recent call last):
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/rules_rust/rust/private/rust.bzl", line 351, column 34, in _rust_binary_impl
                edition = get_edition(ctx.attr, toolchain, ctx.label),
        File "/home/parallels/.cache/bazel/_bazel_parallels/db6a46b6510c6ee4dba1a9500854830b/external/rules_rust/rust/private/rust.bzl", line 125, column 13, in get_edition
                fail("Attribute `edition` is required for {}.".format(label))
Error in fail: Attribute `edition` is required for @//src/summation:summation.
ERROR: /home/parallels/repo/src/summation/BUILD:4:12: Analysis of target '//src/summation:summation' failed
ERROR: Analysis of target '//src/summation:summation' failed; build aborted:
INFO: Elapsed time: 12.699s
INFO: 0 processes.
FAILED: Build did NOT complete successfully (10 packages loaded, 325 targets configured)
```

This time it fails again saying we didn't set the edition. We could manually set the edition on the rule, but that's kind of annoying if we want to use the same edition across the repo, so lets open up our `$HOME/repo/WORKSPACE` file and specify
an edition on the `rust_register_toolchains()` call by changing it to:
```python
rust_register_toolchains(edition = "2021")
```

Hopefully the third time is a charm? Let's see what `bazel build :summation` does this time:
```shell
$ bazel build :summation
INFO: Analyzed target //src/summation:summation (1 packages loaded, 60 targets configured).
INFO: Found 1 target...
Target //src/summation:summation up-to-date:
  bazel-bin/src/summation/summation
INFO: Elapsed time: 31.562s, Critical Path: 9.39s
INFO: 94 processes: 91 internal, 3 linux-sandbox.
INFO: Build completed successfully, 94 total actions
```

Success! Now before we get up for a coffee break lets just make sure it actually runs. You can use the `bazel run`
subcommand to run the binary.
```shell
$ bazel run :summation
INFO: Analyzed target //src/summation:summation (24 packages loaded, 172 targets configured).
INFO: Found 1 target...
Target //src/summation:summation up-to-date:
  bazel-bin/src/summation/summation
INFO: Elapsed time: 0.520s, Critical Path: 0.01s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/src/summation/summation
Hello world
```

It worked! One thing to note is this ran it inside bazel, and it output a bunch of bazel log messages. The second
to last line of the output says `INFO: Running command line: bazel-bin/src/summation/summation`.

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
produces. If we want, we can run the binary directly by running `$HOME/repo/bazel-bin/src/summation/summation`. Let's try that:
```shell
$ $HOME/repo/bazel-bin/src/summation/summation
Hello world
```
Now we see the output of the binary without any of the bazel messages because we are invoking it directly.

With this we've successfully configured bazel to compile rust and get a binary out. When we're back from our coffee break we'll add in a C++ library, make a rust crate that links to it, and make our Hello world program call the rust crate.
