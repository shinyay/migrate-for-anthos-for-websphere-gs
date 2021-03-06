# Migrate for Anthos Getting Started for WebSphere

WebSphere Application Server (WAS) traditional is a platform for deploying apps in a Java Enterprise Edition (Java EE) based runtime environment. Migrate for Anthos lets you modernize app workloads running in WAS traditional by converting them to application containers

## Description

## Demo

## Features

- feature:1
- feature:2

## Requirement

## Usage
### Prepare WebSphere Application Server Migration Toolkit for Application Binaries
Download [Migration Toolkit for Application Binaries](https://www.ibm.com/support/pages/migration-toolkit-application-binaries)

Extract toolkit
```
$ java -jar binaryAppScannerInstaller.jar --acceptLicense --verbose

Target directory for product files? ./tool
```

Upload Migration Toolkit
```
$ cd tool/wamt/
$ gsutil mb gs://(gcloud config get-value project)-migration-artifacts
$ gsutil cp binaryAppScanner.jar gs://(gcloud config get-value project)-migration-artifacts
```

### Configure Service Account

Enable required services
```
$ gcloud services enable \
    servicemanagement.googleapis.com \
    servicecontrol.googleapis.com \
    cloudresourcemanager.googleapis.com \
    container.googleapis.com \
    compute.googleapis.com \
    containerregistry.googleapis.com
```

|Name|Title|
|----|-----|
|servicemanagement.googleapis.com|Service Management API|
|servicecontrol.googleapis.com|Service Control API|
|cloudresourcemanager.googleapis.com|Cloud Resource Manager API|
|compute.googleapis.com	Compute|Engine API|
|container.googleapis.com|Kubernetes Engine API|
|containerregistry.googleapis.com|Google Container Registry API|

#### Service Account for Artifacts by M4A

Create Service Account for Artifacts by M4A
```
$ gcloud iam service-accounts create m4a-process
```

Bind Service Account to Role
```
$ gcloud projects add-iam-policy-binding (gcloud config get-value project) \
    --member=serviceAccount:m4a-process@(gcloud config get-value project).iam.gserviceaccount.com \
    --role=roles/storage.admin
```

Retrieve Key file of Service Account
- `m4a-process-sa.json`: Service Account Key File

```
$ gcloud iam service-accounts keys create m4a-process-sa.json \
    --iam-account=m4a-process@(gcloud config get-value project).iam.gserviceaccount.com \
    --project (gcloud config get-value project)
```

#### Service Account for Target Source by M4A

Create Service Account for Target Source on Compute Engine
```
$ gcloud iam service-accounts create m4a-source
```

Bind Service Account to the following Roles
- `roles/compute.viewer`
- `roles/compute.storageAdmin`

```
$ gcloud projects add-iam-policy-binding (gcloud config get-value project) \
    --member=serviceAccount:m4a-source@(gcloud config get-value project).iam.gserviceaccount.com \
    --role=roles/compute.viewer
```
```
$ gcloud projects add-iam-policy-binding (gcloud config get-value project) \
    --member=serviceAccount:m4a-source@(gcloud config get-value project).iam.gserviceaccount.com \
    --role=roles/compute.storageAdmin
```

Retrieve Key file of Service Account
- `m4a-source-sa.json`: Service Account Key File

```
$ gcloud iam service-accounts keys create m4a-source-sa.json \
    --iam-account=m4a-source@(gcloud config get-value project).iam.gserviceaccount.com \
    --project (gcloud config get-value project)
```

### Create Processing Cluster
Creating Cluster
```
$ gcloud container clusters create m4a-process \
    --project (gcloud config get-value project) \
    --zone=us-central1-c \
    --num-nodes=1 \
    --machine-type=n1-standard-2
```

Retrieve Credential of Processing Cluster
```
$ gcloud container clusters get-credentials m4a-process \
    --zone=us-central1-c \
    --project (gcloud config get-value project)
```

### Install Migrate for Anthos
Install M4A module to Processing Cluster
```
$ migctl setup install --json-key m4a-process-sa.json
```

Verify the installation
```
$ migctl doctor
[???] Deployment
[???] Docker Registry
[???] Artifacts Repository
[???] Source Status
```

List the repository for Artifacts by M4A
```
$ migctl artifacts-repo list

NAME                                             TYPE    BUCKET
gcs-<PROJECT_ID>-migration-artifacts (default)   gcs     <PROJECT_ID>-migration-artifacts
```

### Add Migration Source
Create Migration Source
- Migration Source: `websphere-source`

```
$ migctl source create ce websphere-source --project (gcloud config get-value project) --json-key m4a-source-sa.json
```

Verify Migration Source
```
$ kubectl get SourceProvider
NAME               STATE
websphere-source   READY

$ migctl source list

$ migctl source status ce-source
State: READY
```

### Create Migration Plan
Stop Micgration Source Instance on Compute Engine
```
$ gcloud compute instances list

$ gcloud compute instances stop <INSTANCE_NAME> --zone <ZONE>
```

Create Migration Plan
- Migration Plan: `was-migration`

```
$ migctl migration create was-migration \
    --source websphere-source \
    --vm-id my-instance \
    --intent Image \
    --os-type Linux \
    --app-type websphere-traditional

Migration was-migration was created. Run `migctl migration status was-migration` to see its status.
```

```
$ migctl migration status was-migration

NAME            CURRENT-OPERATION       PROGRESS        STEP            STATUS  AGE
was-migration   GenerateMigrationPlan   [4/4]           Completed       Running 2m20s
```

### Review WebSphere Migration Report
[Open Cloud Stroage Browser](https://console.cloud.google.com/storage/browser?_ga=2.194885034.647664900.1618187125-431614551.1617859607&_gac=1.190165209.1614817503.Cj0KCQiAhP2BBhDdARIsAJEzXlG2TlvmzfehByHzkbrtwrm3QBiyvkBudRTe5AxxdvHgCv3mGgC9bo8aAnZ1EALw_wcB)

Find `<APP_NAME>.ear_MigrationReport.html` on the following directory:

- <PROJECT_ID>-migration-artifacts/
  - v2k-system-was-migration/
    - <HASH_ID>/
      - discovery/

![report-dir](https://user-images.githubusercontent.com/3072734/114657960-0ac90480-9d2c-11eb-9860-3057827b3cdb.png)

Review the Report HTML
![was-report](https://user-images.githubusercontent.com/3072734/114659032-cb9bb300-9d2d-11eb-9758-e11cc1781f8f.png)

Retrieve Migration Plan
- Migration Plan: `was-migration.yaml`

```
$ migctl migration get was-migration
```

### Customize the Migration Plan
You can find multiple apps on the plan:

```yaml
apiVersion: anthos-migrate.cloud.google.com/v1beta2
kind: WebSphereGenerateArtifactsFlow
  :
  :
spec:
  appBinariesMigrationToolkit:
    applications:
    - httpEndpoints:
      :
      :
      my-instanceNode01Cell/applications/JaxWSServicesSamples.ear
    - httpEndpoints:
      :
      :
      my-instanceNode01Cell/applications/DefaultApplication.ear
      :
    - httpEndpoints:
      :
      :
      my-instanceNode01Cell/applications/query.ear
    - httpEndpoints:
      :
      :
      my-instanceNode01Cell/applications/ivtApp.ear
    detectSharedLibraries: true
    gcsPath: binaryAppScanner.jar
    includePackages: ch.qos,com.fasterxml,com.ibm,com.informix,com.lowagie,com.mchange,com.meterware,com.microsoft,com.sun,com.sybase,freemarker,groovy,java,javax,net,oracle,org,sqlj,sun,twitter4j,_ibmjsp
    includeSensitiveData: true
    sourceAppServer: was90
  image:
    name: my-instance
  deployment:
    appName: my-instance
    folder: v2k-system-was-migration/94b13602-8f1f-416f-86c4-d69f6a3c396a/
  configs: {}
```

- You should delete all but one path definition
- You should name for app uniquely
```yaml
  image:
    name: jax-app-was:v0.0.1
  deployment:
    appName: jax-app
```

Update Migration Plan
```
$ migctl migration update was-migration --file was-migration.yaml
```

### Generate Artifacts
Generate Artifact with rewied migration plan
```
$ migctl migration generate-artifacts was-migration

Generate Artifacts task started for Migration was-migration. Run `migctl migration status was-migration` to see its status.
```

Verify status
```
$ migctl migration status was-migration

NAME            CURRENT-OPERATION       PROGRESS        STEP            STATUS          AGE
was-migration   GenerateArtifacts       [1/1]           ExtractImage    Completed       2h13m51s
```

You can find the following files when the process finishes:

![artifact](https://user-images.githubusercontent.com/3072734/114661793-8b8aff00-9d32-11eb-8576-be0dd57c009f.png)

Download the generated artifact files
```
$ migctl migration get-artifacts was-migration
```

|File|Description|
|----|-----------|
|Dockerfile|It is used to build the image for the migrated app|
|build.sh|It is used to build deployment using `gcloud builds`|
|deployment_spec.yaml|It is used by `build.sh` or `kubectl apply`|


<details><summary>Dockerfile</summary><div>

```dockerfile
FROM ibmcom/websphere-traditional:9.0.5.6

ADD --chown=was:root additionalFiles.tar.gz /

COPY --chown=was:root JaxWSServicesSamples.ear_wsadmin.py /work/config/

COPY --chown=was:root JaxWSServicesSamples.ear /work/app/

RUN /work/configure.sh
```

</div></details>

<details><summary>JaxWSServicesSamples.ear_wsadmin.py</summary><div>

```python
Cell=AdminConfig.getid('/Cell:' + AdminControl.getCell() + '/')
Node=AdminConfig.getid('/Cell:' + AdminControl.getCell() + '/Node:' + AdminControl.getNode() + '/')
Server=AdminConfig.getid('/Cell:' + AdminControl.getCell() + '/Node:' + AdminControl.getNode() + '/Server:server1')
NodeName=AdminControl.getNode()

print 'Starting Creating JVM Properties'

print 'Starting Creating Authentication Alias'

print 'Starting Creating Queues'

print 'Starting Creating Topics'

print 'Starting Creating Activation Specifications'

print 'Starting Creating Connection Factories'

print 'Starting Creating JDBC Providers'

print 'Starting Creating Variables'

print 'Starting Saving Configuration Changes Before Application Deployment'
AdminConfig.save()
print 'Starting Application Deployment'
AdminApp.install('/work/app/JaxWSServicesSamples.ear', ["-node", NodeName, "-server", "server1", "-appname", "JaxWSServicesSamples.ear", "-CtxRootForWebMod", [["SampleClientSei", "SampleClientSei.war,WEB-INF/web.xml", "/wssamplesei"], ["SampleServicesSei", "SampleServicesSei.war,WEB-INF/web.xml", "/WSSampleSei"], ["SampleMTOMClient", "SampleMTOMClient.war,WEB-INF/web.xml", "/wssamplemtom"], ["SampleMTOMService", "SampleMTOMService.war,WEB-INF/web.xml", "/WSSampleMTOM"]]])
AdminConfig.save()
```

</div></details>

<details><summary>build.sh</summary><div>

```bash
#!/bin/bash
gsutil -m cp -n gs://PROJECT_ID-migration-artifacts/v2k-system-was-migration/3c70d462-2b91-4288-a370-db7af82da85a/JaxWSServicesSamples.ear/* ./
gcloud builds submit --timeout 1h -t gcr.io/PROJECT_ID/jax-app-was:v0-0-1-jaxwsservicessamples-ear
```

</div></details>

<details><summary>deployment_spec.yaml</summary><div>

```yaml
# Stateless application specification
# The Deployment creates a single replicated Pod, indicated by the 'replicas' field
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: jax-app-jaxwsservicessamples-ear
    migrate-for-anthos-optimization: "true"
    migrate-for-anthos-version: v1.7.0
  name: jax-app-jaxwsservicessamples-ear
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jax-app-jaxwsservicessamples-ear
      migrate-for-anthos-optimization: "true"
      migrate-for-anthos-version: v1.7.0
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: jax-app-jaxwsservicessamples-ear
        migrate-for-anthos-optimization: "true"
        migrate-for-anthos-version: v1.7.0
    spec:
      containers:
      - image: gcr.io/shinyay-works-201123-296509/jax-app-was:v0-0-1-jaxwsservicessamples-ear
        name: jax-app-jaxwsservicessamples-ear
        resources: {}
status: {}

---
# Headless Service specification -
# No load-balancing, and a single cluster internal IP, only reachable from within the cluster
# The Kubernetes endpoints controller will modify the DNS configuration to return records (addresses) that point to the Pods, which are labeled with "app": "jax-app-jaxwsservicessamples-ear"
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  name: jax-app-jaxwsservicessamples-ear
spec:
  clusterIP: None
  ports:
  - name: defaulthttpendpoint-http
    port: 9080
    protocol: TCP
    targetPort: 9080
  - name: defaulthttpendpoint-https
    port: 9443
    protocol: TCP
    targetPort: 9443
  selector:
    app: jax-app-jaxwsservicessamples-ear
  type: ClusterIP
status:
  loadBalancer: {}

---
```

</div></details>

### Build Container Image
Change service type `LoadBalancer` from `ClusterIP` on `deployment_spec.yaml`

```yaml
apiVersion: v1 
kind: Service 
:
:
spec: 
  :
  : 
  type: LoadBalancer # <- from ClusterIP
```

```
$ chmod +x ./build.sh
$ ./build.sh
```


- Layered Image
![dive-in-container](https://user-images.githubusercontent.com/3072734/114809056-6951b980-9de4-11eb-9d45-c3be5e3daedb.png)


- Running Image
![generated-image](images/m4a-container.gif)

### Deploy App Container
```
# kubectl apply -f deployment_spec.yaml
```

## Installation

## References

## Licence

Released under the [MIT license](https://gist.githubusercontent.com/shinyay/56e54ee4c0e22db8211e05e70a63247e/raw/34c6fdd50d54aa8e23560c296424aeb61599aa71/LICENSE)

## Author

[shinyay](https://github.com/shinyay)
