# Rust gRPC Server

We'll be expending our repo to get a gRPC server binary created. In the next chapter we'll use Bazel to build this into a docker container.

We'll put the server in a new directory, `$HOME/repo/src/services/summation`. Again, Bazel doesn't care how we
arrange our repository, at this point we've come up with this package structure:
```
src/proto/summation
src/summation
src/services/summation
```

The src/summation directory looks a little weird. We could pretty easily move it to something like `src/lib/summation`.
We could also move things aroung to have:
```
src/summation/proto
src/summation/lib
src/summation/services
```

One feature of a monorepo where all the dependencies are self-contained is it's easier to move things after the fact. In
general it's easier to make breaking and backwards incompatible changes in a monorepo. We won't do that here and we'll
keep things as is.

## Exposing tokio crate
We'll use tokio to run our server. If you look in `$HOME/repo/third_party/rust/remote` you'll see that `cargo raze`
has already pulled down tokio because it's a transitive dependency for other things. If you look at ``$HOME/repo/third_party/rust/BUILD.bazel` you'll see `cargo raze` made that BUILD file and exposes third-party crates using the `alias` rule. This is what exposes these crates under the `//third_party/rust` path, and cargo-raze only exposes the dependencies we explictly list in `$HOME/repo/third_party/rust/Cargo.toml`.

Let's update `[dependencies]` of `$HOME/repo/third_party/rust/Cargo.toml` to include tokio:
```toml
[dependencies]
clap = { version = "4.2.2", features = ["derive"] }
log = "0.4.17"
prost = "0.11.6"
tonic = { version = "0.9.1", features = ["tls", "tls-roots", "default"] }
tonic-build = "0.9.1"
tokio = "1.27"
```

Then rerun cargo raze:
```shell
cd $HOME/repo/third_party/rust
cargo raze
```

We didn't remove the lock file because we're not expecting and don't want this step to change the versions of
any of our third party crates.

## Creating the gRPC server
Now Let's start making `$HOME/repo/src/services/summation/main.rs` with:
```rust
use src_proto_summation::summation_server::SummationServer;
use std::env;
use std::net::{IpAddr, Ipv4Addr, SocketAddr};
use tonic::transport::Server;

mod my_summation;
use my_summation::MySummation;

#[tokio::main(flavor = "current_thread")]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let port = env::var("PORT")
        .map(|p| p.parse::<u16>())
        .unwrap_or(Ok(50051))?;
    let addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(0, 0, 0, 0)), port);
    let summation = MySummation::new();

    Server::builder()
        .add_service(SummationServer::new(summation))
        .serve(addr)
        .await?;

    Ok(())
}

```

Then create `$HOME/repo/src/services/summation/my_summation.rs`
```rust
use src_proto_summation::summation_server::Summation;
use src_proto_summation::ComputeSumF64Request;
use src_proto_summation::ComputeSumF64Response;
use src_summation::f64::summation_f64;
use tonic::{Request, Response, Status};

pub struct MySummation {}

impl MySummation {
    pub fn new() -> Self {
        MySummation {}
    }
}

#[tonic::async_trait]
impl Summation for MySummation {
    async fn compute_sum_f64(
        &self,
        request: Request<ComputeSumF64Request>,
    ) -> Result<Response<ComputeSumF64Response>, Status> {
        let request = request.into_inner();
        let sum = summation_f64(&request.value);
        Ok(Response::new(ComputeSumF64Response { sum }))
    }
}
```

And finally make `$HOME/repo/src/services/summation/BUILD`:
```python
load("@rules_rust//rust:defs.bzl", "rust_binary")

rust_binary(
    name = "server",
    srcs = [
        "main.rs",
        "my_summation.rs",
    ],
    deps = [
        "//src/proto/summation:src_proto_summation",
        "//src/summation:src_summation",
        "//third_party/rust:tokio",
        "//third_party/rust:tonic",
    ],
)
```

Now let's try to build with `bazel build //...`. Oops, it doesn't work because `//src/summation:src_summation` isn't visible to our new package.

## Updating visibility of //src/summation:src_summation
The [visibility](https://bazel.build/concepts/visibility#target-visibility) attribute on our targets controls
who can depend on a target. In `//src/summation/BUILD` we omitted visibility for the `src_summation` target, which
means it defaults to only being visible to targets in that same `BUILD` file. So our `//src/summation:executable` target
could depend on it, but `//src/services/summation` can't.

In a multi-owner repo where one team might own `//src/summation` and another team owns `//src/services/summation` this
helps the first team ensure they control who can depend on them. (Usually you'll have a code review process with [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) to ensure the `//src/summation` team reviews/approves any changes to visibility).

To make `//src/summation:src_summation` visible to `/src/services/summation` we'll add
`visibility = ["//src/services/summation:__pgk__"]` to the `src_summation` target in `$HOME/repo/src/summation/BUILD`:
```python
rust_library(
    name = "src_summation",
    srcs = [
        "lib.rs",
        "f64.rs",
        "u32.rs",
    ],
    deps = ["//third_party/rust:log"],
    visibility = ["//src/services/summation:__pkg__"],
)
```

## Build and test

Now when we run `bazel build //...` everything should build.

Next, lets run our server:
```shell
bazel run -c opt //src/services/summation:server
```

Finally, to test it we'll use `grpcurl`. If you're on the debian VM we built you can use the following to get the binary:
```shell
cd $HOME/repo
curl -L https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz -o grpcurl.tar.gz
tar -xzvf grpcurl.tar.gz grpcurl
```

And then run it:
```shell
cd $HOME/repo
./grpcurl -proto repo/src/proto/summation/summation.proto -plaintext -d '{"value": 5.0, "value": 2.0}' localhost:50051 src_proto_summation.Summation/ComputeSumF64
```

This should output:
```
{
  "sum": 7
}
```