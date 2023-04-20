# Pulling in external crates like Clap

We'll use [cargo raze](https://github.com/google/cargo-raze). There's a newer alternate way to pull crates.io crates
into bazel using [crate universe](http://bazelbuild.github.io/rules_rust/crate_universe.html) that we don't cover here.
With cargo raze you create a `Cargo.toml` file to specify the crates.io crates you will depend on and use the
`raze` extension to create Bazel rules for compiling each of these. The dependencies can be vendored, but we'll
use the non-vendored mode in the examples below.

If you're starting from our blank VM we'll need to install rust and cargo. Using [The Cargo Book](https://doc.rust-lang.org/cargo/getting-started/installation.html) instructions:
```shell
curl https://sh.rustup.rs -sSf | sh
```

Add cargo to the path:
```shell
source "$HOME/.cargo/env"
```

Next lets install `cargo raze`
```shell
cargo install cargo-raze
```

If you're using the VM we setup, that failed saying `The pkg-config command could not be found` and also complaining about open ssl. Let's fix that and try
again:
```shell
sudo apt install pkg-config libssl-dev
cargo install cargo-raze
```

Now that we have Cargo and cargo-raze, lets put our third-party rust dependencies under the `//third_party/rust` path in our repo. First let's make that
directory:
```shell
mkdir $HOME/repo/third_party
mkdir $HOME/repo/third_party/rust
```

Now lets make `$HOME/repo/third_party/rust/Cargo.toml` and follow the [instructions](https://github.com/google/cargo-raze#generate-a-cargotoml) for this:
```
[package]
name = "compile_with_bazel"
version = "0.0.0"

# Mandatory (or Cargo tooling is unhappy)
[lib]
path = "fake_lib.rs"

[dependencies]
log = "0.4.17"

[package.metadata.raze]
# The path at which to write output files.
#
# `cargo raze` will generate Bazel-compatible BUILD files into this path.
# This can either be a relative path (e.g. "foo/bar"), relative to this
# Cargo.toml file; or relative to the Bazel workspace root (e.g. "//foo/bar").
workspace_path = "//third_party/rust"

# This causes aliases for dependencies to be rendered in the BUILD
# file located next to this `Cargo.toml` file.
package_aliases_dir = "."

# The set of targets to generate BUILD rules for.
targets = [
    "x86_64-unknown-linux-gnu",
]

# The two acceptable options are "Remote" and "Vendored" which
# is used to indicate whether the user is using a non-vendored or
# vendored set of dependencies.
genmode = "Remote"

default_gen_buildrs = true
```

Now from the `$HOME/repo/third_party/rust` directory run cargo raze
```shell
cd $HOME/repo/third_party/rust
cargo raze
```

> For some reason on my VM cargo raze failed and looking at the [cargo-raze code](https://github.com/google/cargo-raze/blob/v0.16.1/impl/src/rendering/bazel.rs#L78) it seems to be because a dummy directory is missing. I
fixed this by running `mkdir -p "/tmp/cargo-raze/doesnt/exist/"` and then running `cargo raze` again.

This should create a few different files in that directory, and a remote directory.
The `$HOME/repo/third_party/rust/BUILD.bazel` file creates a new `:log` target which
allows you to depend on the log crate. We can add `//third_party/rust:log` to the `deps`
attribute of our `rust_library` and `rust_binary` rules to pull in the `log` crate.

We also need to update `$HOME/repo/WORKSPACE` to pull down the remote crates. Add this to the
bottom of `WORKSPACE`:
```python
### Cargo raze deps
###
load("//third_party/rust:crates.bzl", "raze_fetch_remote_crates")

# Note that this method's name depends on your gen_workspace_prefix setting.
# `raze` is the default prefix.
raze_fetch_remote_crates()
```

Let's add `log` to our library by editing `$HOME/repo/src/summation/BUILD` and updating
the rust_library deps to say:
```python
rust_library(
    name = "src_summation",
    srcs = [
        "lib.rs",
        "f64.rs",
        "u32.rs",
    ],
    deps = ["//third_party/rust:log"],
)
```

Now lets use `log` in `f64.rs` by changing the top of the file to this:
```rust
use log::trace;

pub fn summation_f64(values: &[f64]) -> f64 {
    trace!("summation_f64");
    values.iter().sum()
}
```

Then lets rebuild and see what happens:
```shell
$ bazel build //...
INFO: Analyzed 5 targets (5 packages loaded, 43 targets configured).
INFO: Found 5 targets...
INFO: Elapsed time: 2.326s, Critical Path: 1.91s
INFO: 14 processes: 5 internal, 9 linux-sandbox.
INFO: Build completed successfully, 14 total actions
```

You should see some output showing it's pulling down the third party crates and then everything compiles.

## Adding Clap
Now let's add clap. We'll see there's a gotcha we need to deal with for that crate due to Bazel's sandboxing.

First we'll add Clap along with the `derive` featur to `$HOME/repo/third_party/rust/Cargo.toml` under the `[dependencies]` section:
```
[dependencies]
log = "0.4.17"
clap = { version = "4.2.2", features = ["derive"] }
```

Then lets rerun `cargo raze` from `$HOME/repo/third_party/rust`
```shell
cd $HOME/repo/third_party/rust
cargo raze
```

You might see a warning about needing to run `cargo generate-lockfile`. We can delete `Cargo.raze.lock` and rerun
cargo raze to update versions of packages and create a new lockfile.
```shell
rm Cargo.raze.lock
cargo raze
```

Now lets go back to `$HOME/repo/src/summation/BUILD` and add `"//third_party/rust:log"` to our binary `deps`, resulting
in:
```python
rust_binary(
    #We are going to call the target/binary summation
    name = "executable",
    #The list of src files it needs (just main.rs)
    srcs = ["main.rs"],
    #Any libraries/crates it depends on, for now we'll leave this blank
    deps = [
        ":src_summation",
        "//third_party/rust:clap",
    ],
    #The crate_root file, this would default to main.rs but we put it in for clarity
    crate_root = "main.rs",
)
```

We'll try to build it before actually updating main.rs to use the code by running
```shell
bazel build //...
```

And it fails with output looking like:
```
INFO: Analyzed 5 targets (18 packages loaded, 806 targets configured).
INFO: Found 5 targets...
ERROR: /home/parallels/.cache/bazel/_bazel_parallels/8136e33dd0c038f4f223262d62801c45/external/raze__clap_builder__4_2_2/BUILD.bazel:34:13: Compiling Rust rlib clap_builder v4.2.2 (54 files) failed: (Exit 1): process_wrapper failed: error executing command (from target @raze__clap_builder__4_2_2//:clap_builder) bazel-out/k8-opt-exec-2B5CBBC6/bin/external/rules_rust/util/process_wrapper/process_wrapper --arg-file ... (remaining 57 arguments skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging
error: couldn't read external/raze__clap_builder__4_2_2/src/../README.md: No such file or directory (os error 2)
 --> external/raze__clap_builder__4_2_2/src/lib.rs:7:10
  |
7 | #![doc = include_str!("../README.md")]
  |          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: this error originates in the macro `include_str` (in Nightly builds, run with -Z macro-backtrace for more info)

error: aborting due to previous error

INFO: Elapsed time: 5.310s, Critical Path: 4.04s
INFO: 29 processes: 8 internal, 21 linux-sandbox.
FAILED: Build did NOT complete successfully
```

What's going on here? It's hard to believe the clap release from crates.io doesn't build, but that's what Bazel tells us.
If you look at the error, you see it's using the `include_str!` macro not being able to find `../README.md`. Back in the hello world chapter we mentioned Bazel tries to ensure [hermetic builds](https://bazel.build/basics/hermeticity) by compiling code in a sandbox. One goal of the sandbox is to ensure you can't depend on anything that you haven't told Bazel explictly about. By default, cargo raze tells Bazel to bring over all the `*.rs` files, but it doesn't specify the compile needs  `README.md`. We can set an option to tell it we need this by adding these lines to `$HOME/repo/third_party/rust/Cargo.toml`:
```
[package.metadata.raze.crates.clap.'*']
compile_data_attr = "glob([\"**/*.md\"])"

[package.metadata.raze.crates.clap_builder.'*']
compile_data_attr = "glob([\"**/*.md\"])"

[package.metadata.raze.crates.clap_derive.'*']
compile_data_attr = "glob([\"**/*.md\"])"
```

Then let's run cargo raze again to get it to regen the third party crate build files.
```shell
cargo raze
```

And try building again
```shell
bazel build //...
```

At this point it should have built and you should be able to run your executable:
```shell
bazel run //src/summation:executable -- 0.0 1.0 2.0
```

That should output `sum = 3`. For completeness let's open up `$HOME/repo/src/summation/main.rs` and use clap
to parse the args. The final code we'll end up there will be:
```rust
use clap::{Parser, Subcommand};
use src_summation::f64::summation_f64;
use src_summation::u32::summation_u32;

#[derive(Subcommand)]
enum Cmd {
    U32 { args: Vec<String> },
    F64 { args: Vec<String> },
}

#[derive(Parser)]
struct Arguments {
    #[command(subcommand)]
    cmd: Cmd,
}

fn main() {
    let args = Arguments::parse();
    match args.cmd {
        Cmd::U32 { args } => {
            let args: Vec<u32> = args.into_iter().map(|a| a.parse().unwrap()).collect();
            println!("sum = {}", summation_u32(&args))
        }
        Cmd::F64 { args } => {
            let args: Vec<f64> = args.into_iter().map(|a| a.parse().unwrap()).collect();
            println!("sum = {}", summation_f64(&args))
        }
    }
}
```

Which we can run and get the help usage for by running:
```shell
bazel run //src/summation:executable -- --help
```

Which shows us:
```
WARNING: Ignoring JAVA_HOME, because it must point to a JDK, not a JRE.
INFO: Analyzed target //src/summation:executable (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //src/summation:executable up-to-date:
  bazel-bin/src/summation/executable
INFO: Elapsed time: 0.104s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/src/summation/executable --help
Usage: executable <COMMAND>

Commands:
  u32
  f64
  help  Print this message or the help of the given subcommand(s)

Options:
  -h, --help  Print help

```
