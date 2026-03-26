# OpenShift API Data Protection (OADP) - Basic Usage Demo

This repository demonstrates the basic usage of OpenShift API Data Protection (OADP) configured with MultiCluster Gateway (MCG). It walks through creating a storage backend, configuring the Data Protection Application (DPA), deploying a sample application, and performing a full backup and restore cycle.

### Prerequisites
* An OpenShift Container Platform (OCP) cluster.
* OADP Operator installed.
* OpenShift Data Foundation (ODF) / MultiCluster Gateway (MCG) configured.
* `oc` CLI tool installed and authenticated.

> [!NOTE]
> Instructions here: https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/backup_and_restore/
index#configuring-oadp-with-mcg

## Phase 1: Environment Setup 

First, set up your environment variables and create the namespace for our demo application.

```bash
export DEMO_PROJECT=oadp-demo
export CREDENTIALS_VELERO_FILE=/tmp/credentials-velero
oc new-project $DEMO_PROJECT
```

## Phase 2: Storage Configuration

We need to create an Object Bucket Claim (OBC) for the MultiCluster Gateway. This will automatically provision a bucket in the storage backend, generate a ConfigMap with the bucket details, and create a Secret containing the S3 access credentials.

```bash
cat <<EOF | oc apply -f -
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata: 
  name: cluster-backup-bucket
  namespace: openshift-adp
spec:
  generateBucketName: ocp1-cluster-backup
  storageClassName: openshift-storage.noobaa.io
EOF
```

## Phase 3: OADP Configuration

### 1. Extract Credentials

Retrieve the generated S3 access keys and format them into a credentials file for Velero.

```bash
cat <<EOF > $CREDENTIALS_VELERO_FILE
[default]
aws_access_key_id=$(oc get secret cluster-backup-bucket -n openshift-adp -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
aws_secret_access_key=$(oc get secret cluster-backup-bucket -n openshift-adp -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
EOF
```
### 2. Create the Cloud Credentials Secret

Create the generic secret that OADP will use to access the backup storage location.

```bash
oc create secret generic cloud-credentials \
  --namespace openshift-adp \
  --from-file=cloud=$CREDENTIALS_VELERO_FILE \
  --dry-run=client -o yaml | oc apply -f -
```

### 3. Deploy the DataProtectionApplication (DPA)

Apply the DPA custom resource. This configuration dynamically pulls the required S3 URL and Bucket Name from the cluster resources we created earlier.

```bash
cat <<EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-demo-oadp
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - aws
        - openshift
      resourceTimeout: 10m
    nodeAgent:
      enable: true
      uploaderType: kopia
  backupLocations:
    - velero:
        config:
          profile: "default"
          region: "noobaa"
          s3Url: https://$(oc get route -n openshift-storage s3 -ojsonpath='{.spec.host}')
          insecureSkipTLSVerify: "true"
          s3ForcePathStyle: "true"
        provider: aws
        default: true
        credential:
          key: cloud
          name: cloud-credentials
        objectStorage:
          bucket: $(oc get configmap -n openshift-adp cluster-backup-bucket -ojsonpath='{.data.BUCKET_NAME'})
          prefix: backup
EOF
```

### 4. Set Velero Alias

Once the DPA is fully installed and running, set an alias to use the Velero CLI directly from the deployment pod:

```bash
alias velero='oc -n openshift-adp exec deployment/velero -c velero -it -- ./velero'
```

## Phase 4: Deploying a Sample Application

Let's deploy a simple "Hello OpenShift" application to test our backup process.

```bash
oc create deploy hello -n $DEMO_PROJECT --image=openshift/hello-openshift
oc expose deploy/hello -n $DEMO_PROJECT --port=8080
oc expose svc/hello -n $DEMO_PROJECT 
oc get deploy,pod,svc,route -n $DEMO_PROJECT
```

Test the application to ensure it is responding correctly:

```bash
curl $(oc get route -n $DEMO_PROJECT hello -ojsonpath='{.spec.host}')
# Expected result: "Hello OpenShift!"
```

## Phase 5: Backup and Restore Process

### 1. Create a Backup

Initiate a backup of the entire demo namespace.

```bash
velero create backup demo-backup --include-namespaces=$DEMO_PROJECT
```

Check the progress and details of your backup:

```bash
velero backup describe demo-backup
```

### 2. Simulate a Disaster

Delete the namespace to simulate a data loss event.

```bash
oc delete project $DEMO_PROJECT
```

Verify that the application is no longer reachable:

```bash
curl $(oc get route -n $DEMO_PROJECT hello -ojsonpath='{.spec.host}')
# Expected result: Error
```

### 3. Restore the Backup

Restore the namespace and its resources from the Velero backup.

```bash
velero create restore demo-restore --from-backup=demo-backup
```

### 4. Verify the Restore

Test the application route again to confirm the restore was successful.

```bash
curl $(oc get route -n $DEMO_PROJECT hello -ojsonpath='{.spec.host}')
# Expected result: "Hello OpenShift!"
```

