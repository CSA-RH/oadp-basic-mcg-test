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
export DEMO_PROJECT=oadp-demo-stateless
export DEMO_PROJECT_STATEFUL=oadp-demo-stateful
export CREDENTIALS_VELERO_FILE=/tmp/credentials-velero
oc new-project $DEMO_PROJECT
```


## Phase 2: Storage Configuration

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

### 1. Extract Credentials

Retrieve the generated S3 access keys and format them into a credentials file for Velero.

```bash
cat <<EOF > $CREDENTIALS_VELERO_FILE
[default]
aws_access_key_id=$(oc get secret cluster-backup-bucket -n openshift-adp -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
aws_secret_access_key=$(oc get secret cluster-backup-bucket -n openshift-adp -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
EOF
```
### 2. Create the Cloud Credentials Secret

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
        - openshift
        - aws
        - csi
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
          bucket: $(oc get configmap -n openshift-adp cluster-backup-bucket -ojsonpath='{.data.BUCKET_NAME}')
          prefix: backup
EOF
```

### 4. Set Velero Alias

Once the DPA is fully installed and running, set an alias to use the Velero CLI directly from the deployment pod:

```bash
alias velero='oc -n openshift-adp exec deployment/velero -c velero -it -- ./velero'
```


## Backup and restore of Stateless applications

### Phase 4: Deploying a Sample Application (Stateless)

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

### Phase 5: Backup and Restore Process (Stateless)

#### 1. Create a Backup

Initiate a backup of the entire demo namespace.

```bash
velero create backup demo-backup --include-namespaces=$DEMO_PROJECT
```

Check the progress and details of your backup:

```bash
velero backup describe demo-backup
```

#### 2. Simulate a Disaster

Delete the namespace to simulate a data loss event.

```bash
oc delete project $DEMO_PROJECT
```

Verify that the application is no longer reachable:

```bash
curl $(oc get route -n $DEMO_PROJECT hello -ojsonpath='{.spec.host}')
# Expected result: Error
```

#### 3. Restore the Backup

Restore the namespace and its resources from the Velero backup.

```bash
velero create restore demo-restore --from-backup=demo-backup
```

#### 4. Verify the Restore

Test the application route again to confirm the restore was successful.

```bash
curl $(oc get route -n $DEMO_PROJECT hello -ojsonpath='{.spec.host}')
# Expected result: "Hello OpenShift!"
```


## Backup and restore of Stateful applications

### Phase 4: Deploying a Sample Application (Stateful)

```bash
cat <<EOF | oc apply -f - 
apiVersion: v1
kind: List
items:
  - kind: Namespace
    apiVersion: v1
    metadata:
      name: ${DEMO_PROJECT_STATEFUL}
      labels:
        app: mysql
  - kind: ServiceAccount
    apiVersion: v1
    metadata:
      name: ${DEMO_PROJECT_STATEFUL}-sa
      namespace: ${DEMO_PROJECT_STATEFUL}
      labels:
        component: mysql-persistent
  - kind: SecurityContextConstraints
    apiVersion: security.openshift.io/v1
    metadata:
      name: ${DEMO_PROJECT_STATEFUL}-scc
    allowPrivilegeEscalation: true
    allowPrivilegedContainer: true
    runAsUser:
      type: RunAsAny
    seLinuxContext:
      type: RunAsAny
    fsGroup:
      type: RunAsAny
    supplementalGroups:
      type: RunAsAny
    volumes:
    - '*'
    users:
    - system:admin
    - system:serviceaccount:${DEMO_PROJECT_STATEFUL}:${DEMO_PROJECT_STATEFUL}-sa
  - kind: Service
    apiVersion: v1
    metadata:
      name: todolist
      namespace: ${DEMO_PROJECT_STATEFUL}
      labels:
        app: todolist
        service: todolist
        e2e-app: "true"
    spec:
      ports:
        - name: web
          port: 8000
          targetPort: 8000
      selector:
        app: todolist
        service: todolist
  - kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: todolist
      namespace: ${DEMO_PROJECT_STATEFUL}
      labels:
        app: todolist
        service: todolist
        e2e-app: "true"
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: todolist
          service: todolist
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: todolist
            service: todolist
            e2e-app: "true"
        spec:
          securityContext:
            runAsGroup: 27
            runAsUser: 27
            fsGroup: 27
          serviceAccountName: ${DEMO_PROJECT_STATEFUL}-sa
          containers:
          - name: todolist
            image: quay.io/migtools/oadp-ci-todo2-go-testing-mariadb:latest
            securityContext:
              runAsGroup: 27
              runAsUser: 27
              privileged: true
            env:
              - name: DB_BACKEND
                value: mariadb
              - name: MYSQL_USER
                value: todouser
              - name: MYSQL_PASSWORD
                value: todopass
              - name: MYSQL_ROOT_PASSWORD
                value: root
              - name: MYSQL_DATABASE
                value: todolist
            ports:
              - containerPort: 8000
                protocol: TCP
            resources:
              limits:
                memory: 512Mi
            volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql/data
            startupProbe:
              httpGet:
                path: /healthz
                port: 8000
              initialDelaySeconds: 10
              periodSeconds: 5
              timeoutSeconds: 10
              successThreshold: 1
              failureThreshold: 60
            livenessProbe:
              httpGet:
                path: /healthz
                port: 8000
              initialDelaySeconds: 30
              periodSeconds: 10
            readinessProbe:
              httpGet:
                path: /healthz
                port: 8000
              initialDelaySeconds: 10
              periodSeconds: 5
          volumes:
          - name: mysql-data
            persistentVolumeClaim:
              claimName: mysql
  - kind: Route
    apiVersion: route.openshift.io/v1    
    metadata:
      name: todolist-route
      namespace: ${DEMO_PROJECT_STATEFUL}
    spec:
      path: "/"
      tls: 
        termination: edge
      to:
        kind: Service
        name: todolist
  - kind: PersistentVolumeClaim
    apiVersion: v1 
    metadata:
      name: mysql
      namespace: ${DEMO_PROJECT_STATEFUL}
      labels:
        app: mysql
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: ocs-external-storagecluster-ceph-rbd
      resources:
        requests:
          storage: 1Gi
EOF
```

Wait to the todolist pod to start, open the route path in a brower and create some items in the list. Connect to the database in the todolist pod with the following credentials inside the pod (for instance, in the Pod Terminal window in openshift): 

```bash
mariadb -u $MYSQL_USER --password=$MYSQL_PASSWORD $MYSQL_DATABASE
```

Show the tables available. 

```sql
SHOW TABLES;
```

Show the items of the table `todo_items`.

```sql
SELECT * FROM todo_items;
```

### Phase 5: Backup and Restore Process (Stateful)

#### 1. Create a Backup

```bash
velero create backup demo-backup-stateful --include-namespaces=$DEMO_PROJECT_STATEFUL
```

#### 2. Simulate a Disaster

```bash
oc delete project $DEMO_PROJECT_STATEFUL
```

#### 3. Restore the Backup

```bash
velero create restore demo-restore-stateful --from-backup=demo-backup-stateful
```

#### 4. Verify the Restore

Browse to the route address and check the items. 