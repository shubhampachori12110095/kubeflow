# Test Infrastructure

This directory contains the Kubeflow test Infrastructure.

This is a work in progress see [google/kubeflow#38](https://github.com/google/kubeflow/issues/38)

The current thinking is this will work as follows

  * Prow will be used to trigger E2E tests
  * The E2E test will launch an Argo workflow that describes the tests to run
  * Each step in the Argo workflow will be a binary invoked inside a container
  * The Argo workflow will use an NFS volume to attach a shared POSIX compliant filesystem to each step in the 
    workflow.
  * Each step in the pipeline can write outputs and junit.xml files to a test directory in the volume
  * A final step in the Argo pipeline will upload the outputs to GCS so they are available in gubernator

## Accessing Argo UI

You can access the Argo UI over the API Server proxy.

We currently use the cluster

```
PROJECT=mlkube-testing
ZONE=us-east1-d
CLUSTER=kubeflow-testing
NAMESPACE=kubeflow-test-infra
```

After starting `kubectl proxy` on `127.0.0.1:8001`, you can connect to the argo UI via the local proxy at

```
http://127.0.0.1:8001/api/v1/proxy/namespaces/kubeflow-test-infra/services/argo-ui:80/
```

TODO(jlewi): We can probably make the UI publicly available since I don't think it offers any ability to launch workflows.


## Running the tests

### Run a presubmit

```
ks param set workflows name e2e-test-pr-`date '+%Y%m%d-%H%M%S'`
ks param set workflows prow_env REPO_OWNER=google,REPO_NAME=kubeflow,PULL_NUMBER=${PULL_NUMBER},PULL_PULL_SHA=${COMMIT}
ks param set workflows commit ${COMMIT}
ks apply prow -c workflows
```
	* You can set COMMIT to `pr` to checkout the latest change on the PR.

### Run a postsubmit

```
ks param set workflows name e2e-test-postsubmit-`date '+%Y%m%d-%H%M%S'`
ks param set workflows prow_env REPO_OWNER=google,REPO_NAME=kubeflow,PULL_BASE_SHA=${COMMIT}
ks param set workflows commit ${COMMIT}
ks apply prow -c workflows
```
  * You can set COMMIT to `master` to use HEAD


## Setting up the Test Infrastructure

Our tests require a K8s cluster with Argo installed. This section provides the instructions 
for setting this.

Create a GKE cluster

```
PROJECT=mlkube-testing
ZONE=us-east1-d
CLUSTER=kubeflow-testing
NAMESPACE=kubeflow-test-infra

gcloud --project=${PROJECT} container clusters create \
	--zone=${ZONE} \
	--machine-type=n1-standard-8 \
	--cluster-version=1.8.4-gke.1 \
	${CLUSTER}
```


### Create a GCP service account
	
	* The tests need a GCP service account to upload data to GCS for Gubernator

	```
	SERVICE_ACCOUNT=kubeflow-testing
	gcloud iam service-accounts --project=mlkube-testing create ${SERVICE_ACCOUNT} --display-name "Kubeflow testing account"
	gcloud projects add-iam-policy-binding ${PROJECT} \
    	--member serviceAccount:${SERVICE_ACCOUNT}@${PROJECT}.iam.gserviceaccount.com --role roles/container.developer
	```
		* The service account needs to be able to create K8s resources as part of the test.


	Create a secret key for the service account

	```
	gcloud iam service-accounts keys create ~/tmp/key.json \
    	--iam-account ${SERVICE_ACCOUNT}@${PROJECT}.iam.gserviceaccount.com
    kubectl create secret generic kubeflow-testing-credentials \
        --namespace=kubeflow-test-infra --from-file=`echo ~/tmp/key.json`
    rm ~/tmp/key.json
	```

	Make the service account a cluster admin

	```
	kubectl create clusterrolebinding  ${SERVICE_ACCOUNT}-admin --clusterrole=cluster-admin  \
		--user=${SERVICE_ACCOUNT}@${PROJECT}.iam.gserviceaccount.com 
	```
		* The service account is used to deploye Kubeflow which entails creating various roles; so 
		  it needs sufficient RBAC permission to do so.

### Create a GitHub Token

You need to use a GitHub token with ksonnet otherwise the test quickly runs into GitHub API limits.

TODO(jlewi): We should create a GitHub bot account to use with our tests and then create API tokens for that bot.

You can use the GitHub API to create a token

   * The token doesn't need any scopes because its only accessing public data and is just need for API metering.

To create the secret run

```
kubectl create secret generic github-token --namespace=kubeflow-test-infra --from-literal=github_token=${TOKEN}
```

### Create a PD for NFS

Create a PD to act as the backing storage for the NFS filesystem that will be used to store data from
the test runs.

```
  gcloud --project=${PROJECT} compute disks create  \
  	--zone=${ZONE} kubeflow-testing --description="PD to back NFS storage for kubeflow testing." --size=1TB
```
### Create K8s Resources for Testing

The ksonnet app `test-infra` contains ksonnet configs to deploy the test infrastructure.

You can deploy argo as follows (you don't need to use argo's CLI)

```
ks apply prow -c argo
```  

Deploy NFS & Jupyter

```
ks apply prow -c nfs-jupyter
```

	* This creates the NFS share
	* We use JupyterHub as a convenient way to access the NFS share for manual inspection of the file contents.

#### Troubleshooting

User or service account deploying the test infrastructure needs sufficient permissions to create the roles that are created as part deploying the test infrastructure. So you may need to run the following command before using ksonnet to deploy the test infrastructure.

```
kubectl create clusterrolebinding default-admin --clusterrole=cluster-admin --user=user@gmail.com
```

## Managing namespaces

All namespaces created for the tests should be labeled with `app=kubeflow-e2e-test`.

This can be used to manually delete old namespaces that weren't properly garbage collected.
