# Provisioning Datera Storage in Kubernetes
Kubernetes provides two different ways to provisioning storage:

  * **Manual Provisioning**: the cluster administrator has to manually make calls to the storage infrastructure to create persistent volumes and then users need to create volume claims to consume storage volumes.

  * **Dynamic Provisioning**: storage volumes are automatically created on-demand when users claim for storage avoiding the cluster administrator to pre-provision storage.

Datera DSP supports both the ways of storage provisioning in Kubernetes.

## Storage Secrets
If you are going to provision storage, first, define a secret to access the Datera volume as in the following ``datera-storage-secrets.yaml`` file:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: datera
  namespace: datera
type: datera/iscsi
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
  server: dGx4MTgwLnRseC5kYXRlcmFpbmMuY29tOjc3MTc=
```

The above file is created by base64 encoding of "admin", "password" and “server:port”, as for example

```bash
$ echo -n "admin" | base64
YWRtaW4=

$ echo -n "password" | base64
cGFzc3dvcmQ=

$ echo -n "SE-Cluster.tlx.daterainc.com:7717" | base64
dGx4MTgwLnRseC5kYXRlcmFpbmMuY29tOjc3MTc=
```

Then create the secret in the datera namespace

    $ kubectl create -f datera-storage-secrets.yaml

This secret can now be used to supply the access credentials for the Datera system. 

Kubernetes secrets are namespace based so possible for each namespace to use their own username and password.  When users are mapped to tenants in the Datera system, any provisioning request to the Datera system will occur under the designated tenant. Refer to the Datera system user guide for user to tenant mappings.

## Manual Datera storage provisioning
If you’re using manual storage provisioning, create a Persistent Volume definition as in the following ``datera-volume.yaml`` file and refer to secret just created above

```yaml
kind: Persistent Volume
apiVersion: v1
metadata:
  name: datera-volume
  labels:
    manual: 'true'
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  flexVolume:
    driver: datera/iscsi
    fsType: xfs
    secretRef:
      name: datera
    options:
      template: basic
      retainPolicy: never
      volumeID: datera-kube-volume
```

Datera allows the Kubernetes cluster administrator to configure a wide range of additional parameters in the Persistent Volume definition. See section below for a detailed list of the parameters.

Now define a Persistent Volume Claim as in the ``datera-volume-claim.yaml`` file

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: datera-volume-claim
  namespace: 
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      manual: 'true'
```

A volume claim can define a label selector to bound a specific volume. In our case, we are using the label selector to make sure claim is bound to the persistent volume we defined above.

Create the Persistent Volume.

    $ kubectl create -f datera-volume.yaml

Create to bind the claim to the volume above.

    $ kubectl create -f datera-volume-claim.yaml

Check if the Persistent Volume Claim bound to the Persistent Volume as expected

    $ kubectl get pv
    NAME          CAPACITY  ACCESS MODES RECLAIM POLICY  STATUS   CLAIM      STORAGECLASS  REASON AGE
    datera-volume 10Gi      RWO          Delete          Bound    default/datera-volume-claim     7m

Now, use that claim in a pod definition. Please, note that the actual volume on the Datera backend storage is not created until a pod tries to mount the claim. Once the pod is scheduled by Kubernetes on a given host, then the volume is created on the Datera backend and then mounted on the host.

## Volume Access Mode
A persistent volume can be mounted on a host in any way supported by the resource provider. Different storage providers have different capabilities and access modes are set to the specific modes supported by the storage technology. Datera supports “ReadWriteOnce” access mode, meaning the volume can be mounted as read-write by a single worker node at time.

## Volume Reclaim Policy
Datera supports “Retain” and “Delete” as volume reclaim policy. When set to “Retain”, and the Persistent Volume Claim is deleted, the volume is released but it is not yet available for another claim because the previous claimant’s data are still on the volume. When set to “Delete”, the Persistent Volume, and the volume on Datera side, are dynamically deleted when the user deletes the attached Persistent Volume Claim.

## Storage Class
Dynamic storage provisioning in Kubernetes is based on the Storage Classes. A persistent volume uses a given storage class specified into its definition file. A claim can request a particular class by specifying the name of a storage class in its definition file. Only volumes of the requested class can be bound to the claim requesting that class.

Multiple storage classes can be defined specifying the volume provisioner to use when creating a volume of that class. This allows the cluster administrator to define multiple type of storage within a cluster, each with a custom set of parameters.

Define a Storage Class for the Datera storage backend as in the following ``datera-storage-class.yaml`` file example below

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  labels: 
  name: datera
  namespace: 
provisioner: datera.io/provisioner-datera
reclaimPolicy: Delete
parameters:
  template: basic
  secretName: datera
  secretNamespace: datera
  fsType: xfs
```

Create the Storage Class

    $ kubectl create -f datera-storage-class.yml

Datera allows the Kubernetes cluster administrator to configure a wide range of parameters in the Storage Class definition.

## Dynamic Datera storage provisioning
If you’re using dynamic storage provisioning, install first the Datera agent (please, refer to the Manual provisioning section above) and then, install the Datera external storage provisioner.

By default, these components will be installed in to the ``datera`` namespace.

    $ kubectl create -f http://datera.io/assets/files/datera-k8s-external-provisioner.yaml

Wait for the provisioner image is pulled and check the pod is running

    $ kubectl get pods -l tier=datera-dynamic-provisioner –-namespace=datera

    NAMESPACE     NAME                                       READY STATUS  RESTARTS AGE
    datera        datera-provisioner-agent-7bc697b8bd-mq9tj  1/1   Running 0        3m

Make sure a Datera storage class is available and define a Persistent Volume Claim referring to that storage class as in the ``datera-volume-claim-sc.yaml`` file

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: datera-volume-claim
  namespace: 
spec:
  storageClassName: datera
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Create the claim to bind to a dynamic volume.

    $ kubectl create -f datera-volume-claim-sc.yaml

Check if a dynamically created Persistent Volume has been created and bound to the Persistent Volume Claim.

    $ kubectl get pv
    
    NAME         CAPACITY ACCESS MODES  RECLAIM POLICY   STATUS  CLAIM                           STORAGECLASS   REASON AGE  
    pvc-9fa9ea57 10Gi     RWO           Delete           Bound   default/datera-data-vol-claim   datera                2m              
Now, you can use the claim in a pod definition. Please, note that the actual volume on the Datera backend storage is not created until a pod tries to mount the claim. Once the pod is scheduled by Kubernetes on a given host, then the volume is created on the Datera backend and then mounted on the host.

## Supported Driver Parameters
The following is a list of storage parameters that an administrator can use for defining the Storage provisioning in Kubernetes with Datera:

|Parameter|Provisioner|Mandatory|Description|Notes|
|---------|-----------|---------|-----------|-----|
|driver|Dynamic/Manual|Yes|Type of driver used in Flex Volume template. Must be set to "datera/iscsi"||
|fsType|Dynamic/Manual|Yes|Type of the file system to be used when the volume is made available to the container.  |Specify the Filesystem name per the linux command line such as “ext4”. Default is ”xfs”.|
|template|Dynamic/Manual|No|Specifies the Datera Application template (AppTemplate) to use for provisioning storage.|The AppTemplate settings override the following other parameters: size, replica count, maxIops, maxBW, ip Pool, placementMode.|
|size|Dynamic/Manual|Yes|The size of the volume in Gigabytes|Ignored, if AppTemplates are used|
|replica|Dynamic/Manual|No|The number of copies of data stored on the Datera, see the Datera user guide for details.|Default is 3. Ignored, if AppTemplates are used|
|maxIops|Dynamic/Manual|No|QoS for the volume.  The max number of IOPs available to the PV, see the Datera user guide for details.|Default is unlimited. Ignored, if AppTemplates are used|
|maxBW|Dynamic/Manual|No|QoS for the volume.  Maximum bandwidth to allow for the volume, in KiB/s|Default is unlimited. Ignored, if AppTemplates are used|
|ipPool|Dynamic/Manual|No|The Ip Pool to use for the provisioning of volumes.  Caution should be used to ensure the Kubernetes worker has access to the network pool|Default is “default”. Ignored, if AppTemplates are used|
|placementMode|Dynamic/Manual|No|Defines how the data is placed in the Datera between the different media types.  Consult your local Datera administrator as to what options are available.  Specifying a media type that is not present will not fail, the system will make the “best effort” to meet the requirements|Default is “hybrid”. Ignored, if AppTemplates are used|
|tenant|Dynamic/Manual|No|Path to valid tenant on the Datera system. If the user has system administration rights, specific tenants can be specified here. The tenant must exist before provisioning|Default is “/root”. Valid Path: “/root/<tenant name>”|
|volumeID|Manual|Yes|ID for the datera volume that manual PV should get attached to. If Datera volume with this ID doesn’t exist, driver would create one with this ID and map it to the PV.||
|retainPolicy|Manual|No|Controls the retain behavior regarding the volume on the Datera. Options are: “never”, “retain”. |When manually provisioning storage, don’t mix “persistentVolumeReclaimPolicy” and “retainPolicy”, it will create a conflict.|
|secretName|Dynamic|Yes|Name of the secret to communicate with Datera system.|User needs to enter information about username, password and Datera server in base64 encoding to communicate to Datera system.|
|secretRef|Manual|Yes|Name of the secret to communicate with Datera system.|User needs to enter information about username, password and Datera server in base64 encoding to communicate to Datera system.|
|secretNamespace|Dynamic/Manual|Yes|Namespace for the secret used to communicate with Datera system.|Any namespace accessible by the user|
|volumeSource|Dynamic/Manual|No|Volume source allows for each PV request to be a clone operation from another volume or snapshot in the system|Path to Datera volume snapshot.|

## Volumes lifecycle
The reclaim policy of a Persistent Volume tells the cluster what to do with the volume after it has been released of its claim. Currently, the Datera volume plugin supports the Retain and the Delete reclaim policy. Deletion removes the Persistent Volume objects from Kubernetes, as well as deleting the associated storage asset in the Datera backend storage device (external infrastructure).

Retain reclaim policy allows for manual reclamation of the resource. When the Persistent Volume Claim is deleted, the Persistent Volume continues to exist in the Datera backend storage device (and the data is preserved), and the volume is considered “released”, but however, the volume is not available for another claim.

### Manual volume lifecycle with retain policy
When manually create a volume with policy set to “retain”, the following happens:

 1. Administrator creates the Persistent Volume (PV). The PV moves to the available state.
 2. User creates the Persistent Volume Claim (PVC): binding happens between the Persistent Volume Claim and the Persistent Volume. The PV moves to the bound state.
 3. User creates the PodA and the Kubernetes will schedule it on a given worker node. The Datera volume is created based on the parameters specified into the Datera template. The worker node where the PodA landed mounts the Datera volume.
 4. User deletes the PodA: the worker node unmounts the Datera volume.
 5. Users create a new PodB and the Kubernetes will schedule it on a different or the same worker node. The worker node where the PodB landed mounts the Datera volume.
 6. User deletes the PodB: the worker node unmounts the Datera volume.
 7. User deletes PVC. The PVC is gone but the PV is still there in released state. However, the PV is not available for binding with other PVCs.
 8. Administrator deletes the PV.
 9. Administrator deletes the Datera volume.  

![](./img/manual-provisioned-volume-with-retain-policy.png?raw=true)

### Manual volume lifecycle with delete policy
When manually create a volume with policy set to “delete”, the following happens:

 1. Administrator creates the Persistent Volume (PV). The PV moves to the available state.
 2. User creates the Persistent Volume Claim (PVC): binding happens between the Persistent Volume Claim and the Persistent Volume. The PV moves to the bound state.
 3. User creates the PodA and the Kubernetes will schedule it on a given worker node. The Datera volume is created based on the parameters specified into the Datera template. The worker node where the PodA landed mounts the Datera volume.
 4. User deletes the PodA: the worker node unmounts the Datera volume.
 5. Users create a new PodB and the Kubernetes will schedule it on a different or the same worker node. The worker node where the PodB landed mounts the Datera volume.
 6. User deletes the PodB: the worker node unmounts the Datera volume.
 7. User deletes PVC. The PV is auto deleted because of the policy. The Datera volume is deleted too.

![](./img/manual-provisioned-volume-with-delete-policy.png?raw=true)

### Dynamic volume lifecycle with retain policy
When dynamically creating a volume with policy set to “retain”, the following happens:

 1. User creates Persistent Volume Claim. The Datera dynamic provisioner creates the PVC and the Persistent Volume on Kubernetes and the physical volume is provisioned on the Datera backend. Binding between PV and PVC happens. The PV moves to the bound state.
 2. User creates the PodA and the Kubernetes will schedule it on a given worker node. The Datera volume is created based on the parameters specified into the Datera template. The worker node where the PodA landed mounts the Datera volume.
 3. User deletes the PodA: the worker node unmounts the Datera volume.
 4. Users create a new PodB and the Kubernetes will schedule it on a different or the same worker node. The worker node where the PodB landed mounts the Datera volume.
 5. User deletes the PodB: the worker node unmounts the Datera volume.
 6. User deletes PVC. The PV moves to released state because of the policy.
 7. Admin deletes the PV on Kubernetes and the physical volume on the Datera side.

![](./img/dynamic-provisioned-volume-with-retain-policy.png?raw=true)

### Dynamic volume lifecycle with delete policy
When dynamically creating a volume with policy set to “delete”, the following happens:

 1. User creates Persistent Volume Claim. The Datera dynamic provisioner creates the PVC and the Persistent Volume on Kubernetes and the physical volume is provisioned on the Datera backend. Binding between PV and PVC happens. The PV moves to the bound state.
 2. User creates the PodA and the Kubernetes will schedule it on a given worker node. The Datera volume is created based on the parameters specified into the Datera template. The worker node where the PodA landed mounts the Datera volume.
 3. User deletes the PodA: the worker node unmounts the Datera volume.
 4. Users create a new PodB and the Kubernetes will schedule it on a different or the same worker node. The worker node where the PodB landed mounts the Datera volume.
 5. User deletes the PodB: the worker node unmounts the Datera volume.
 6. User deletes PVC. The Datera dynamic provisioner deletes the Persistent Volume on Kubernetes because of the policy. The physical volume is deleted from the Datera storage backend.

![](./img/dynamic-provisioned-volume-with-delete-policy.png?raw=true)
