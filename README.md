# Migration Toolkit for Containers Demo

## Setup

1. Provision two clusters (OCP 4.8 and 4.9).
2. Install ACM on the 4.9 cluster.
3. Import the 4.8 cluster into ACM.
4. Run the "Initial Setup" commands to import all demo config policies.
5. Label the 4.9 cluster with the following labels:
    * `odfVersion=4.9`
    * `mtc-demo=destintion`
6. Label the 4.8 cluster with the following labels:
    * `mtc-demo=source`
7. When logged into the 4.8 cluster with the `oc` cli, run the following command to install the app that will be migrated:
    * `oc apply -k demo`

## On the "Source" Cluster

Deploy the stateful demo apps:

```
oc apply -k demo
```

Create a `ServiceAccount` for the migration tool to use.  Give it the `cluster-admin` role and fetch it's token:

```
oc create sa migration-sa -n openshift-migration

oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccounts:openshift-migration:migration-sa -n openshift-migration

oc sa get-token migration-sa -n openshift-migration

```

Get the API url for this cluster:

```
oc whoami --show-server
```

Get the registry URL:

```
echo "https://$(oc get route default-route -n openshift-image-registry -o template='{{.spec.host}}')"
```

## Configure MTC on "Destination" Cluster

1. Create an `ObjectBucketClaim` to use for data transfer:
    * `oc apply -f bucketclaim -n openshift-storage`
2. Get the details for the bucket:
    * S3 URL: `echo "https://$(oc get route s3 -n openshift-storage -o template='{{.spec.host}}')"`
    * AWS Secret Key ID: `echo "AWS_ACCESS_KEY_ID: $(oc get secret mig-repository -n openshift-migration -o template='{{.data.AWS_ACCESS_KEY_ID}}' | base64 -d)"`
    * AWS Secret Access Key: `echo "AWS_SECRET_ACCESS_KEY: $(oc get secret mig-repository -n openshift-migration -o template='{{.data.AWS_SECRET_ACCESS_KEY}}' | base64 -d)"`
    * Bucket Name: `echo "Bucket Name: $(oc get cm mig-repository -n openshift-migration -o template='{{.data.BUCKET_NAME}}')"`
3. Login to MTC.  You can find the route in `openshift-migration` project.
4. Add a "Replication Repository" with the following configuration:
    * **Storage provider Type:** S3
    * **Replication repository name:** migration-repsitory
    * **S3 bucket name:** `<bucket name from above>`
    * **S3 bucket region:** local
    * **S3 endpoint:** `<S3 url from above>`
    * **S3 provider access key:** `<access key id from above>`
    * **S3 provider secret access key:** `<secret access key from above>`
    * Click "Add Repository"
5. Add the "Source" cluster.
    * Under "Clusters", click the "Add cluster" button.
    * **Name:** source
    * **URL:** `<api url from above>`
    * **Service account token:** `<service account token from above>`
    * **Exposed route to image registry:** `<registry route from above>`
    * Add cluster

## Prepare a Migration

Under "Migration plans" in the side nav, click the "Add migration plan" to create a new plan.

Follow the guided steps to pick the namespaces you want to migrate from the "source" cluster to the "destination" cluster.

