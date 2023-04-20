# PS: Tearing down

If you ran all the gcloud commands in earlier chapters you'll have created a vm, an artifact registry, and a cloud run service as part of going through the guide.

To tear these all down use the following commands:
```shell
# Delete the VM
gcloud compute instances delete hellobazel

# Delete the cloud run service
gcloud run services delete hello-bazel-service --region us-central1

# Delete the artifact registry
gcloud artifacts repositories delete quickstart-docker-repo --location=us-central1
```