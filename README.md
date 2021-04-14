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
was-migration   GenerateMigrationPlan   [4/4]           Discovery       Running 1m20s
```

## Installation

## References

## Licence

Released under the [MIT license](https://gist.githubusercontent.com/shinyay/56e54ee4c0e22db8211e05e70a63247e/raw/34c6fdd50d54aa8e23560c296424aeb61599aa71/LICENSE)

## Author

[shinyay](https://github.com/shinyay)
