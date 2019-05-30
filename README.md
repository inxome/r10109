[![inxome-image][inXomeImage]][report]

###### Report entry 2019 May 27: raaabc v12

***

### 1. Google [CodeLab][1] For [Spinnaker][2] 

![CodeLab][Fig1]

***

### 2. Set up Spinnaker

- Enabled [Cloud Sub/Pub API][3], [Cloud Build API][4] and [Kubernetes Engine API][5].
- Deploy Spinnaker

```text
assembly_checkpoint@cloudshell:~ (inxome-10101)$ gsutil cp gs://gke-spinnaker-codelab/install.tgz .
Copying gs://gke-spinnaker-codelab/install.tgz...
/ [1 files][  3.9 KiB/  3.9 KiB]
Operation completed over 1 objects/3.9 KiB.
```

#### 2.1 Unpack

```text
assembly_checkpoint@cloudshell:~ (inxome-10101)$ tar -xvzf install.tgz
./
./setup.sh
./manifests.yml
./connect.sh
./cleanup.sh
./properties
```

#### 2.2 Review the manifests input file for Kubernetes

This is the manifests template where variables like ```{%PROJECT_ID%}``` will be replace by the environment variable ```$PROJECT_ID``` declared in the properties file.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: spinnaker

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: spinnaker-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: spinnaker

---

apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: spin-halyard
  namespace: spinnaker
  labels:
    app: spin
    stack: halyard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spin
      stack: halyard
  template:
    metadata:
      labels:
        app: spin
        stack: halyard
    spec:
      containers:
      - name: halyard-daemon
        # todo - make :stable or digest of :stable
        image: gcr.io/spinnaker-marketplace/halyard:0.43.0-20180321134158
        imagePullPolicy: Always
        readinessProbe:
          exec:
            command:
            - wget
            - -q
            - --spider
            - http://localhost:8064/health
        ports:
        - containerPort: 8064
        volumeMounts:
        - name: halconfig
          mountPath: /root/.hal/config
          subPath: config
        - name: halconfig
          mountPath: /root/.hal/default/service-settings/deck.yml
          subPath: deck.yml
        - name: halconfig
          mountPath: /root/.hal/default/service-settings/gate.yml
          subPath: gate.yml
        - name: halconfig
          mountPath: /root/.hal/default/service-settings/igor.yml
          subPath: igor.yml
        - name: halconfig
          mountPath: /root/.hal/default/service-settings/fiat.yml
          subPath: fiat.yml
      volumes:
      - name: halconfig
        configMap:
          name: halconfig
---

apiVersion: v1
kind: Service
metadata:
  name: spin-halyard
  namespace: spinnaker
spec:
  ports:
    - port: 8064
      targetPort: 8064
      protocol: TCP
  selector:
    app: spin
    stack: halyard

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: halconfig
  namespace: spinnaker
data:
  igor.yml: |
    enabled: false
    skipLifeCycleManagement: true
  fiat.yml: |
    enabled: false
    skipLifeCycleManagement: true
  gate.yml: |
    host: 0.0.0.0
  deck.yml: |
    host: 0.0.0.0
    env:
      API_HOST: http://spin-gate.spinnaker:8084/
  config: |
    currentDeployment: default
    deploymentConfigurations:
    - name: default
      version: io-codelab
      providers:
        appengine:
          enabled: false
          accounts: []
        aws:
          enabled: false
          accounts: []
          defaultKeyPairTemplate: '{{name}}-keypair'
          defaultRegions:
          - name: us-west-2
          defaults:
            iamRole: BaseIAMRole
        azure:
          enabled: false
          accounts: []
          bakeryDefaults:
            templateFile: azure-linux.json
            baseImages: []
        dcos:
          enabled: false
          accounts: []
          clusters: []
        dockerRegistry:
          enabled: false
          accounts: []
        google:
          enabled: false
          accounts: []
          bakeryDefaults:
            templateFile: gce.json
            baseImages: []
            zone: us-central1-f
            network: default
            useInternalIp: false
        kubernetes:
          enabled: true
          accounts:
          - name: my-kubernetes-account
            requiredGroupMembership: []
            providerVersion: V2
            dockerRegistries: []
            configureImagePullSecrets: true
            serviceAccount: true
            namespaces: []
            omitNamespaces:
            - kube-system
            - spinnaker
            kinds: []
            omitKinds: []
            customResources: []
            oauthScopes: []
            oAuthScopes: []
          primaryAccount: my-kubernetes-account
        openstack:
          enabled: false
          accounts: []
          bakeryDefaults:
            baseImages: []
        oraclebmcs:
          enabled: false
          accounts: []
      deploymentEnvironment:
        size: SMALL
        type: Distributed
        accountName: my-kubernetes-account
        updateVersions: true
        consul:
          enabled: false
        vault:
          enabled: false
        customSizing: {}
        gitConfig:
          upstreamUser: spinnaker
      persistentStorage:
        persistentStoreType: gcs
        azs: {}
        gcs:
          project: {%PROJECT_ID%}
          bucket: {%BUCKET_NAME%}
          rootFolder: front50
        redis: {}
        s3: {}
        oraclebmcs: {}
      features:
        artifacts: true
        auth: false
        fiat: false
        chaos: false
        entityTags: false
        jobs: false
      metricStores:
        datadog:
          enabled: false
        prometheus:
          enabled: false
          add_source_metalabels: true
        stackdriver:
          enabled: false
        period: 30
        enabled: false
      notifications:
        slack:
          enabled: false
      timezone: America/Los_Angeles
      ci:
        jenkins:
          enabled: false
          masters: []
        travis:
          enabled: false
          masters: []
      security:
        apiSecurity:
          ssl:
            enabled: false
          overrideBaseUrl: /gate
        uiSecurity:
          ssl:
            enabled: false
        authn:
          oauth2:
            enabled: false
            client: {}
            resource: {}
            userInfoMapping: {}
          saml:
            enabled: false
          ldap:
            enabled: false
          x509:
            enabled: false
          enabled: false
        authz:
          groupMembership:
            service: EXTERNAL
            google:
              roleProviderType: GOOGLE
            github:
              roleProviderType: GITHUB
            file:
              roleProviderType: FILE
          enabled: false
      artifacts:
        gcs:
          enabled: true
          accounts:
          - name: {%SPIN_GCS_ACCOUNT%}
        github:
          enabled: false
          accounts: []
        http:
          enabled: false
          accounts: []
      pubsub:
        google:
          enabled: true
          subscriptions:
          - name: {%SPIN_GCS_PUB_SUB%}
            subscriptionName: {%GCS_SUB%}
            project: {%PROJECT_ID%}
            messageFormat: GCS
          - name: {%SPIN_GCR_PUB_SUB%}
            subscriptionName: {%GCR_SUB%}
            project: {%PROJECT_ID%}
            messageFormat: GCR

---

apiVersion: batch/v1
kind: Job
metadata:
  name: hal-deploy-apply
  namespace: spinnaker
  labels:
    app: job
    stack: hal-deploy
spec:
  template:
    metadata:
      labels:
        app: job
        stack: hal-deploy
    spec:
      restartPolicy: OnFailure
      containers:
      - name: hal-deploy-apply
        # todo use a custom image
        image: gcr.io/spinnaker-marketplace/halyard:0.43.0-20180321134158
        command:
        - /bin/sh
        args:
        - -c
        - "hal deploy apply --no-validate --daemon-endpoint http://spin-halyard.spinnaker:8064"
```

#### 2.3 Review the properties

```bash
#!/usr/bin/env bash

PROJECT_ID=$(gcloud info --format='value(config.project)')

GKE_CLUSTER=spin-demo
ZONE=us-west1-b

if [ -f bucket.txt ]; then
  BUCKET_NAME=$(cat bucket.txt)
else
  BUCKET_NAME="spin-gcs-bucket-$(cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 20 | head -n 1)-$(date +"%s")"
  echo $BUCKET_NAME > bucket.txt
fi

BUCKET_URI="gs://$BUCKET_NAME"

if [ -f account.txt ]; then
  SERVICE_ACCOUNT_NAME=$(cat account.txt)
else
  SERVICE_ACCOUNT_NAME="spin-acc-$(date +"%s")"
  echo $SERVICE_ACCOUNT_NAME > account.txt
fi

SERVICE_ACCOUNT_EMAIL="${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

GCS_TOPIC="spin-gcs-topic"
GCS_SUB="spin-gcs-sub"
GCR_TOPIC="projects/${PROJECT_ID}/topics/gcr"
GCR_SUB="spin-gcr-sub"

SPIN_GCS_ACCOUNT="my-gcs-account"

SPIN_GCS_PUB_SUB="my-gcs-pub-sub"
SPIN_GCR_PUB_SUB="my-gcr-pub-sub"
```

#### 2.4 Review the setup shell script

```bash
#!/usr/bin/env bash
bold() {
  echo ". $(tput bold)" "$*" "$(tput sgr0)";
}
err() {
  echo "$*" >&2;
}
source ./properties
if [ -z "$PROJECT_ID" ]; then
  err "Not running in a GCP project. Exiting."
  exit 1
fi
bold "Starting the setup process in project $PROJECT_ID..."
bold "Creating a service account $SERVICE_ACCOUNT_NAME..."
gcloud iam service-accounts create \
  $SERVICE_ACCOUNT_NAME \
  --display-name $SERVICE_ACCOUNT_NAME
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
  --role roles/owner
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
  --role roles/pubsub.subscriber
bold "Using bucket $BUCKET_URI..."
gsutil mb -p $PROJECT_ID $BUCKET_URI
bold "Configuring sample code & pipelines..."

gsutil cp gs://gke-spinnaker-codelab/services.tgz .
tar -xvzf services.tgz

gsutil cp gs://gke-spinnaker-codelab/front50.tgz .
tar -xvzf front50.tgz

replace() {
  find front50 -type f -name "*.json" -print0 | xargs -0 sed -i $1
  find services -type f -name "*" -print0 | xargs -0 sed -i $1
  sed -i $1 manifests.yml
}

replace 's|{%SPIN_GCS_ACCOUNT%}|'$SPIN_GCS_ACCOUNT'|g'
replace 's|{%SPIN_GCS_PUB_SUB%}|'$SPIN_GCS_PUB_SUB'|g'
replace 's|{%GCS_SUB%}|'$GCS_SUB'|g'
replace 's|{%GCR_SUB%}|'$GCR_SUB'|g'
replace 's|{%SPIN_GCR_PUB_SUB%}|'$SPIN_GCR_PUB_SUB'|g'
replace 's|{%PROJECT_ID%}|'$PROJECT_ID'|g'
replace 's|{%BUCKET_URI%}|'$BUCKET_URI'|g'
replace 's|{%BUCKET_NAME%}|'$BUCKET_NAME'|g'

bold "Configuring sample code & pipelines..."

gsutil cp -r services/manifests/frontend.yml $BUCKET_URI/manifests/frontend.yml
gsutil cp -r services/manifests/backend.yml $BUCKET_URI/manifests/backend.yml
gsutil cp -r front50 $BUCKET_URI

rm -rf front50/ # TODO remove front50.tgz as well

bold "Pushing sample images into gcr.io/$PROJECT_ID..."

gcloud auth configure-docker -q

gcloud docker -- pull gcr.io/spinnaker-marketplace/frontend
gcloud docker -- pull gcr.io/spinnaker-marketplace/backend

gcloud docker -- tag gcr.io/spinnaker-marketplace/frontend \
  gcr.io/$PROJECT_ID/frontend
gcloud docker -- tag gcr.io/spinnaker-marketplace/backend \
  gcr.io/$PROJECT_ID/backend

gcloud docker -- push gcr.io/$PROJECT_ID/frontend
gcloud docker -- push gcr.io/$PROJECT_ID/backend

bold "Configuring pub/sub from $GCS_TOPIC -> $GCS_SUB..."

gsutil notification create -t $GCS_TOPIC -f json $BUCKET_URI
gcloud pubsub subscriptions create $GCS_SUB --topic $GCS_TOPIC

bold "Configuring pub/sub for our docker builds..."

gcloud pubsub topics create projects/${PROJECT_ID}/topics/gcr
gcloud beta pubsub subscriptions create $GCR_SUB --topic $GCR_TOPIC

bold "Creating your cluster $GKE_CLUSTER..."

gcloud container clusters create $GKE_CLUSTER --zone $ZONE \
  --service-account $SERVICE_ACCOUNT_EMAIL \
  --username admin \
  --machine-type n1-standard-4 --image-type COS --disk-size 100 \
  --num-nodes 3 --enable-cloud-logging --enable-cloud-monitoring

gcloud container clusters get-credentials $GKE_CLUSTER --zone $ZONE

bold "Deploying spinnaker..."
kubectl apply -f manifests.yml

bold "Deploying sample service..."
kubectl apply -f services/manifests/seeding.yml
rm services/manifests/seeding.yml

bold "Waiting for spinnaker setup to complete (this might take some time)..."

job_ready() {
  kubectl get job $1 -n spinnaker -o jsonpath="{.status.succeeded}"
}

printf "Waiting on deployment to finish"
while [[ "$(job_ready hal-deploy-apply)" != "1" ]]; do
  printf "."
  sleep 5
done
echo ""

deploy_ready() {
  kubectl get deploy $1 -n spinnaker -o jsonpath="{.status.readyReplicas}"
}

printf "Waiting on API server to come online"
while [[ "$(deploy_ready spin-gate)" != "1" ]]; do
  printf "."
  sleep 5
done
echo ""

printf "Waiting on storage server to come online"
while [[ "$(deploy_ready spin-front50)" != "1" ]]; do
  printf "."
  sleep 5
done
echo ""

printf "Waiting on orchestration engine to come online"
while [[ "$(deploy_ready spin-orca)" != "1" ]]; do
  printf "."
  sleep 5
done
echo ""

bold "Ready! Run ./connect.sh to continue..."
```

#### 2.5 Run the setup shell script

```text
assembly_checkpoint@cloudshell:~ (inxome-10101)$ ./setup.sh
.  Starting the setup process in project inxome-10101... 
.  Creating a service account spin-acc-1558857530... 
Created service account [spin-acc-1558857530].


To take a quick anonymous survey, run:
  $ gcloud alpha survey

Updated IAM policy for project [inxome-10101].
bindings:
- members:
  - serviceAccount:716499896207@cloudbuild.gserviceaccount.com
  role: roles/cloudbuild.builds.builder
- members:
  - serviceAccount:service-716499896207@gcp-sa-cloudbuild.iam.gserviceaccount.com
  role: roles/cloudbuild.serviceAgent
- members:
  - serviceAccount:service-716499896207@compute-system.iam.gserviceaccount.com
  role: roles/compute.serviceAgent
- members:
  - serviceAccount:service-716499896207@container-engine-robot.iam.gserviceaccount.com
  role: roles/container.serviceAgent
- members:
  - serviceAccount:716499896207-compute@developer.gserviceaccount.com
  - serviceAccount:716499896207@cloudservices.gserviceaccount.com
  - serviceAccount:service-716499896207@containerregistry.iam.gserviceaccount.com
  role: roles/editor
- members:
  - serviceAccount:spin-acc-1558857530@inxome-10101.iam.gserviceaccount.com
  - user:assembly.checkpoint@gmail.com
  role: roles/owner
etag: BwWJxc2nTRw=
version: 1
Updated IAM policy for project [inxome-10101].
bindings:
- members:
  - serviceAccount:716499896207@cloudbuild.gserviceaccount.com
  role: roles/cloudbuild.builds.builder
- members:
  - serviceAccount:service-716499896207@gcp-sa-cloudbuild.iam.gserviceaccount.com
  role: roles/cloudbuild.serviceAgent
- members:
  - serviceAccount:service-716499896207@compute-system.iam.gserviceaccount.com
  role: roles/compute.serviceAgent
- members:
  - serviceAccount:service-716499896207@container-engine-robot.iam.gserviceaccount.com
  role: roles/container.serviceAgent
- members:
  - serviceAccount:716499896207-compute@developer.gserviceaccount.com
  - serviceAccount:716499896207@cloudservices.gserviceaccount.com
  - serviceAccount:service-716499896207@containerregistry.iam.gserviceaccount.com
  role: roles/editor
- members:
  - serviceAccount:spin-acc-1558857530@inxome-10101.iam.gserviceaccount.com
  - user:assembly.checkpoint@gmail.com
  role: roles/owner
- members:
  - serviceAccount:spin-acc-1558857530@inxome-10101.iam.gserviceaccount.com
  role: roles/pubsub.subscriber
etag: BwWJxc3XlQ0=
version: 1
.  Using bucket gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530... 
Creating gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/...
.  Configuring sample code & pipelines... 
Copying gs://gke-spinnaker-codelab/services.tgz...
/ [1 files][  6.5 KiB/  6.5 KiB]
Operation completed over 1 objects/6.5 KiB.
services/
services/LICENSE.txt
services/manifests/
services/manifests/frontend.yml
services/manifests/seeding.yml
services/manifests/update-backend.sh
services/manifests/update-frontend.sh
services/manifests/backend.yml
services/frontend/
services/frontend/Dockerfile
services/frontend/build.sh
services/frontend/check.sh
services/frontend/content/
services/frontend/content/index.html
services/frontend/main.go
services/frontend/get-ingress.sh
services/README.md
services/backend/
services/backend/Dockerfile
services/backend/build.sh
services/backend/main.go
Copying gs://gke-spinnaker-codelab/front50.tgz...
/ [1 files][  2.8 KiB/  2.8 KiB]
Operation completed over 1 objects/2.8 KiB.
front50/
front50/pipelines/
front50/pipelines/205a774a-2869-452a-9050-5fb95ae6624a/
front50/pipelines/205a774a-2869-452a-9050-5fb95ae6624a/specification.json
front50/pipelines/last-modified
front50/pipelines/f1d724be-7f75-43fc-b0f5-d7efa4b173af/
front50/pipelines/f1d724be-7f75-43fc-b0f5-d7efa4b173af/specification.json
front50/pipelines/349cbc61-72c9-4720-9c4a-f9fab60afece/
front50/pipelines/349cbc61-72c9-4720-9c4a-f9fab60afece/specification.json
front50/pipeline-strategies/
front50/pipeline-strategies/last-modified
front50/applications/
front50/applications/demo/
front50/applications/demo/specification.json
front50/applications/demo/permission.json
front50/applications/last-modified
.  Configuring sample code & pipelines... 
Copying file://services/manifests/frontend.yml [Content-Type=application/octet-stream]...
/ [1 files][  1.3 KiB/  1.3 KiB]
Operation completed over 1 objects/1.3 KiB.
Copying file://services/manifests/backend.yml [Content-Type=application/octet-stream]...
/ [1 files][  1.0 KiB/  1.0 KiB]
Operation completed over 1 objects/1.0 KiB.
Copying file://front50/applications/last-modified [Content-Type=application/octet-stream]...
Copying file://front50/applications/demo/specification.json [Content-Type=application/json]...
Copying file://front50/applications/demo/permission.json [Content-Type=application/json]...
Copying file://front50/pipelines/last-modified [Content-Type=application/octet-stream]...
\ [4 files][  278.0 B/  278.0 B]
==> NOTE: You are performing a sequence of gsutil operations that may
run significantly faster if you instead use gsutil -m cp ... Please
see the -m section under "gsutil help options" for further information
about when gsutil -m can be advantageous.

Copying file://front50/pipelines/f1d724be-7f75-43fc-b0f5-d7efa4b173af/specification.json [Content-Type=application/json]...
Copying file://front50/pipelines/349cbc61-72c9-4720-9c4a-f9fab60afece/specification.json [Content-Type=application/json]...
Copying file://front50/pipelines/205a774a-2869-452a-9050-5fb95ae6624a/specification.json [Content-Type=application/json]...
Copying file://front50/pipeline-strategies/last-modified [Content-Type=application/octet-stream]...
/ [8 files][ 15.1 KiB/ 15.1 KiB]
Operation completed over 8 objects/15.1 KiB.
.  Pushing sample images into gcr.io/inxome-10101... 
WARNING: Your config file at [/home/assembly_checkpoint/.docker/config.json] contains these credential helper entries:

{
  "credHelpers": {
    "gcr.io": "gcr",
    "us.gcr.io": "gcr",
    "asia.gcr.io": "gcr",
    "staging-k8s.gcr.io": "gcr",
    "eu.gcr.io": "gcr"
  }
}
These will be overwritten.
Docker configuration file updated.
WARNING: `gcloud docker` will not be supported for Docker client versions above 18.03.

As an alternative, use `gcloud auth configure-docker` to configure `docker` to
use `gcloud` as a credential helper, then use `docker` as you would for non-GCR
registries, e.g. `docker pull gcr.io/project-id/my-image`. Add
`--verbosity=error` to silence this warning: `gcloud docker
--verbosity=error -- pull gcr.io/project-id/my-image`.

See: https://cloud.google.com/container-registry/docs/support/deprecation-notices#gcloud-docker

Using default tag: latest
latest: Pulling from spinnaker-marketplace/frontend
c73ab1c6897b: Pull complete
1ab373b3deae: Pull complete
b542772b4177: Pull complete
57c8de432dbe: Pull complete
c81227e1ec90: Pull complete
3e5d5c68d1eb: Pull complete
a4ca93091ca5: Pull complete
f97b05cce426: Pull complete
8c35a849bc90: Pull complete
0b3fbcc3572a: Pull complete
Digest: sha256:ad45afc08876ea832d17a7924198605dbd86715bcea5d2d5281363fba29651e4
Status: Downloaded newer image for gcr.io/spinnaker-marketplace/frontend:latest
WARNING: `gcloud docker` will not be supported for Docker client versions above 18.03.

As an alternative, use `gcloud auth configure-docker` to configure `docker` to
use `gcloud` as a credential helper, then use `docker` as you would for non-GCR
registries, e.g. `docker pull gcr.io/project-id/my-image`. Add
`--verbosity=error` to silence this warning: `gcloud docker
--verbosity=error -- pull gcr.io/project-id/my-image`.

See: https://cloud.google.com/container-registry/docs/support/deprecation-notices#gcloud-docker

Using default tag: latest
latest: Pulling from spinnaker-marketplace/backend
c73ab1c6897b: Already exists
1ab373b3deae: Already exists
b542772b4177: Already exists
57c8de432dbe: Already exists
c81227e1ec90: Already exists
3e5d5c68d1eb: Already exists
a4ca93091ca5: Already exists
302934d533f7: Pull complete
abd9bcbd139a: Pull complete
Digest: sha256:39b386391bba5fa58077956ed858cb7bdbfe1118aab2b27d73ef9c94ddbc5f07
Status: Downloaded newer image for gcr.io/spinnaker-marketplace/backend:latest
WARNING: `gcloud docker` will not be supported for Docker client versions above 18.03.

As an alternative, use `gcloud auth configure-docker` to configure `docker` to
use `gcloud` as a credential helper, then use `docker` as you would for non-GCR
registries, e.g. `docker pull gcr.io/project-id/my-image`. Add
`--verbosity=error` to silence this warning: `gcloud docker
--verbosity=error -- pull gcr.io/project-id/my-image`.

See: https://cloud.google.com/container-registry/docs/support/deprecation-notices#gcloud-docker

WARNING: `gcloud docker` will not be supported for Docker client versions above 18.03.

As an alternative, use `gcloud auth configure-docker` to configure `docker` to
use `gcloud` as a credential helper, then use `docker` as you would for non-GCR
registries, e.g. `docker pull gcr.io/project-id/my-image`. Add
`--verbosity=error` to silence this warning: `gcloud docker
--verbosity=error -- pull gcr.io/project-id/my-image`.

See: https://cloud.google.com/container-registry/docs/support/deprecation-notices#gcloud-docker

WARNING: `gcloud docker` will not be supported for Docker client versions above 18.03.

As an alternative, use `gcloud auth configure-docker` to configure `docker` to
use `gcloud` as a credential helper, then use `docker` as you would for non-GCR
registries, e.g. `docker pull gcr.io/project-id/my-image`. Add
`--verbosity=error` to silence this warning: `gcloud docker
--verbosity=error -- pull gcr.io/project-id/my-image`.

See: https://cloud.google.com/container-registry/docs/support/deprecation-notices#gcloud-docker

The push refers to repository [gcr.io/inxome-10101/frontend]
ddfa314bd442: Mounted from spinnaker-marketplace/frontend
ee25a991aff0: Mounted from spinnaker-marketplace/frontend
b70fddb9ad46: Mounted from spinnaker-marketplace/frontend
b7a313b78c0b: Mounted from spinnaker-marketplace/backend
27c2bda9e357: Mounted from spinnaker-marketplace/backend
e33f0ab4ff77: Mounted from spinnaker-marketplace/backend
20c527f217db: Mounted from spinnaker-marketplace/backend
61c06e07759a: Mounted from spinnaker-marketplace/backend
bcbe43405751: Mounted from spinnaker-marketplace/backend
e1df5dc88d2c: Mounted from spinnaker-marketplace/backend
latest: digest: sha256:ad45afc08876ea832d17a7924198605dbd86715bcea5d2d5281363fba29651e4 size: 2422
WARNING: `gcloud docker` will not be supported for Docker client versions above 18.03.

As an alternative, use `gcloud auth configure-docker` to configure `docker` to
use `gcloud` as a credential helper, then use `docker` as you would for non-GCR
registries, e.g. `docker pull gcr.io/project-id/my-image`. Add
`--verbosity=error` to silence this warning: `gcloud docker
--verbosity=error -- pull gcr.io/project-id/my-image`.

See: https://cloud.google.com/container-registry/docs/support/deprecation-notices#gcloud-docker

The push refers to repository [gcr.io/inxome-10101/backend]
f461ae32ef3d: Mounted from spinnaker-marketplace/backend
535a51256805: Mounted from spinnaker-marketplace/backend
b7a313b78c0b: Layer already exists
27c2bda9e357: Layer already exists
e33f0ab4ff77: Layer already exists
20c527f217db: Layer already exists
61c06e07759a: Layer already exists
bcbe43405751: Layer already exists
e1df5dc88d2c: Layer already exists
latest: digest: sha256:39b386391bba5fa58077956ed858cb7bdbfe1118aab2b27d73ef9c94ddbc5f07 size: 2214
.  Configuring pub/sub from spin-gcs-topic -> spin-gcs-sub... 
Created Cloud Pub/Sub topic projects/inxome-10101/topics/spin-gcs-topic
Created notification config projects/_/buckets/spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/notificationConfigs/1
Created subscription [projects/inxome-10101/subscriptions/spin-gcs-sub].
.  Configuring pub/sub for our docker builds... 
Created topic [projects/inxome-10101/topics/gcr].
Created subscription [projects/inxome-10101/subscriptions/spin-gcr-sub].
.  Creating your cluster spin-demo... 
WARNING: In June 2019, node auto-upgrade will be enabled by default for newly created clusters and node pools. To disable it, use the `--no-enable-autoupgrade` flag.
WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate using the `--[no-]issue-client-certificate` flag.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-enable-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cluster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
WARNING: Your Pod address range (`--cluster-ipv4-cidr`) can accommodate at most 1008 node(s).
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepairs.
Creating cluster spin-demo in us-west1-b... Cluster is being health-checked (master is healthy)...done.
Created [https://container.googleapis.com/v1/projects/inxome-10101/zones/us-west1-b/clusters/spin-demo].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-west1-b/spin-demo?project=inxome-10101
kubeconfig entry generated for spin-demo.
NAME       LOCATION    MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
spin-demo  us-west1-b  1.12.7-gke.10   35.203.143.106  n1-standard-4  1.12.7-gke.10  3          RUNNING
Fetching cluster endpoint and auth data.
kubeconfig entry generated for spin-demo.
.  Deploying spinnaker... 
namespace/spinnaker created
clusterrolebinding.rbac.authorization.k8s.io/spinnaker-admin created
deployment.apps/spin-halyard created
service/spin-halyard created
configmap/halconfig created
job.batch/hal-deploy-apply created
.  Deploying sample service... 
namespace/staging created
configmap/frontend-config created
deployment.apps/frontend-primary created
service/frontend created
deployment.apps/backend-primary created
service/backend created
namespace/production created
configmap/frontend-config created
deployment.apps/frontend-primary created
service/frontend created
deployment.apps/backend-primary created
service/backend created
.  Waiting for spinnaker setup to complete (this might take some time)... 
Waiting on deployment to finish...............................
Waiting on API server to come online..........................
Waiting on storage server to come online
Waiting on orchestration engine to come online
.  Ready! Run ./connect.sh to continue... 
```

#### 2.6 Review the service files

- [services/manifests/frontend.yml](services-manifests-frontend.yml)
- [services/manifests/seeding.yml](services-manifests-seeding.yml)
- [services/manifests/update-backend.sh](services-manifests-update-backend.sh)
- [services/manifests/update-frontend.sh](services-manifests-update-frontend.sh)
- [services/manifests/backend.yml](services-manifests-backend.yml)
- [services/frontend/Dockerfile](services-frontend.Dockerfile)
- [services/frontend/build.sh](services-frontend-build.sh)
- [services/frontend/check.sh](services-frontend-check.sh)
- [services/frontend/content/index.html](services-frontend-content-index.html)
- [services/frontend/main.go](services-frontend-main.go)
- [services/frontend/get-ingress.sh](services-frontend-get-ingress.sh)
- [services/backend/Dockerfile](services-backend.Dockerfile)
- [services/backend/build.sh](services-backend-build.sh)
- [services/backend/main.go](services-backend-main.go)

#### 2.7 Review the frontend files

- [front50/pipelines/205a774a-2869-452a-9050-5fb95ae6624a/specification.json](front50-pipelines-205a774a-2869-452a-9050-5fb95ae6624a-specification.json)
- [front50/pipelines/f1d724be-7f75-43fc-b0f5-d7efa4b173af/specification.json](front50-pipelines-f1d724be-7f75-43fc-b0f5-d7efa4b173af-specification.json)
- [front50/pipelines/349cbc61-72c9-4720-9c4a-f9fab60afece/specification.json](front50-pipelines-349cbc61-72c9-4720-9c4a-f9fab60afece-specification.json)
- [front50/applications/demo/specification.json](front50-applications-demo-specification.json)
- [front50/applications/demo/permission.json](front50-applications-demo-permission.json)

#### 2.8 Review the connect shell script

```bash
#!/usr/bin/env bash

PORT=8080
DECK_POD=$(kubectl get po -n spinnaker -l "cluster=spin-deck" \
  -o jsonpath="{.items[0].metadata.name}")

EXISTING_PID=$(sudo netstat -nlp | grep $PORT | awk '{print $7}' | cut -f1 -d '/')

if [ -n "$EXISTING_PID" ]; then
  echo "PID $EXISTING_PID already listening... restarting port-forward"
  kill $EXISTING_PID
  sleep 5
fi

kubectl port-forward $DECK_POD $PORT:9000 -n spinnaker >> /dev/null &

echo "Port opened on $PORT"
```

#### 2.9 Run the connect shell script

```text
assembly_checkpoint@cloudshell:~ (inxome-10101)$ ./connect.sh
Port opened on 8080
```

#### 2.10 Open port 8080

![Port][Fig2]

#### 2.11 Spinnaker is now ready

![FrontPage][Fig3]

***

### 3. Explore the Spinnaker UI

As a part of the automated setup, a copy of the sample application as well as configured some Spinnaker pipelines to manage its lifecycle have been deployed. Let's navigate to this application by clicking on the "Applications" tab:

![Application][Fig4]

#### 3.1 From Applications click on demo

![Demo][Fig5]

#### 3.2 Pipeline

While the Clusters tab gives ad-hoc actions to perform, and information about the state of your running applications, the Pipelines tab lets one configure repeatable, automated processes to update your running code.

Three pipelines were configured to run in order:

Deploy to Staging: This deploys your Kubernetes resources and built Docker images to the staging environment, and runs an integration test against the running backend service when it is ready to receive traffic.
Deploy Simple Canary in Production: This takes the Kubernetes resources and Docker images from staging and deploys them to receive a small fraction of traffic in the production environment.
Promote Canary to Production: This gives you a chance to validate your Canary in production, and if you want, promote it to receive all production traffic. Once this is completed, the Canary is deleted.

To get a sense of what these pipelines do before we automatically trigger them, select the Configure dropdown, and pick the Deploy to Staging pipeline.

![Pipeline][Fig6]

***

### 4. Explore the Spinnaker Code

The sample application has a frontend and a backend. They communicate like this:

![AppStructure][Fig7]

Every time a user makes a request, the frontend serves some static content along with some information about the backend that served the request. Both the frontend and backend are managed by [Deployments][6], have multiple replicas ([Pods][7]), and are fronted by a load balancer ([Service][8]). We'll use this to demonstrate how to update these two services independently using shared pipelines in Spinnaker.

#### 4.1 Return to the cloud shell and examine the frontend service

```text
assembly_checkpoint@cloudshell:~ (inxome-10101)$ ls -l
total 60
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint   20 May 26 16:58 account.txt
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint   48 May 26 16:58 bucket.txt
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  696 Apr  3 05:45 cleanup.sh
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  461 Apr  3 05:45 connect.sh
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 2857 May 26 16:59 front50.tgz
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 4033 May 26 16:47 install.tgz
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 7155 May 26 16:59 manifests.yml
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint  842 Apr  3 05:48 properties
drwxr-xr-x 5 assembly_checkpoint assembly_checkpoint 4096 May 26 16:59 services
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 6646 May 26 16:59 services.tgz
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint 4078 Apr  3 06:07 setup.sh
assembly_checkpoint@cloudshell:~ (inxome-10101)$ cd services/
assembly_checkpoint@cloudshell:~/services (inxome-10101)$ ls -l
total 28
drwxr-xr-x 2 assembly_checkpoint assembly_checkpoint  4096 May 26 16:59 backend
drwxr-xr-x 3 assembly_checkpoint assembly_checkpoint  4096 May 26 16:59 frontend
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 11342 May 26 16:59 LICENSE.txt
drwxr-xr-x 2 assembly_checkpoint assembly_checkpoint  4096 May 26 17:04 manifests
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint   227 May 26 16:59 README.md
assembly_checkpoint@cloudshell:~/services (inxome-10101)$ cd frontend/
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ ls -l
total 24
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint   82 May 26 16:59 build.sh
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  102 May 26 16:59 check.sh
drwxr-xr-x 2 assembly_checkpoint assembly_checkpoint 4096 May 26 16:59 content
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint  150 May 26 16:59 Dockerfile
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  120 May 26 16:59 get-ingress.sh
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 1206 May 26 16:59 main.go
```

Because this service is already deployed in the cluster, one can take a look at what it is currently configured to serve. 

#### 4.2 Grab the ingress IP address using:

```text
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ pwd
/home/assembly_checkpoint/services/frontend
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ ./get-ingress.sh
34.83.229.135
```

#### 4.3 Copy the IP address, and open it in the browser.

![HelloWorld][Fig8]

***

### 5. Canary a Frontend Code Change

In the ```~/services/frontend/``` folder, make an edit to ```content/index.html```. We recommend changing ```style="background-color:blue"``` to white to make the update easy to see.

#### 5.1 Review the frontend build shell script

```bash
#!/usr/bin/env bash

gcloud builds submit -q --tag gcr.io/inxome-10101/frontend .
```

#### 5.2 Update and rebuild the frontend with the build shell script

```text
assembly_checkpoint@cloudshell:~/services (inxome-10101)$ cd frontend/
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ ls -l
total 24
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint   82 May 26 16:59 build.sh
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  102 May 26 16:59 check.sh
drwxr-xr-x 2 assembly_checkpoint assembly_checkpoint 4096 May 26 16:59 content
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint  150 May 26 16:59 Dockerfile
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  120 May 26 16:59 get-ingress.sh
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 1206 May 26 16:59 main.go
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ pwd
/home/assembly_checkpoint/services/frontend
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ ./get-ingress.sh
34.83.229.135
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ pwd
/home/assembly_checkpoint/services/frontend
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ ls -l content/
total 4
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 192 May 26 16:59 index.html
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ cat content/index.html
<!DOCTYPE html>
<html>
  <body style="background-color:blue">
    <h2>Hello, world!</h2>
    <p>Message from the backend:</p>
    <p>{{.Message}}</p>
    <p>{{.Feature}}</p>
  </body>
</html>
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ sed 's/background-color:blue/background-color:white/g' -i content/index.html
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ cat content/index.html 
<!DOCTYPE html>
<html>
  <body style="background-color:white">
    <h2>Hello, world!</h2>
    <p>Message from the backend:</p>
    <p>{{.Message}}</p>
    <p>{{.Feature}}</p>
  </body>
</html>
assembly_checkpoint@cloudshell:~/services/frontend (inxome-10101)$ ./build.sh
Creating temporary tarball archive of 6 file(s) totalling 1.8 KiB before compression.
Uploading tarball of [.] to [gs://inxome-10101_cloudbuild/source/1558862888.61-1b1147efec3a4cffb361e2ff701c5eac.tgz]
Created [https://cloudbuild.googleapis.com/v1/projects/inxome-10101/builds/78a6578d-9f42-4944-b27e-8ab7c7284640].
Logs are available at [https://console.cloud.google.com/gcr/builds/78a6578d-9f42-4944-b27e-8ab7c7284640?project=716499896207].
------------------------------------------------------------------------ REMOTE BUILD OUTPUT ------------------------------------------------------------------------
starting build "78a6578d-9f42-4944-b27e-8ab7c7284640"

FETCHSOURCE
Fetching storage object: gs://inxome-10101_cloudbuild/source/1558862888.61-1b1147efec3a4cffb361e2ff701c5eac.tgz#1558862890845492
Copying gs://inxome-10101_cloudbuild/source/1558862888.61-1b1147efec3a4cffb361e2ff701c5eac.tgz#1558862890845492...
/ [1 files][  1.2 KiB/  1.2 KiB]
Operation completed over 1 objects/1.2 KiB.
BUILD
Already have image (with digest): gcr.io/cloud-builders/docker
Sending build context to Docker daemon  8.704kB
Step 1/5 : FROM golang
latest: Pulling from library/golang
Digest: sha256:dfec1cb2f37c4dbfb369dac1fe1aa26aad1c516260ca8ed76363b30fed1e9125
Status: Downloaded newer image for golang:latest
 ---> 7ced090ee82e
Step 2/5 : ADD . /go/src/spinnaker.io/demo/k8s-demo
 ---> d83aa5a63ab3
Step 3/5 : RUN go install spinnaker.io/demo/k8s-demo
 ---> Running in 8930c455da3d
Removing intermediate container 8930c455da3d
 ---> 629ff274644b
Step 4/5 : ADD ./content /content
 ---> 3262d3531db4
Step 5/5 : ENTRYPOINT /go/bin/k8s-demo
 ---> Running in 3362429aac1e
Removing intermediate container 3362429aac1e
 ---> 8f6613b8cca5
Successfully built 8f6613b8cca5
Successfully tagged gcr.io/inxome-10101/frontend:latest
PUSH
Pushing gcr.io/inxome-10101/frontend
The push refers to repository [gcr.io/inxome-10101/frontend]
3bf00504e716: Preparing
6fb559436a34: Preparing
a28d23002c60: Preparing
d3c12cc41aa1: Preparing
16f63bc6cca8: Preparing
4879ce009c87: Preparing
4230ff7f2288: Preparing
2c719774c1e1: Preparing
ec62f19bb3aa: Preparing
f94641f1fe1f: Preparing
4879ce009c87: Waiting
4230ff7f2288: Waiting
2c719774c1e1: Waiting
ec62f19bb3aa: Waiting
f94641f1fe1f: Waiting
16f63bc6cca8: Layer already exists
d3c12cc41aa1: Layer already exists
4879ce009c87: Layer already exists
4230ff7f2288: Layer already exists
ec62f19bb3aa: Layer already exists
2c719774c1e1: Layer already exists
f94641f1fe1f: Layer already exists
a28d23002c60: Pushed
3bf00504e716: Pushed
6fb559436a34: Pushed
latest: digest: sha256:37df62f94dd8da234e9d0213ccfb665f261f04d0e35feb7bf0ff973b6e60c87c size: 2422
DONE
---------------------------------------------------------------------------------------------------------------------------------------------------------------------

ID                                    CREATE_TIME                DURATION  SOURCE                                                                                  IMAGES                                  STATUS
78a6578d-9f42-4944-b27e-8ab7c7284640  2019-05-26T09:28:11+00:00  21S       gs://inxome-10101_cloudbuild/source/1558862888.61-1b1147efec3a4cffb361e2ff701c5eac.tgz  gcr.io/inxome-10101/frontend (+1 more)  SUCCESS
```

This will run for a few minutes in Google Container Builder, and will push an updated image to the project's container registry. This event will kick-off the Deploy to Staging pipeline.

#### 5.4 Monitor the "Deploy to Staging" pipeline

When the build have kicked off completes, the Deploy to Staging pipeline automatically starts running. Return to the Spinnaker window and open the Pipelines tab:

![Staging][Fig9]

Once the last pipeline stage turns orange, click on the "Person" icon and "Continue" to approve and complete the pipeline. If desired, you can follow the custom instructions shown on the stage.

![Jugement][Fig10]

#### 5.5 Monitor "Deploy Simple Canary to Production" pipeline

Once you approve and the Deploy to Staging pipeline completes, you will automatically have a canary deployed to production by the Deploy Simple Canary to Production pipeline. You can quickly check that it was deployed on the Clusters tab as shown here:

![InProd][Fig13]

At this point, one can run ```~/services/frontend/get-ingress.sh``` to get the IP address of the frontend production service, and open it in a new tab. If you refresh the page frequently, the color should alternate between white and blue:

![NewFrontend][Fig14]

Note: This canary strategy is extremely rudimentary for the purpose of keeping the codelab simple. Spinnaker features a fully-featured canary analysis engine that monitors pairs of user-configured time-series metrics to automatically judge whether a canary deployment is healthy. This is the preferred way to do canary deployments in Spinnaker, but generally requires at least 30 minutes of data collection.

#### 5.6 Monitor "Promote Canary to Production" pipeline

After the Deploy Simple Canary to Production pipeline completes, the canary will be promoted to production.

![CanaryToProd][Fig11]

Notice, on the Clusters tab the frontend service in staging has been updated. And now points to the digest of the image just created in the build above.

![NewBuild][Fig12]

Using the same ```~/services/frontend/get-ingress.sh``` script from above to get the IP address of the frontend production service, one will find out that the background is now always white.

***

### 6. Canary a Config Change

Similar to how we checked in a code change to build a Docker image for the frontend service, we will now update the ConfigMap that the frontend service loads to enable a feature in our application's landing page.

#### 6.1 Edit the frontend ConfigMap

Open the ```~/services/manifests/``` directory, and edit the ```frontend.yml``` file. Edit the ```kind: ConfigMap``` block, add a ```FEATURE: "A new feature!"``` entry under ```data```.

```text
assembly_checkpoint@cloudshell:~/services/manifests (inxome-10101)$ pwd
/home/assembly_checkpoint/services/manifests
assembly_checkpoint@cloudshell:~/services/manifests (inxome-10101)$ ls -l
total 16
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 1067 May 26 16:59 backend.yml
-rw-r--r-- 1 assembly_checkpoint assembly_checkpoint 1327 May 26 16:59 frontend.yml
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  118 May 26 16:59 update-backend.sh
-rwxr-xr-x 1 assembly_checkpoint assembly_checkpoint  120 May 26 16:59 update-frontend.sh
assembly_checkpoint@cloudshell:~/services/manifests (inxome-10101)$ cat frontend.yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: '${ namespace }'
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: '${ namespace }'
data:
  BACKEND_ENDPOINT: 'http://backend.${ namespace }'
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: 'frontend-${ canary ? "canary" : "primary" }'
  namespace: '${ namespace }'
  labels:
    stack: frontend
    tier: '${ namespace }'
    canary: '${ canary ? "true" : "false" }'
spec:
---
  replicas: '${ #toInt( canary ? 1 : 5 ) }'
  selector:
    matchLabels:
      stack: frontend
      tier: '${ namespace }'
      canary: '${ canary ? "true" : "false" }'
  template:
    metadata:
      labels:
        stack: frontend
        tier: '${ namespace }'
        canary: '${ canary ? "true" : "false" }'
    spec:
      containers:
      - name: primary
        image: gcr.io/inxome-10101/frontend
        ports:
        - containerPort: 8000
        readinessProbe:
          httpGet:
            path: /
            port: 8000
        envFrom:
        - configMapRef:
            name: frontend-config
---
kind: Service
apiVersion: v1
metadata:
  name: frontend
  namespace: '${ namespace }'
spec:
  selector:
    stack: frontend
    tier: '${ namespace }'
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
assembly_checkpoint@cloudshell:~/services/manifests (inxome-10101)$ vim frontend.yml
assembly_checkpoint@cloudshell:~/services/manifests (inxome-10101)$ grep FEATURE frontend.yml
  FEATURE: 'A new feature!'
```

#### 6.2 Review the update frontend shell script

```bash
#!/usr/bin/env bash

gsutil cp frontend.yml gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/manifests/frontend.yml
```

#### 6.3 Run the script

```text
assembly_checkpoint@cloudshell:~/services/manifests (inxome-10101)$ ./update-frontend.sh
Copying file://frontend.yml [Content-Type=application/octet-stream]...
/ [1 files][  1.3 KiB/  1.3 KiB]
Operation completed over 1 objects/1.3 KiB.
```

The updated manifest will automatically push a message to Spinnaker via [Pub/Sub][3].

#### 6.4 Check the "Deploy to Staging" pipeline

![ConfigDeployment][Fig15]

In particular, notice that this deployment has assigned a version to the ConfigMap that we edited, giving it the name frontend-config-v001. In the prior deployment, we deployed frontend-config-v000. This was done automatically by Spinnaker to ensure that only this deployment of your frontend is affected by this config change.

#### 6.5 Push the change to production

In the same way we updated our frontend docker image, allow the three (Deploy to Staging, Deploy Simple Canary to Production, and Promote Canary To Production) pipelines to finish running, accepting the Manual Judgements along the way.

![ConfigInProd][Fig16]

Now the feature "A new feature!" added to the config manifest can be seen by reloading the front page.

![NewFeatureOnFrontend][Fig17]

***

### 7. Configure Automated Rollbacks

Let's create a pipeline that codifies the rollback policy, and allows you to rollback production at the click of a button.

#### 7.1 Use the Spinnaker UI to create the rollback pipeline

Fallow [section 7][Sec7] of this CodeLab.

![Rolleback][Fig18]

Spinnaker increments version on rollbacks.

![NewFeatureOnFrontend][Fig19]

We can see now that the feature is no longer in the front page.

![NoFeature][Fig20]

***

### 8. Cleanup

Simply run the ```~/cleanup.sh``` script in the home directory to delete the resources created by this codelab.

#### 8.1 Review the cleanup script

```bash
#!/usr/bin/env bash

bold() {
  printf "$(tput bold)" "$*" "$(tput sgr0)"
}

err() {
  printf "%s\n" "$*" >&2;
}

source ./properties

if [ -z "$PROJECT_ID" ]; then
  err "Not running in a GCP project. Exiting."
  exit 1
fi

SA_EMAIL=$(gcloud iam service-accounts list \
  --filter="displayName:$SERVICE_ACCOUNT_NAME" \
  --format='value(email)')

gcloud iam service-accounts delete $SA_EMAIL -q

rm account.txt

gcloud container clusters delete $GKE_CLUSTER --zone $ZONE -q

gsutil rm -r $BUCKET_URI

rm bucket.txt

gcloud pubsub subscriptions delete $GCS_SUB -q
gcloud pubsub topics delete $GCS_TOPIC -q

gcloud pubsub subscriptions delete $GCR_SUB -q
gcloud pubsub topics delete $GCR_TOPIC -q
```

#### 8.2 Run the cleanup script

```text
assembly_checkpoint@cloudshell:~ (inxome-10101)$ ./cleanup.sh
deleted service account [spin-acc-1558857530@inxome-10101.iam.gserviceaccount.com]
Deleting cluster spin-demo...⠛E0526 20:53:01.565099    3251 portforward.go:178] lost connection to pod
Deleting cluster spin-demo...done.
Deleted [https://container.googleapis.com/v1/projects/inxome-10101/zones/us-west1-b/clusters/spin-demo].
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/applications/demo/permission.json#1558857559258173...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/applications/demo/specification.json#1558857558847057...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/applications/last-modified#1558857558403077...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/pipeline-strategies/last-modified#1558857561552490...
/ [4 objects]
==> NOTE: You are performing a sequence of gsutil operations that may
run significantly faster if you instead use gsutil -m rm ... Please
see the -m section under "gsutil help options" for further information
about when gsutil -m can be advantageous.

Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/pipelines/205a774a-2869-452a-9050-5fb95ae6624a/specification.json#1558857561088177...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/pipelines/349cbc61-72c9-4720-9c4a-f9fab60afece/specification.json#1558857560590302...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/pipelines/88ba064c-b699-4fde-96e8-c48e420f312b/specification.json#1558869674756449...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/pipelines/f1d724be-7f75-43fc-b0f5-d7efa4b173af/specification.json#1558857560203414...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/pipelines/last-modified#1558857559676038...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/front50/projects/last-modified#1558871077156537...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/manifests/backend.yml#1558857555060615...
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/manifests/frontend.yml#1558866543576344...
/ [12 objects]
Operation completed over 12 objects.
Removing gs://spin-gcs-bucket-bx9kevep2yxq92j2k0la-1558857530/...
Deleted subscription [projects/inxome-10101/subscriptions/spin-gcs-sub].
Deleted topic [projects/inxome-10101/topics/spin-gcs-topic].
Deleted subscription [projects/inxome-10101/subscriptions/spin-gcr-sub].
Deleted topic [projects/inxome-10101/topics/gcr].
```

***

### Bibliography

1) [Continuous Delivery to Kubernetes Using Spinnaker CodeLab.][1]
2) [Spinnaker, Continuous Delivery for Enterprise, fast, safe, repeatable deployments.][2]
3) [Cloud Pub/Sub API][3]
4) [Cloud Build API][4]
5) [Kubernetes Engine API][5]
6) [What is a Kubernetes deployment?][6]
7) [What is a Kubernetes pod?][7]
8) [What is a Kubernetes service?][8]

[1]: https://codelabs.developers.google.com/codelabs/cloud-spinnaker-kubernetes-cd/index.html?index=..%2F..index#0
[2]: https://www.spinnaker.io/
[3]: https://console.cloud.google.com/apis/api/pubsub.googleapis.com/overview?project=inxome-10101&folder&organizationId
[4]: https://console.cloud.google.com/apis/api/cloudbuild.googleapis.com/overview?project=inxome-10101&folder&organizationId
[5]: https://console.cloud.google.com/apis/api/container.googleapis.com/overview?project=inxome-10101&folder&organizationId
[6]: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
[7]: https://kubernetes.io/docs/concepts/workloads/pods/pod/
[8]: https://kubernetes.io/docs/concepts/services-networking/service/

[inXomeImage]: ../../../inxome.png
[report]: https://github.com/inxome/project/tree/master/report "Report"

[Sec7]: https://codelabs.developers.google.com/codelabs/cloud-spinnaker-kubernetes-cd/index.html?index=..%2F..index#6
[Fig1]: codelab.png
[Fig2]: port.png
[Fig3]: frontpage.png
[Fig4]: applications.png
[Fig5]: demo.png
[Fig6]: pipeline.png
[Fig7]: app_structure.png
[Fig8]: hello_world.png
[Fig9]: staging.png
[Fig10]: jugement.png
[Fig11]: canary_to_prod.png
[Fig12]: new_build.png
[Fig13]: in_prod.png
[Fig14]: new_frontend.png
[Fig15]: config_depo.png
[Fig16]: new_config_in_prod.png
[Fig17]: feature.png
[Fig18]: rolleback.png
[Fig19]: v004.png
[Fig20]: no_feature.png
