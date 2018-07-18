# Deploy a HA MySQL on Red Hat OpenShift with Datera
OpenShift Container Platform is Red Hat’s on-premises and cloud container management platform. It is build on top of Kubernetes and adds more features to simplify the software applications delivery process. Many customers have been using OpenShift to run stateless applications, but running stateful applications like databases may be are a challenge on OpenShift. 

Red Hat offers a portfolio of enterprise-class storage solutions, but neither GlusterFS (the basis for what Red Hat calls “Container Native Storage”) or Ceph were designed to run high-performance low-latency databases. GlusterFS and Ceph are great projects, but they both have specific drawbacks for database workloads.

The key to running a database on OpenShift is to leverage a cloud-native storage solution designed from the ground up for high-performance databases or other stateful services. Datera is the solution for running stateful containers in production designed with DevOps in mind. With Datera, users can manage any database or stateful service on any infrastructure using any container scheduler, including OpenShift, Kubernetes, DC/OS, and Docker Swarm.

Red Hat OpenShift version 3.9 with support for external volume plugins allowing you to take advantage of Datera enterprise-class storage features to encrypt, snapshot, backup, and ensure high availability to protect your mission-critical applications.

In this section, we're going to run an HA MySQL database on OpenShift 3.9 in few steps:

  1. Install the Datera volume plugin for OpenShift so you can use replicas, snapshots, backups, availability, encryption from the Datera platform.
  2. Install the Datera dynamic provisioner and create a Storage Class.
  3. Create a MySQL template in OpenShift and configure your OpenShift MySQL persistent volume with a memory limit, MySQL parameters, and size.
  4. Deploy a MySQL application from the template and create a pod using this volume.
  5. Demonstrate MySQL HA by cordoning off the node, deleting the pod, and seeing that MySQL has been automatically rescheduled on another node with persistance of data.

We're assuming you already have a running OpenShift setup in place before to attempt to use it with Datera. Read on [here](https://docs.openshift.com/) for more detailed information about how to setup OpenShift.

## Installing the Datera volume plugin
Datera volume plugin gets deployed as a Kubernetes DaemonSet with pods running in priviledged mode. As ``system:admin`` user, add the service account in the ``datera`` namespace to the privileged security context

```bash
oc adm policy \
   add-scc-to-user privileged \
   system:serviceaccount:datera:default
```

and deploy the plugin as stated in the [Deploying the Datera Driver for Kubernetes](./deploying.md) section of this guide.

*Note: you can use any dedicated service account instead of the default one and give it the permissions.*

## Install the Datera dynamic provisioner
Datera supports both the manual and dynamic storage provisioning model of Kubernetes. In this section, for a more smooth user's experience, we're going to use the dynamic model. Install the Datera dynamic storage provisioner and create a storage class as stated in the [Provisioning Datera Storage in Kubernetes](./provisioning.md) section of this guide.

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

In the Storage Class of this example we are referring to a template with the following parameters:

**Replication** – replica:  “3”
We are able to specify how many copies of this volume we want in the cluster. Datera supports a replication factor of 1|2|3|4|5.  Setting a replication factor of 3 ensures that Datera is synchronous in replicating the volume to 3 nodes in the Datera cluster, ensuring the data is persistent against node failures.

**Data Placement** – placement:  “hybrid”
Datera defines how the data is placed between the different media types. Specifying a media type that is not present will not fail, the system will make the “best effort” to meet the requirements so you can have servers with SSD, HDD, and SATA for example. Specifying a media type that is not present will not fail, the system will make the “best effort” to meet the requirements.

**Snapshots** – snap_interval:  “60”
Datera will create snapshot every 60 minutes with retention of 36. These snapshots can be used to roll back your database, test upgrades, and for dev testing.

*Consult your local Datera administrator as to what options are available.* 

## Create a MySQL template in OpenShift
We created a sample MySQL OpenShift template located [here](./examples/mysql-openshift-template.yaml).

Login to the Openshift platform as ``system:admin`` user, and deploy the file above

    oc create -f mysql-openshift-template.yaml

The template will be deployed into the ``openshift`` namespace and will be available as system template for all projects. You can select the memory limits and all the MySQL parameters, e.g. database name, user and password, or have them auto-generated for you. You can also set the size of the volume and the Storage Class you want to use.

## Deploy a MySQL application from the template
From the OpenShift web console, login as an unpriviledged user, e.g. user ``demo``, select the current project, e.g. ``demo`` and click on the Catalog menu item. Find the template as with the ``datera`` or ``mysql`` keywords.

Figure below show the template as it should appear on the GUI.

![](./img/oscm.png?raw=true)

Create an application from this template by filling all the parameters or leave the defaults. Make sure the Storage Class you entered matches the Datera Storage Class you created earlier.

Alternatively, you can create the application from command line instead of using the GUI. Login to the platform as ``demo`` user and select the ``demo`` project:

    oc login http://<openshift_master>:8443 -u demo -p ******
    oc project demo

Create the application from the template in the current project

```
oc new-app --template=mysql-datera-persistent
--> Deploying template "openshift/mysql-datera-persistent" to project demo

     Datera MySQL (Persistent)
     ---------
     NOTE: Scaling to more than one replica is not supported. You must have persistent volumes available in your cluster to use this template.

     The following service(s) have been created in your project: mysql.
     
            Username: user305
            Password: TTg2JAb1233XRkf6
       Database Name: sampledb
      Connection URL: mysql://mysql:3306/

     * With parameters:
        * Memory Limit=512Mi
        * ImageStream=openshift
        * Database Service Name=mysql
        * MySQL Connection Username=user305 # generated
        * MySQL Connection Password=TTg2JAb1233XRkf6 # generated
        * MySQL root user Password=sSgh0hXquEr2SgOc # generated
        * MySQL Database Name=sampledb
        * Volume Capacity=10Gi
        * Datera Storage Class=datera
        * Version of MySQL Image=5.7
```

Parameters can be specified by passing the ``--param`` options as in the following example
```
oc new-app --template=mysql-datera-persistent \
   --param=MYSQL_USER=admin \
   --param=MYSQL_PASSWORD=******* \
   --param=MYSQL_DATABASE=employees \
   ...
```

The application will create a new MySQL deploy

    oc get dc
    NAME      REVISION   DESIRED   CURRENT   TRIGGERED BY
    mysql     1          1         0         config,image(mysql:5.6)

Verify the Persistent Volume claim has been created and bound to a Persistent Volume

    oc get pvc
    NAME      STATUS    VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    mysql     Bound     pvc-a06f490b-8a61   10Gi       RWO            datera         1h

Login to the Datera GUI and cross-check the volume has been created as expected.

The container in the pod should take a minute or two to come up

    oc get pods
    NAME            READY     STATUS    RESTARTS   AGE
    mysql-1-lrvx7   1/1       Running   0          2m

After the container is running, login to the container and verify the storage has been connected by checking the container

    oc exec -it mysql-1-lrvx7 bash
    
    bash-4.2$ df -Th
    Filesystem          Type     Size  Used Avail Use% Mounted on
    overlay             overlay   39G  7.0G   32G  19% /
    /dev/mapper/os-root xfs       39G  7.0G   32G  19% /etc/hosts
    shm                 tmpfs     64M     0   64M   0% /dev/shm
    /dev/sdb            xfs       10G  199M  9.8G   2% /var/lib/mysql/data
    ...

You can see that the ``/var/lib/mysql/data directory`` is mounted to the 10GB Datera volume seen as ``/dev/sdb`` device.

Login to the database, create a table and add some data

    bash-4.2$ mysql -D employee -u root
    mysql> CREATE TABLE IF NOT EXISTS employees (
                  id INT(11) NOT NULL AUTO_INCREMENT,
                  name VARCHAR(45) DEFAULT NULL,
                  surname VARCHAR(45) DEFAULT NULL,
                  department VARCHAR(200) DEFAULT NULL,
                  PRIMARY KEY (id)
                  ) ENGINE=InnoDB;

    mysql> INSERT INTO employees VALUES (100, "John", "Smith", "Sales");
    mysql> ...
    mysql> exit
    Bye
    bash-4.2$ exit


## Demonstrate MySQL HA operations
Login back to the Openshift platform as ``system:admin`` user and determine what node the pod is running on

    oc get pods -o wide -n demo

    NAME            READY     STATUS    RESTARTS   AGE       IP            NODE
    mysql-1-lrvx7   1/1       Running   0          2h        10.128.0.57   oscm

Cordon off the node that the pod is running on

    oc adm cordon oscm

and verify the scheduling is disabled on that node

    oc get nodes
    NAME      STATUS                     ROLES            AGE       VERSION
    oscm      Ready,SchedulingDisabled   compute,master   2h        v1.9.1+a0ce1bc657
    oscna     Ready                      compute          2h        v1.9.1+a0ce1bc657

Login back as ``demo`` user and delete the MySQL pod

    oc delete pod mysql-1-lrvx7

Verify the pod has been rescheduled on another node in the cluster

    oc get pods -o wide
    NAME            READY     STATUS              RESTARTS   AGE       IP        NODE
    mysql-1-rdbgl   0/1       ContainerCreating   0          17s       <none>    oscna
  
Wait till the pod is becoming running, and check your data are still there.

## Summary
In summary, we have just seen how to run an HA MySQL database on OpenShift in 5 steps:

  1. Install the Datera volume plugin for OpenShift.
  2. Install the Datera dynamic provisioner and create a Storage Class.
  3. Create a MySQL template in OpenShift and configure your OpenShift MySQL persistent volume.
  4. Deploy a MySQL application from the template and create a pod using that volume.
  5. Demonstrate MySQL HA and seeing that MySQL has been automatically rescheduled on another node with persistance of data.


  

