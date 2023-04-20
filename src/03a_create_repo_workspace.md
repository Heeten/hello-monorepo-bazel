# Create repository and bazel workspace

One of the benefits of Bazel is when you have a lot of different
components/modules/crates/packages/whatever-you-call-it that need to connect together. In our example
we're going to imagine there's a team that owns a rust summation() function and that they
provide a rust interface and rust CLI to call these functions. Then we'll have downstream
dependencies in C++ and Rust that use this summation() function.

> **NOTE**: There are different ways to organize a monorepo. Bazel doesn't
> care which way you use, but sometimes IDEs do. In this example repo, we
> organize directories by project/team ownership and mix source files from different
> languages together. An alternate is to have the top-level directories
> be split by language (c++, rust, py, etc) or by project ang language.

## Create repository
We'll put our repository in the `$HOME/repo`. Let's go ahead and make that
```shell
mkdir $HOME/repo
cd $HOME/repo
```

## Create workspace
Bazel has a concept of workspace which I think of as the root of the monorepo. In the repo directory
we'll create a file called `WORKSPACE` which we'll put repo wide configuration in for pulling
down external dependencies. In this chapter the main external dependency we'll need is pulling down
[rules_rust](https://bazelbuild.github.io/rules_rust/) which contains the bazel rules for how to go from rust
source files to libraries (crates) and binaries.

Go ahead and open up `$HOME/repo/WORKSPACE` in your favorite text editors[^note] and put this in there. Bazel uses
the Starlark language for configuration files, which is a dialect of Python.
```python
# This command tells bazel to load the http_archive rule which is used
# to download other rulesets like rules_rust below
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# This pulls down rules_rust
# The path and sha256 come from https://github.com/bazelbuild/rules_rust/releases
http_archive(
    name = "rules_rust",
    sha256 = "950a3ad4166ae60c8ccd628d1a8e64396106e7f98361ebe91b0bcfe60d8e4b60",
    urls = ["https://github.com/bazelbuild/rules_rust/releases/download/0.20.0/rules_rust-v0.20.0.tar.gz"],
)

#What to load and run to setup rust are documented at https://bazelbuild.github.io/rules_rust/
load("@rules_rust//rust:repositories.bzl", "rules_rust_dependencies", "rust_register_toolchains")

rules_rust_dependencies()

rust_register_toolchains()
```

Once we've saved that file if we run `bazel` again we should see it doesn't output the `WARNING: Invoking Bazel in batch mode...` line.
```shell
$ bazel
Starting local Bazel server and connecting to it...
                                                           [bazel release 6.1.1]
Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  aquery              Analyzes the given targets and queries the action graph.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  coverage            Generates code coverage report for specified test targets.
  cquery              Loads, analyzes, and queries the specified targets w/ configurations.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  license             Prints the license of this software.
  mobile-install      Installs targets to mobile devices.
  modquery            Queries the Bzlmod external dependency graph
  print_action        Prints the command line args for compiling a file.
  query               Executes a dependency graph query.
  run                 Runs the specified target.
  shutdown            Stops the bazel server.
  sync                Syncs all repositories specified in the workspace file
  test                Builds and runs the specified test targets.
  version             Prints version information for bazel.

Getting more help:
  bazel help <command>
                   Prints help and options for <command>.
  bazel help startup_options
                   Options for the JVM hosting bazel.
  bazel help target-syntax
                   Explains the syntax for specifying targets.
  bazel help info-keys
                   Displays a list of keys used by the info command.
```

[^note]: I installed emacs on the vm at this point with `sudo apt-get install emacs-nox`