# Link rust binary to library

Now that we have our library, lets make our rust binary use it.

First we'll update the `$HOME/repo/src/summation/BUILD` and add `":src_summation"` to the rust_binary deps, which tells Bazel to pull that crate into the sandbox our target is built in. The full `BUILD` file after this will look like:
```python
load("@rules_rust//rust:defs.bzl", "rust_binary", "rust_library", "rust_test")

rust_binary(
    #We are going to call the target/binary summation
    name = "executable",
    #The list of src files it needs (just main.rs)
    srcs = ["main.rs"],
    #Any libraries/crates it depends on, for now we'll leave this blank
    deps = [
        ":src_summation",
    ],
    #The crate_root file, this would default to main.rs but we put it in for clarity
    crate_root = "main.rs",
)

rust_library(
    name = "src_summation",
    srcs = [
        "lib.rs",
        "f64.rs",
        "u32.rs",
    ],
    deps = [],
)

rust_test(
    name = "lib_test",
    crate = ":src_summation",
    deps = [],
)
```

Next let's update `$HOME/repo/src/summation/main.rs` to use our crate. We'll have it parse command line arguments
as `f64` and then sum all of them.
```rust
use src_summation::f64::summation_f64;
use std::env;

fn main() {
    let args: Vec<f64> = env::args().skip(1).map(|a| a.parse().unwrap()).collect();
    println!("sum = {}", summation_f64(&args))
}

```

Now lets build and run it:
```shell
$ bazel run //src/summation:executable
INFO: Analyzed target //src/summation:executable (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //src/summation:executable up-to-date:
  bazel-bin/src/summation/executable
INFO: Elapsed time: 0.405s, Critical Path: 0.28s
INFO: 2 processes: 1 internal, 1 linux-sandbox.
INFO: Build completed successfully, 2 total actions
INFO: Running command line: bazel-bin/src/summation/executable
sum = 0
```

Let's try running it with some arguments. We'll use `--` to seperate arguments to bazel run and arguments
to our binary (we'll also omit some bazel log statements):
```shell
bazel run //src/summation:executable -- 1 2 3.0
sum = 6
```

Finally, let's run with an `optimized` binary by adding `-c opt`, getting something equivilant to running with `--release` in cargo:
```shell
bazel run -c opt //src/summation:executable -- 1 2 3.0
INFO: Build option --compilation_mode has changed, discarding analysis cache.
INFO: Analyzed target //src/summation:executable (0 packages loaded, 517 targets configured).
INFO: Found 1 target...
Target //src/summation:executable up-to-date:
  bazel-bin/src/summation/executable
INFO: Elapsed time: 0.840s, Critical Path: 0.40s
INFO: 48 processes: 46 internal, 2 linux-sandbox.
INFO: Build completed successfully, 48 total actions
INFO: Running command line: bazel-bin/src/summation/executable 1 2 3.0
sum = 6
```