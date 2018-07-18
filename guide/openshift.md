# Run a HA MySQL database on Red Hat OpenShift with Datera
OpenShift Container Platform is Red Hat’s on-premises and cloud container management platform. It is build on top of Kubernetes and adds more features to simplify the software applications delivery process. Many customers have been using OpenShift to run stateless applications, but running stateful applications like databases may be are a challenge on OpenShift. 

Red Hat offers a portfolio of enterprise-class storage solutions, but neither GlusterFS (the basis for what Red Hat calls “Container Native Storage”) or Ceph were designed to run high-performance low-latency databases. GlusterFS and Ceph are great projects, but they both have specific drawbacks for database workloads.

The key to running a database on OpenShift is to leverage a cloud-native storage solution designed from the ground up for high-performance databases or other stateful services. Datera is the solution for running stateful containers in production designed with DevOps in mind. With Datera, users can manage any database or stateful service on any infrastructure using any container scheduler, including OpenShift, Kubernetes, DC/OS, and Docker Swarm.

Red Hat OpenShift version 3.9 with support for external volume plugins allowing you to take advantage of Datera enterprise-class storage features to encrypt, snapshot, backup, and ensure high availability to protect your mission-critical applications.

In this section, we're going to run an HA MySQL database on OpenShift 3.9 in few steps:

  1. Install the Datera volume plugin for OpenShift so you can use replicas, snapshots, backups, availability, encryption from the Datera platform.
  2. Install the Datera dynamic provisioner and create a Storage Class.
  3. Create a MySQL template in OpenShift and configure your OpenShift MySQL persistent volume with a memory limit, MySQL parameters, and size.
  4. Deploy a MySQL application from the template and create a pod using this volume.
  5. Demonstrate MySQL HA by cordoning off the node, deleting the pod, and seeing that MySQL has been automatically rescheduled on another node.

We're assuming you already have a running OpenShift setup in place before to attempt to use it with Datera. Read on [here](https://docs.openshift.com/) for more detailed information about how to setup OpenShift.

## Installing the Datera volume plugin
Datera volume plugin gets deployed as a Kubernetes DaemonSet with pods running in priviledged mode. Add the service account  in the ``datera`` namespace to the privileged security context

```bash
oc adm policy \
   add-scc-to-user privileged \
   system:serviceaccount:datera:default
```

and deploy the plugin as stated in the [Deploying the Datera Driver for Kubernetes](../deploying.md) section of this guide.

*Note: you can use any dedicated service account instead of the default one and give it the permissions.*

## Install the Datera dynamic provisioner and create a Storage Class
Datera supports both the manual and dynamic storage provisioning model of Kubernetes. In this section, for a more smooth user's experience, we're going to use the dynamic model. Install the Datera dynamic storage provisioner and create the Storage Class as stated in the [Provisioning Datera Storage in Kubernetes](../provisioning.md) section of this guide.

Please, note that OpenShift comes with security enabled and you need to give permissions to the related service account before to attempt to install the Datera dynamic provisioner.

```bash
oc adm policy \
   add-cluster-role-to-user \
   system:persistent-volume-provisioner \
   system:serviceaccount:datera:default
   
oc adm policy \
   add-cluster-role-to-user \
   system:controller:persistent-volume-binder \
   system:serviceaccount:datera:default
```
Also make sure the secret containing the credentials used to login the Datera platform is accessible by the user, e.g. it is in the current project (namespace) where the user is working.

## Create a MySQL template in OpenShift

## Deploy a MySQL application from the template

## Demonstrate MySQL HA operations

