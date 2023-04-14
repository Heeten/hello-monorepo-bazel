# 'Release mode' builds

In bazel instead of passing `--release` you need to set the [compilation-mode](https://bazel.build/docs/user-manual#compilation-mode), which is a command-line option you can set that affects both the rust and c++ compilation in our example repo. When you run `bazel build` or `bazel run` you'll want to use this flag to get compiler optimizations turned on.
```shell
#Compiles everything with optimization enabled
bazel build -c opt //...
```

```shell
#Runs out example CLI from a build with optimization enabled
bazel run -c opt //src/summation:executable
```