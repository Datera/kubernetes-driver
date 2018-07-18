# How to Run a HA MySQL Database on Red Hat OpenShift with Datera
OpenShift Container Platform is Red Hat’s on-premises and cloud container management platform. Many customers have been using OpenShift to run stateless applications, but running stateful applications like databases may be a challenge on OpenShift. 

Red Hat offers a portfolio of enterprise-class storage solutions, but neither GlusterFS (the basis for what Red Hat calls “Container Native Storage”) or Ceph were designed to run high-performance low-latency databases. GlusterFS and Ceph are great projects, but they both have specific drawbacks for database workloads.

The key to running a database on OpenShift is to leverage a cloud-native storage solution designed from the ground up for high-performance databases or other stateful services. Datera is the solution for running stateful containers in production designed with DevOps in mind. With Datera, users can manage any database or stateful service on any infrastructure using any container scheduler, including OpenShift, Kubernetes, Mesosphere DC/OS, and Docker Swarm.

Red Hat OpenShift version 3.9 with support for external volume plugins allowing you to take advantage of Datera enterprise-class storage features to encrypt, snapshot, backup, and ensure high availability to protect your mission-critical applications.

In this section, we're going to run an HA MySQL database on OpenShift 3.9 in few steps:

  1. Install the Datera volume plugin for OpenShift so you can use replicas, snapshots, backups, availability, encryption from the Datera platform.
  2. Install the Datera dynamic provisioner and create a Storage Class.
  3. Create a MySQL template in OpenShift and configure your OpenShift MySQL persistent volume with a memory limit, MySQL parameters, and size.
  4. Deploy a MySQL application from the template and create a pod using this volume.
  5. Demonstrate MySQL HA by cordoning off the node, deleting the pod, and seeing that MySQL has been automatically rescheduled on another node.

We're assuming you already have a running OpenShift setup in place before to attempt to use it with Datera. Read on [here](https://docs.openshift.com/) for more detailed information about how to setup OpenShift.

## Installing the Datera volume plugin

## Install the Datera dynamic provisioner and create a Storage Class

## Create a MySQL template in OpenShift

## Deploy a MySQL application from the template

## Demonstrate MySQL HA operations
