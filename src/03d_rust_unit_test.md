# Unit test rust library

To add tests to our rust library we'll use the rules_rust [rust_test](https://bazelbuild.github.io/rules_rust/defs.html#rust_test) rule. We can test in our library source files or seperate out testing into their own files. In this case since our unit tests are simple we'll put them in the same source files as the functions they are testing.

Let's add the rule to `$HOME/repo/src/summation/BUILD` by adding these lines:
```python
rust_test(
    name = "lib_test",
    crate = ":src_summation",
    deps = [],
)
```

We'll also have to update the top load line to include "rust_test" now:
```python
load("@rules_rust//rust:defs.bzl", "rust_binary", "rust_library", "rust_test")
```

Now we can run it. Bazel can automatically detect and rerun all the applicable tests that might be affected
by a change, and I usually run `bazel test //...` which says run all the tests when I'm testing.
If you run it you should see something like
```shell
INFO: Analyzed 3 targets (2 packages loaded, 156 targets configured).
INFO: Found 2 targets and 1 test target...
INFO: Elapsed time: 0.671s, Critical Path: 0.38s
INFO: 5 processes: 2 internal, 3 linux-sandbox.
INFO: Build completed successfully, 5 total actions
//src/summation:lib_test                                                 PASSED in 0.0s

Executed 1 out of 1 test: 1 test passes.
There were tests whose specified size is too big. Use the --test_verbose_timeout_warnings command line option to see which ones these are.
###TODO UPDATE
```

You can see it "ran" our tests but right now we haven't defined any tests so it's a bit of no-op.
Let's add some test that we expect to fail and see what happens.
We'll add this to `$HOME/repo/src/summation/f64.rs`
```rust
pub fn summation_f64(values: &[f64]) -> f64 {
    values.iter().sum()
}

#[cfg(test)]
mod test {
    use super::summation_f64;

    #[test]
    fn simple_test() {
        let res = summation_f64(&[0.0, 1.0,  2.0]);
        assert_eq!(res, 0.0);
    }
}
```

And run `bazel test //...` again:
```shell
INFO: Analyzed 3 targets (0 packages loaded, 0 targets configured).
INFO: Found 2 targets and 1 test target...
FAIL: //src/summation:lib_test (see /home/parallels/.cache/bazel/_bazel_parallels/8136e33dd0c038f4f223262d62801c45/execroot/__main__/bazel-out/k8-fastbuild/testlogs/src/summation/lib_test/test.log)
INFO: Elapsed time: 0.384s, Critical Path: 0.24s
INFO: 4 processes: 4 linux-sandbox.
INFO: Build completed, 1 test FAILED, 4 total actions
//src/summation:lib_test                                                 FAILED in 0.0s
  /home/parallels/.cache/bazel/_bazel_parallels/8136e33dd0c038f4f223262d62801c45/execroot/__main__/bazel-out/k8-fastbuild/testlogs/src/summation/lib_test/test.log

Executed 1 out of 1 test: 1 fails locally.
```

Ok we can see our test failed. Bazel also gives us the path to the `test.log` file. If we look at that it contains:
```
exec ${PAGER:-/usr/bin/less} "$0" || exit 1
Executing tests from //src/summation:lib_test
-----------------------------------------------------------------------------

running 1 test
test f64::test::simple_test ... FAILED

failures:

---- f64::test::simple_test stdout ----
thread 'f64::test::simple_test' panicked at 'assertion failed: `(left == right)`
  left: `3.0`,
 right: `0.0`', src/summation/f64.rs:12:9
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    f64::test::simple_test

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

So we can see our test failed because 3.0 does not equal 0.0. Let's fix the test by changing the `assert_eq!(res, 0.0);` to `assert_eq!(res, 3.0);` and rerun:
```shell
$ bazel test //...
WARNING: Ignoring JAVA_HOME, because it must point to a JDK, not a JRE.
INFO: Analyzed 3 targets (0 packages loaded, 0 targets configured).
INFO: Found 2 targets and 1 test target...
INFO: Elapsed time: 0.396s, Critical Path: 0.26s
INFO: 4 processes: 4 linux-sandbox.
INFO: Build completed successfully, 4 total actions
//src/summation:lib_test                                                 PASSED in 0.0s

Executed 1 out of 1 test: 1 test passes.
There were tests whose specified size is too big. Use the --test_verbose_timeout_warnings command line option to see which ones these are.
```

Looks like our tests are working! Next lets make the CLI call our library.