# Custom Function Router Implementation

## Create a GCP Pubsub Source

### Prerequisites

1. Create a
   [Google Cloud project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
   and install the `gcloud` CLI and run `gcloud auth login`. This sample will
   use a mix of `gcloud` and `kubectl` commands. The rest of the sample assumes
   that you've set the `$PROJECT_ID` environment variable to your Google Cloud
   project id, and also set your project ID as default using
   `gcloud config set project $PROJECT_ID`.

1. Setup [Knative Serving](https://github.com/knative/docs/blob/master/install)

1. Setup
   [Knative Eventing](https://github.com/knative/docs/tree/master/eventing). In
   addition, install the GCP PubSub event source from `release-gcppubsub.yaml`:

   ```shell
   kubectl apply --filename https://github.com/knative/eventing-sources/releases/download/v0.3.0/release-gcppubsub.yaml
   ```

1. Enable the `Cloud Pub/Sub API` on your project:

   ```shell
   gcloud services enable pubsub.googleapis.com
   ```

1. Create a
   [GCP Service Account](https://console.cloud.google.com/iam-admin/serviceaccounts/project).
   This sample creates one service account for both registration and receiving
   messages, but you can also create a separate service account for receiving
   messages if you want additional privilege separation.

   1. Create a new service account named `knative-source` with the following
      command:
      ```shell
      gcloud iam service-accounts create knative-source
      ```
   1. Give that Service Account the `Pub/Sub Editor` role on your GCP project:
      ```shell
      gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member=serviceAccount:knative-source@$PROJECT_ID.iam.gserviceaccount.com \
        --role roles/pubsub.editor
      ```
   1. Download a new JSON private key for that Service Account. **Be sure not to
      check this key into source control!**
      ```shell
      gcloud iam service-accounts keys create knative-source.json \
        --iam-account=knative-source@$PROJECT_ID.iam.gserviceaccount.com
      ```
   1. Create two secrets on the kubernetes cluster with the downloaded key:

      ```shell
      # Note that the first secret may already have been created when installing
      # Knative Eventing. The following command will overwrite it. If you don't
      # want to overwrite it, then skip this command.
      kubectl --namespace knative-sources create secret generic gcppubsub-source-key --from-file=key.json=knative-source.json --dry-run --output yaml | kubectl apply --filename -

      # The second secret should not already exist, so just try to create it.
      kubectl --namespace default create secret generic google-cloud-key --from-file=key.json=knative-source.json
      ```

      `gcppubsub-source-key` and `key.json` are pre-configured values in the
      `controller-manager` StatefulSet which manages your Eventing sources.

      `google-cloud-key` and `key.json` are pre-configured values in
      [`gcp-pubsub-source.yaml`](./gcp-pubsub-source.yaml).
      
      
### Deployment
1. Create a channel called `isv-channel`.

	 ```shell
   kubectl apply --filename isv-channel.yaml
   ```
   

1. Create a GCP PubSub Topic. If you change its name (`raw_event`), you also need
   to update the `topic` in the
   [`gcp-pubsub-source.yaml`](./gcp-pubsub-source.yaml) file:

   ```shell
   gcloud pubsub topics create raw_event
   ```

1. Replace the
   [`MY_GCP_PROJECT` placeholder](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
   in [`gcp-pubsub-source.yaml`](./gcp-pubsub-source.yaml) and apply it.

   If you're in the samples directory, you can replace `MY_GCP_PROJECT` and
   apply in one command:

   ```shell
    sed "s/MY_GCP_PROJECT/$PROJECT_ID/g" gcp-pubsub-source.yaml | \
        kubectl apply --filename -
   ```

   If you are replacing `MY_GCP_PROJECT` manually, then make sure you apply the
   resulting YAML:

   ```shell
   kubectl apply -f gcp-pubsub-source.yaml
   ```
 
1. Create a service account so that the subscriber can make k8s api calls.

	```shell
	kubectl apply -f custom-router-service-account.yaml
	``` 
	
1. Create a cluster role binding so that the subscriber can make k8s api calls.

	```shell
	kubectl apply -f custom-router-cluster-role.yaml
	``` 
   
1. Create a function and subscribe it to the `isv-channel` channel:
   Change the env variables if adding additional sinks.

	```shell 
	kubectl apply --filename custom-router-subscriber.yaml   
	```
