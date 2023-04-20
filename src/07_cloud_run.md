# Pushing containers and running on Google Cloud

`rules_docker` provides a [container_push](https://github.com/bazelbuild/rules_docker#container_push) rule which can be used to push the container to a container/artifact registry. We'll provide an example of doing that and then using
Google Cloud Run to launch our gRPC server container on Google Cloud.

You'll need `gcloud` and `docker` installed for these instructions to work.

## Setting up artifact registry

[Google's artifact-registry instructions](https://cloud.google.com/artifact-registry/docs/docker/store-docker-container-images#create) say you can use the following to create the artifact registry instance we'll use below:
```shell
gcloud artifacts repositories create quickstart-docker-repo --repository-format=docker \
--location=us-central1 --description="Docker repository"
```

Then we need to [configure authentication](https://cloud.google.com/artifact-registry/docs/docker/store-docker-container-images#auth):
```shell
gcloud auth configure-docker us-central1-docker.pkg.dev
```


## Pushing the container
We're going to add the `container_push` instructions to `$HOME/repo/src/services/summation/BUILD` by adding:
```python
load("@io_bazel_rules_docker//container:container.bzl", "container_push")

### ... existing lines in file

container_push(
   name = "server_push",
   image = ":server_image",
   format = "Docker",
   registry = "us-central1-docker.pkg.dev",
   repository = "%YOU_PROJECT_NAME%/quickstart-docker-repo/server-image",
   tag = "dev",
)
```

You'll need to change `%YOUR_PROJECT_NAME%` to your Google cloud project name for this to work.

Now you can push a new image using `bazel run`:
```shell
bazel run -c opt //src/services/summation:server_push
```

If that succeeds it should tell you where the image was pushed and the sha256.

## Running on Google Cloud Run
Following