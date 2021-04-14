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
$ gsutil mb gs://(gcloud config get-value project)-websphere
```

## Installation

## References

## Licence

Released under the [MIT license](https://gist.githubusercontent.com/shinyay/56e54ee4c0e22db8211e05e70a63247e/raw/34c6fdd50d54aa8e23560c296424aeb61599aa71/LICENSE)

## Author

[shinyay](https://github.com/shinyay)
