# Openshift Registry on Minio+Datera
Openshift can build container images from source code, deploy them, and manage their entire lifecycle. To enable this, Openshift provides an internal, integrated Docker registry that can be deployed in your environment to locally manage images.

By default, the Openshift registry uses a filesystem backend placed on ephemeral storage that is destroyed if the pod exits.

Another option is to configure the OpenShift Registry to use an Object Storage S3 backend provided, by example, by AWS or by any local AWS/S3 compatible solution, e.g. [Minio](https://minio.io). 

In this section, we're going to document how to setup the Openshift Registry using an object storage S3 backend provided by Minio server runnin on Openshift and using Datera volume as persistent storage.

1. [Build the Minio image](#build-the-minio-image)
2. [Run Minio service with Datera](#run-minio-service-with-datera)
3. [Patch the Registry](#patch-the-registry)
4. [Use the Registry](#use-the-registry)
5. [Test data persistence](#test-data-persistence)

## Build the Minio image
The default Minio image available on the Docker Hub cannot run on OpenShift due to the security constraints enforced by Openshift. To run Minio, we need to leave Openshift to build its own image and store it on the local registry.

## Run Minio service with Datera

## Patch the Registry

## Test data persistence
