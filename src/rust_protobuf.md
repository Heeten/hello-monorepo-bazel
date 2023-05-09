# Building Rust protobuf

Generating code for protobufs in rust is a lot more complicated than other languages. We'll cover how to do both.

We'll start with other language support since that includes pulling down the protoc compiler which we'll also
need for rust.

## Creating the protofile
Lets decide where we want to put our protobuf file. Bazel doesn't care where we put it,
so I'm going to arbitarly pick `//src/proto/summation` as the path for it.

Let's put to protobuf file in `$HOME/repo/src/proto/summation/summation.proto` with these contents
```
syntax = "proto3";

package src_proto_summation;

service Summation {
  rpc ComputeSumF64(ComputeSumF64Request) returns (ComputeSumF64Response);
}

message ComputeSumF64Request {
  repeated double value = 1;
}

message ComputeSumF64Response {
  double sum = 1;
}
```

Our package name above is unconventional, we are using underscores instead of dots. This is to match
the crate naming convention we adopted for rust libraries.

## Adding rules_proto for protobuf generation in protoc supported languages

We're going to use the bazel rules from [rules_proto](https://github.com/bazelbuild/rules_proto).
Let's pull this down by adding this section to our `$HOME/repo/WORKSPACE` file:
```python
### rules_proto
### Release info from https://github.com/bazelbuild/rules_proto/releases
http_archive(
    name = "rules_proto",
    sha256 = "dc3fb206a2cb3441b485eb1e423165b231235a1ea9b031b4433cf7bc1fa460dd",
    strip_prefix = "rules_proto-5.3.0-21.7",
    urls = [
        "https://github.com/bazelbuild/rules_proto/archive/refs/tags/5.3.0-21.7.tar.gz",
    ],
)
load("@rules_proto//proto:repositories.bzl", "rules_proto_dependencies", "rules_proto_toolchains")
rules_proto_dependencies()
rules_proto_toolchains()
```

Next lets make `$HOME/repo/src/proto/summation/BUILD`:
```python
load("@rules_proto//proto:defs.bzl", "proto_library")

proto_library(
    name = "proto",
    srcs = [
        "summation.proto",
    ],
    visibility = ["//visibility:public"],
)
```

Finally run `bazel build //...` to build everything.

## Protobuf generation in Rust
We're going to use [tonic](https://github.com/hyperium/tonic) for our gRPC server and use tonic_build to generate the protobuf and gRPC code.

### Pulling in tonic, tonic_build, and prost
To generate and compile the protobuf and gRPC code we'll need to explictly expose the tonic, tonic_build, and prost crates.

When we pull down tonic we'll enable the tls features, even though we won't use them (yet) in this guide. We'll also need
to tell bazel that the compile depends on `*.md` files for prost, and some other files for other transitive dependencies (which
we don't explictly pull down but it an implicit dependency pulled down).

So we'll add the following under `[dependencies]`:
```toml
prost = "0.11.6"
tonic = { version = "0.9.1", features = ["tls", "tls-roots", "default"] }
tonic-build = "0.9.1"
```

And these to the bottom of the file:
```toml
[package.metadata.raze.crates.prost.'*']
compile_data_attr = "glob([\"**/*.md\"])"

[package.metadata.raze.crates.rustls-webpki.'*']
compile_data_attr = "glob([\"**/*.der\"])"

[package.metadata.raze.crates.ring.'*']
compile_data_attr = "glob([\"**/*.der\"])"

[package.metadata.raze.crates.axum.'*']
compile_data_attr = "glob([\"**/*.md\"])"
```

Our full `$HOME/repo/third_party/rust/Cargo.toml` will look like:
```toml
[package]
name = "compile_with_bazel"
version = "0.0.0"

# Mandatory (or Cargo tooling is unhappy)
[lib]
path = "fake_lib.rs"

[dependencies]
clap = { version = "4.2.2", features = ["derive"] }
log = "0.4.17"
prost = "0.11.6"
tonic = { version = "0.9.1", features = ["tls", "tls-roots", "default"] }
tonic-build = "0.9.1"

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

[package.metadata.raze.crates.clap.'*']
compile_data_attr = "glob([\"**/*.md\"])"

[package.metadata.raze.crates.clap_builder.'*']
compile_data_attr = "glob([\"**/*.md\"])"

[package.metadata.raze.crates.clap_derive.'*']
compile_data_attr = "glob([\"**/*.md\"])"

[package.metadata.raze.crates.prost.'*']
compile_data_attr = "glob([\"**/*.md\"])"

[package.metadata.raze.crates.rustls-webpki.'*']
compile_data_attr = "glob([\"**/*.der\"])"

[package.metadata.raze.crates.ring.'*']
compile_data_attr = "glob([\"**/*.der\"])"

[package.metadata.raze.crates.axum.'*']
compile_data_attr = "glob([\"**/*.md\"])"
```

Now lets delete the lock file and run cargo raze:
```shell
cd $HOME/repo/third_party/rust/
rm Cargo.raze.lock
cargo raze
```

And finally run `bazel build //...` to make sure we haven't broken anything yet.

### Using tonic to generate protobuf and gRPC code

To get a rust protobuf/gRPC library we need to run two steps. The first is running the
tonic_build generator, which we can think of as having a `build.rs` or cargo build script
that we run first. We'll use rules_rust [cargo_build_script] to do this.

First lets make the `$HOME/repo/src/proto/summation/build.rs`:
```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("./summation.proto")?;
    Ok(())
}
```

Next lets update `$HOME/repo/src/proto/summation/BUILD` to use it:
```python
load("@rules_proto//proto:defs.bzl", "proto_library")
load("@rules_rust//cargo:cargo_build_script.bzl", "cargo_build_script")

proto_library(
    name = "proto",
    srcs = [
        "summation.proto",
    ],
    visibility = ["//visibility:public"],
)

cargo_build_script(
    name = "generate_rust_proto",
    srcs = [
        "build.rs",
    ],
    deps = [
        "//third_party/rust:tonic_build",
    ],
    build_script_env = {
        "RUSTFMT": "$(execpath @rules_rust//:rustfmt)",
        "PROTOC": "$(execpath @com_google_protobuf//:protoc)"
    },
    data = [
        "summation.proto",
        "@rules_rust//:rustfmt",
        "@com_google_protobuf//:protoc",
    ],
)
```

We've added a load line for `cargo_build_script` and then invoked that rule to run the generator. There's a lot going on here. One thing to note is Bazel uses different attributes to convey different [types of dependencies](https://bazel.build/concepts/dependencies#types-of-dependencies). We've seen `srcs` and `deps` already, `data` is kind of a catch-all used when
things don't fit in `srcs` or `deps`. How these attributes are used varied based on the rule, so it's useful to check the
docs of the rule you're using.

In this case we're telling bazel that if the protofil, rustfmt, or protoc change we need to rerun the build script using the `data` attribute, and we're also telling it to expose those in the sandbox it runs the compile in.

When we run the build script, we also need to set the environment variables `RUSTFMT` and `PROTOC` so that tonic_build knows where to find those, which is what the `build_script_env` attribute does. The `@rules_rust` is us pointing to the
external `rules_rust` bazel workspace we're depending on in our `WORKSPACE` file. The `srcs` attribute points to our `build.rs` file (which we could have named something else like generate_rust_proto.rs if we wanted.

### Exposing in a rust library
The prior step just runs the build script. We need to add a `rust_library` target that includes it.

To do this we'll add this to `$HOME/repo/src/proto/summation/BUILD`
```python
load("@rules_rust//rust:defs.bzl", "rust_library")

rust_library(
    name = "src_proto_summation",
    srcs = [
        "lib.rs",
    ],
    deps = [
        ":generate_rust_proto",
        "//third_party/rust:prost",
        "//third_party/rust:tonic",
    ],
    visibility = ["//visibility:public"],
)
```

And we'll create `$HOME/repo/src/proto/summation/lib.rs` with
```rust
tonic::include_proto!("src_proto_summation");
```

Finally lets run `bazel build //...` and make sure everything builds!

There's a decent amount of boilerplate for creating the proto. If I was doing it a lot, I would make my own bazel rule
that does all this for me.

### Examining the generated rust file
Above we're generating the `src_proto_summation.rs` file. You can find the generated file in your $HOME/repo/bazel-out directory. The exact path will vary slightly, for me it's in `$HOME/repo/bazel-out/k8-fastbuild/bin/src/proto/summation/generate_rust_proto.out_dir/src_proto_summation.rs`
