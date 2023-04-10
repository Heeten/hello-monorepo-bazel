# Prerequisites and Installing Bazel

I'm going to start of with a minimal VM built on Google Cloud's Platform and walk through the process
of installing and running Bazel on it. I do this so that these instructions are reproducible for people
to follow along with and I don't accidently miss a step (like installing a C++ toolchain).

Feel free to jump down to Step X which covers installing Bazelisk.

## Step 0: Building our VM and connecting to it

I'm going to use the gcloud CLI ([gcloud CLI instalation instructions](https://cloud.google.com/sdk/docs/install)) to setup a VM on Google Cloud. If you already have a linux machine you can skip this.

Log into Google Cloud and set the project
```shell
gcloud auth login
gcloud config set project [PROJECT_ID]
```

Create a debian instance (Bazel of course works on all kinds of different environments, this is just the one I use for this doc).
```shell
gcloud compute instances create hellobazel \
  --image-family debian-11 \
  --image-project debian-cloud \
  --zone us-central1-a \
  --machine-type e2-standard-2 \
  --boot-disk-size 20GB
```

Now lets connect to it
```shell
gcloud compute ssh hellobazel
```

## Step 1: Downloading bazelisk and running
Now that we have a new machine we can mess around with, let's install Bazelisk.

You can find the official Bazel install docs at [https://bazel.build/install](https://bazel.build/install).
Those mention Bazelisk is the recommended way to install Bazel, which is what we'll use. Bazelisk is available
as a binary you can download from their [GitHub release page](https://github.com/bazelbuild/bazelisk/releases).
We're going to pull down the v1.16.0 binary by running:
```shell
mkdir bazel
cd bazel
curl \
  -L https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-linux-amd64 \
  -o bazel
```

Now let's make the file we just downloaded executable
```shell
chmod +x bazel
```

And add it to our path
```shell
export PATH="${PATH}:$HOME/bazel"
```

Finally lets run `bazel` and see what happens. It should download something and then give you the Usage.
Here's what I get
```shell
$ bazel
2023/04/10 17:25:16 Downloading https://releases.bazel.build/6.1.1/release/bazel-6.1.1-linux-x86_64...
WARNING: Invoking Bazel in batch mode since it is not invoked from within a workspace (below a directory having a WORKSPACE file).
Extracting Bazel installation...
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

Now that we have bazel working, let's start using it in the next chapter!