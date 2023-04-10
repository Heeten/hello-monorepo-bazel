# Rust and C++ depending on each other
For demonstration purposes, in this chapter we'll make a C++ library that a rust library depends on and wraps,
and then we'll make the rust binary depend on the rust library.

It's also possible to make a C++ binary depend on a rust library, to do that you would use the `rust_static_library` rule instead of `rust_library`, but we don't yet provide an example of doing that.

We'll also use Bazel's [visibility](https://bazel.build/concepts/visibility) concept to restrict other monorepo packages from depending on anything but the Rust library.

# Making the C++ library