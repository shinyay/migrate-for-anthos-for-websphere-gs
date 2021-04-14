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
[✓] Deployment
[✓] Docker Registry
[✓] Artifacts Repository
[✗] Source Status
```

List the repository for Artifacts by M4A
```
$ migctl artifacts-repo list

NAME                                             TYPE    BUCKET
gcs-<PROJECT_ID>-migration-artifacts (default)   gcs     <PROJECT_ID>-migration-artifacts
```

### Add Migration Source
Create Migration Source
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
```
$ migctl migration create was-migration \
    --source was-source \
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
- `was-migration.yaml`

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

### Generate Artifacts
```
$ migctl migration generate-artifacts was-migration

Generate Artifacts task started for Migration was-migration. Run `migctl migration status was-migration` to see its status.
```

## Installation

## References

## Licence

Released under the [MIT license](https://gist.githubusercontent.com/shinyay/56e54ee4c0e22db8211e05e70a63247e/raw/34c6fdd50d54aa8e23560c296424aeb61599aa71/LICENSE)

## Author

[shinyay](https://github.com/shinyay)
