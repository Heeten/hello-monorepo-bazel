# Docker Container

Getting our binary into a docker container requires getting [rules_docker](https://github.com/bazelbuild/rules_docker)
setup and then using the `rust_image` rule it provides to build the container.

## Workspace setup
Let's add the following to `$HOME/repo/WORKSPACE` to pull rules_docker and the rust_image related config into our WORKSPACE:
```python
### rules_docker setup
### FROM https://github.com/bazelbuild/rules_docker#setup
###
http_archive(
    name = "io_bazel_rules_docker",
    sha256 = "b1e80761a8a8243d03ebca8845e9cc1ba6c82ce7c5179ce2b295cd36f7e394bf",
    urls = ["https://github.com/bazelbuild/rules_docker/releases/download/v0.25.0/rules_docker-v0.25.0.tar.gz"],
)

load(
    "@io_bazel_rules_docker//repositories:repositories.bzl",
    container_repositories = "repositories",
)
container_repositories()

load("@io_bazel_rules_docker//repositories:deps.bzl", container_deps = "deps")

container_deps()

load(
    "@io_bazel_rules_docker//container:container.bzl",
    "container_pull",
)

# rust_image
load(
    "@io_bazel_rules_docker//rust:image.bzl",
    _rust_image_repos = "repositories",
)

_rust_image_repos()
```

These rules won't work without also having an empty `BUILD` file in the root of the repo, so lets make that:
```shell
touch $HOME/repo/BUILD
```

## Building container
Now lets build the rust container image by editing `$HOME/repo/src/services/summation/BUILD`. We're going to add
a new `load` rule and also a `rust_image` target, resulting in:
```python
load("@rules_rust//rust:defs.bzl", "rust_binary")
load("@io_bazel_rules_docker//rust:image.bzl", "rust_image")

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

rust_image(
    name = "server_image",
    binary = ":server",
)
```

That's pretty simple, we just have to tell the `rust_image` rule the binary target we want it
to put in a container.

Let's try building it, this time using `-c opt` so we get the equivilant of a `--release` build:
```shell
bazel build -c opt //...

```

If you have docker installed, you can run the image using:
```shell
bazel run -c opt //src/services/summation:server_image
```

And then from another window you should be able to test with grpcurl:
```shell
$ grpcurl -proto $HOME/repo/src/proto/summation/summation.proto -plaintext -d '{"value": 5.0, "value": 2.0}' localhost:50051 src_proto_summation.Summation/ComputeSumF64
{
  "sum": 7
}
```

To bring down the container you can run `docker ps` to find its id and use `docker kill [container id]` to bring it down.

If you wanted to pass arguments to docker you can use `--` in the bazel run command and then include them. For example:
```
bazel run -c opt //src/services/summation:server_image -- -d -p 50051:50051
```
