# Kubernetes Persistent Volumes with Datera EDF
One of the key challenges in running containerized applications is dealing with data persistence. Unlike virtual machines that offer durable and persistent storage, containers come with ephemeral storage. Kubernetes is a containers orchestration platform that offers a complete set of tools to support the deployment of storage in stateful workloads. 

Having realized the benefits of containers, there is an ongoing effort from the industry to support the containerization of stateful applications that can be seamlessly run with stateless workload. **Datera Inc.** provides **Datera Elastic Data Fabric (EDF)**, a block storage solution based on iSCSI technology for on-premise Clouds with high performance, low latency, highly scalable and high resilient storage for the most demanding stateful applications.

In this tutorial, we are going to see the **Kubernetes** storage model in action with **Datera EDF** as storage backend for stateful applications.

## Prerequisites
We assume you have already a kubernetes cluster up and running as well as a Datera EDF cluster available and connected togheter. Details on how to setup a Kubernetes cluster or a Datera EDF will not be covered in this tutorial. Please refer to their official documentation to learn how to.

Used Releases in this tutorial:

  * Kubernetes 1.8
  * CentOS 7.4
  * Datera EDF 2.2

## Persistent Storage Model in Kubernetes
When it comes to data persistence, Kubernetes introduces the concepts of volumes, persistent volumes, persistent volume claims and the stateful set constructs.

The Kubernetes storage model is based on the following abstractions:

  * **PersistentVolume**: it models shared storage that has been provisioned by the cluster administrator. It is a resource in the cluster just like a node is a cluster resource. Persistent volumes are like standard volumes, but having a lifecycle independent of any individual pod. Also they hide to the users the details of the implementation of the storage, e.g. NFS, iSCSI, or other cloud storage systems.

  * **PersistentVolumeClaim**: it is a request for storage by a user. It requests specific access modes like read-write or read-only and the storage capacity.

Kubernetes provides two different ways to provisioning storage:

  * **Manual Provisioning**: the cluster administrator has to manually make calls to the storage infrastructure to create persisten volumes and then users need to create volume claims to consume storage volumes.
  * **Dynamic Provisioning**: storage volumes are automatically created on-demand when users claim for storage avoiding the cluster administrator to pre-provision storage. 

In the next sections we're going to introduce this model by using simple examples. Please, refer to official kubernetes documentation for more details.

  * [Local Persistent Volumes](#local-persistent-volumes)
  * [Volume Access Mode](#volume-access-mode)
  * [Volume State](#volume-state)
  * [Volume Reclaim Policy](#volume-reclaim-policy)
  * [Manual volumes provisioning](#manual-volumes-provisioning)
  * [Storage Classes](#storage-classes)
  * [Dynamic volumes provisioning](#dynamic-volumes-provisioning)
  * [Redis benchmark](#redis-benchmark)
  * [Deploy a Consul cluster as StefulSet](#deploy-a-consul-cluster-as-statefulset)


  
## Local Persistent Volumes
Start by defining a persistent volume ``local-persistent-volume-recycle.yaml`` configuration file

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: local
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data"
  persistentVolumeReclaimPolicy: Recycle
```

The configuration file specifies that the volume is at ``/data`` on the the cluster’s node. The volume type is ``hostPath`` meaning the volume is local to the host node. The configuration also specifies a size of 2GB and the access mode of ``ReadWriteOnce``, meanings the volume can be mounted as read write by a single pod at time. The reclaim policy is ``Recycle`` meaning the volume can be used many times.  It defines the Storage Class name manual for the persisten volume, which will be used to bind a claim to this volume.

Create the persistent volume

    kubectl create -f local-persistent-volume-recycle.yaml
    
and view information about it 

    kubectl get pv
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Available             manual                   33m

Now, we're going to use the volume above by creating a claiming for persistent storage. Create the following ``volume-claim.yaml`` configuration file
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: volume-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

Note the claim is for 500MB of space where the the volume is 2GB. The claim will bound any volume meeting the minimum requirements specified into the claim definition. 

Create the claim

    kubectl create -f volume-claim.yaml

Check the status of persistent volume to see if it is bound

    kubectl get pv
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Bound     project/volume-claim   manual                   37m

Check the status of the claim

    kubectl get pvc
    NAME           STATUS    VOLUME    CAPACITY   ACCESSMODES   STORAGECLASS   AGE
    volume-claim   Bound     local     2Gi        RWO           manual         1m

Create a ``nginx-pod-pvc.yaml`` configuration file for a nginx pod using the above claim for its html content directory
```yaml
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx
  namespace: default
  labels:
spec:

  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
      - mountPath: "/usr/share/nginx/html"
        name: html

  volumes:
    - name: html
      persistentVolumeClaim:
       claimName: volume-claim
```

Note that the pod configuration file specifies a persistent volume claim, but it does not specify a persistent volume. From the pod point of view, the claim is the volume. Please note that a claim must exist in the same namespace as the pod using the claim.

Create the nginx pod

    kubectl create -f nginx-pod-pvc.yaml

Accessing the nginx will return *403 Forbidden* since there are no html files to serve in the data volume

    kubectl get pod nginx -o yaml | grep IP
      hostIP: 10.10.10.86
      podIP: 172.30.5.2

    curl 172.30.5.2:80
    403 Forbidden

Let's login to the worker node and populate the data volume

    echo "Welcome to $(hostname)" > /data/index.html

Now try again to access the nginx application

     curl 172.30.5.2:80
     Welcome to kubew05

To test the persistence of the volume and related claim, delete the pod and recreate it

    kubectl delete pod nginx
    pod "nginx" deleted

    kubectl create -f nginx-pod-pvc.yaml
    pod "nginx" created

Locate the IP of the new nginx pod and try to access it

    kubectl get pod nginx -o yaml | grep podIP
      podIP: 172.30.5.2

    curl 172.30.5.2
    Welcome to kubew05

## Volume Access Mode
A persistent volume can be mounted on a host in any way supported by the resource provider. Different storage providers have different capabilities and access modes are set to the specific modes supported by that particular volume. For example, NFS can support multiple read write clients, but an iSCSI volume can be support only one.

The access modes are:

  * **ReadWriteOnce**: the volume can be mounted as read-write by a single node
  * **ReadOnlyMany**: the volume can be mounted read-only by many nodes
  * **ReadWriteMany**: the volume can be mounted as read-write by many nodes

Claims and volumes use the same conventions when requesting storage with specific access modes. Pods use claims as volumes. For volumes which support multiple access modes, the user specifies which mode desired when using their claim as a volume in a pod.

A volume can only be mounted using one access mode at a time, even if it supports many. For example, a NFS volume can be mounted as ReadWriteOnce by a single node or ReadOnlyMany by many nodes, but not at the same time.

## Volume state
When a pod claims for a volume, the cluster inspects the claim to find the volume meeting claim requirements and mounts that volume for the pod. Once a pod has a claim and that claim is bound, the bound volume belongs to the pod.

A volume will be in one of the following state:

  * **Available**: a volume that is not yet bound to a claim
  * **Bound**: the volume is bound to a claim
  * **Released**: the claim has been deleted, but the volume is not yet available
  * **Failed**: the volume has failed 

The volume is considered released when the claim is deleted, but it is not yet available for another claim. Once the volume becomes available again then it can bound to another other claim. 

In our example, delete the volume claim

    kubectl delete pvc volume-claim

See the status of the volume

    kubectl get pv persistent-volume
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Available             manual                   57m

## Volume Reclaim Policy
When deleting a claim, the volume becomes available to other claims only when the volume claim policy is set to ``Recycle``. Volume claim policies currently supported are:

  * **Retain**: the content of the volume still exists when the volume is unbound and the volume is released
  * **Recycle**: the content of the volume is deleted when the volume is unbound and the volume is available
  * **Delete**: the content and the volume are deleted when the volume is unbound. 
  
*Please note that, currently, only NFS and HostPath support recycling.* 

When the policy is set to ``Retain`` the volume is released but it is not yet available for another claim because the previous claimant’s data are still on the volume.

Define a persistent volume ``local-persistent-volume-retain.yaml`` configuration file

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: local-retain
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data"
  persistentVolumeReclaimPolicy: Retain
```

Create the persistent volume and the claim

    kubectl create -f local-persistent-volume-retain.yaml
    kubectl create -f volume-claim.yaml

Login to the pod using the claim and create some data on the volume

    kubectl exec -it nginx bash
    root@nginx:/# echo "Hello World" > /usr/share/nginx/html/index.html
    root@nginx:/# exit

Delete the claim

    kubectl delete pvc volume-claim

and check the status of the volume

    kubectl get pv
    NAME           CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS     CLAIM                  STORAGECLASS     AGE
    local-retain   2Gi        RWO           Retain          Released   project/volume-claim   manual           3m
    
We see the volume remain in the released status and not becomes available since the reclaim policy is set to ``Retain``. Now login to the worker node and check data are still there.

An administrator can manually reclaim the volume by deleteting the volume and creating a another one.
 
## Manual volumes provisioning 
The main limit of local storage for container volumes is that storage area is tied to the host where it resides. If kubernetes moves the pod from another host, the moved pod is no more to access the data since local storage is not shared between multiple hosts of the cluster. To achieve a more useful storage backend we need to leverage on the shared storage provided by Datera elastic block storage.

### Install the Datera driver
We are going to install the Datera storage drivers on all the worker nodes. From the master node, install the driver as daemon set defined in the following ``datera-installer-agent.yaml`` file

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: datera
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    app: datera-installer-agent
    tier: driver-monitoring
    version: v1
  name: datera-installer-agent
  namespace: datera
spec:
  selector:
    matchLabels:
      name: datera-installer
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: datera-installer
    spec:
      containers:
      - command:
        - bash
        - -c
        - /datera-driver/datera-daemon.sh
        env:
        - name: DATERA_DRIVER_PATH
          value: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/datera~iscsi
        - name: DATERA_INSTALLER_LOGFILE
          value: /log/datera-installer.log
        image: dateraio/kube-dev:latest
        imagePullPolicy: Always
        name: datera-installer
        resources:
          requests:
            cpu: 150m
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
        volumeMounts:
        - mountPath: /datera
          name: datera-installer
        - mountPath: /udev-rules
          name: udev-rules
        - mountPath: /scripts
          name: scripts
        - mountPath: /log
          name: log
      dnsPolicy: ClusterFirst
      hostIPC: true
      hostNetwork: true
      hostPID: true
      restartPolicy: Always
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - hostPath:
          path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/datera~iscsi
        name: datera-installer
      - hostPath:
          path: /etc/udev/rules.d
        name: udev-rules
      - hostPath:
          path: /sbin/
        name: scripts
      - hostPath:
          path: /var/log
        name: log
```

A daemon set is a controller type ensuring each node in the cluster runs a pod. As new node is added to the cluster, a new pod is added to the node. As the node is removed from the cluster, the pod running on it is removed and not scheduled on another node. 

Create the daemon set

    kubectl create –f  datera-installer-agent.yaml

This file also create a dedicated namespace called ``datera`` where the pods drivers will run.

Check the driver images are pulled and pods are running

    kubectl -n datera get ds
    NAME                     DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    datera-installer-agent   4         4         4         4            4           <none>          14d

    kubectl -n datera get pods -o wide
    NAME                           READY     STATUS    RESTARTS   AGE       IP             NODE
    datera-installer-agent-5wqxt   1/1       Running   0          14d       172.19.3.199   kubew05
    datera-installer-agent-6x8bm   1/1       Running   0          14d       172.19.3.196   kubew06
    datera-installer-agent-jd56p   1/1       Running   0          14d       172.19.3.188   kubew04
    datera-installer-agent-rkt79   1/1       Running   0          14d       172.19.3.193   kubew07

### Create a Persistent Volume and the Persistent Volume Claim
Wait all pods are running and define a persistent volume as in the ``datera-volume.yaml`` file

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: datera-volume
  labels:
    vendor: datera
spec:
  capacity:
    storage: 10Gi
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  flexVolume:
    driver: "datera/iscsi"
    fsType: "xfs"
    options:
      backstoreServer: "datera_server_name:7717"
      serverPasswd: "password"
      serverUser: "admin"
      template: "kube-workload"
      retainPolicy: "never"
      volumeID: "datera-kube-volume"
```

Now define a volume claim as in the ``datera-volume-claim.yaml`` file
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: datera-volume-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      vendor: "datera"
```

Create the volume and the claim

    kubectl create -f datera-volume.yaml
    kubectl create -f datera-volume-claim.yaml

Check the volume and claim are bound togheter

    kubectl get pv
    NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                            STORAGECLASS  
    datera-volume        10Gi        RWO            Retain           Released   default/datera-volume-claim      manual      

    kubectl get pvc
    NAME                  STATUS    VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS
    datera-volume-claim   Bound     datera-volume    10Gi        RWO            manual      

### Create an application using that volume
Now we are going to create an nginx application using the above claim. For example, create the ``nginx-pvc-template.yaml`` template for a nginx application having the html content folder placed on the Datera storage
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  generation: 1
  labels:
    run: nginx
  name: nginx-pvc
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
          name: "http-server"
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: html
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: datera-volume-claim
      dnsPolicy: ClusterFirst
      restartPolicy: Always
```

The template above defines a nginx application with a persistent volume for its html content folder.

Deploy the application 
  
    kubectl create -f nginx-pvc-template.yaml 

Check pod is running

    kubectl get pods -o wide
    NAME                         READY     STATUS    RESTARTS   AGE       IP            NODE
    nginx-pvc-3474572923-3cxnf   1/1       Running   0          2m        10.38.5.89    kubew05

Login to the pod and create some simple HTML content for the nginx application

     kubectl exec -it nginx-pvc-3474572923-3cxnf bash
     root@nginx-pvc-3474572923-3cxnf:/# echo 'Hello from Datera Elastic Data Fabric!' > /usr/share/nginx/html/index.html                
     root@nginx-pvc-3474572923-3cxnf:/# exit

Now access the pod by its IP address

    curl http://10.38.5.89
    Hello from Datera Elastic Data Fabric!

### Volume selector
Having multiple volumes, a claim can define a label selector to bound to a specific volume. For example, define a claim as in the ``pvc-volume-selector.yaml`` configuration file

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-volume-selector
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      volumeName: "volume00"
```

## Storage Classes
A Persistent Volume uses a given storage class specified into its definition file. A claim can request a particular class by specifying the name of a storage class in its definition file. Only volumes of the requested class can be bound to the claim requesting that class.

If the storage class is not specified in the persistent volume definition, the volume has no class and can only be bound to claims that do not require any class. 

Multiple storage classes can be defined specifying the volume provisioner to use when creating a volume of that class. This allows the cluster administrator to define multiple type of storage within a cluster, each with a custom set of parameters.

For example, the following ``datera-storage-class.yaml`` configuration file defines a storage class for Datera backend
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: datera-storage-class
provisioner: datera.io/provisioner-datera
reclaimPolicy: Retain
parameters:
  template: "kube-workload"
  replica: "3"
  size: 100G
  backstoreServer: "datera_server_name:7717"
  serverUser: "admin"
  serverPasswd: "password"
  fsType: "xfs"
```

Create the storage class

    kubectl create -f datera-storage-class.yaml
    
    kubectl get sc
    NAME                              PROVISIONER
    datera-storage-class              datera.io/provisioner-datera

The cluster administrator can define a class as default storage class by setting an annotation in the class definition file
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: datera-storage-class
   annotations:
     storageclass.kubernetes.io/is-default-class: "true"
provisioner: datera.io/provisioner-datera
reclaimPolicy: Delete
parameters:
  template: "kube-workload"
  replica: "3"
  size: 100G
  backstoreServer: "datera_server_name:7717"
  serverUser: "admin"
  serverPasswd: "password"
  fsType: "xfs"
```

Check the storage classes

    kubectl get sc
    NAME                              PROVISIONER
    datera-storage-class (default)   datera.io/provisioner-datera

If the cluster administrator defines a default storage class, all claims that do not require any class will be dynamically bound to volumes having the default storage class. 

## Dynamic volumes provisioning
In this section we are going to configure dynamic storage provisioning. Make sure the storage class has been created for the Datera provisioner

    kubectl get sc
    NAME                              PROVISIONER
    datera-storage-class (default)   datera.io/provisioner-datera

Also make sure the Datera driver is running as daemonset on all the worker nodes.

### Install the Datera storage provisioner
We are going to install the Datera external storage provisioner in the ``datera`` namespace. From the master node, install the Datera external provisioner as deployment defined in the ``datera-k8s-external-provisioner.yaml`` file

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: datera-provisioner-agent
  namespace: datera
  labels:
    tier: datera-dynamic-provisioner
    app: datera-provisioner-agent
    version: v1.8
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: datera-provisioner
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: datera-provisioner
        imagePullPolicy: Always
        image: dateraio/datera-provisioner:latest
        securityContext:
          privileged: true
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: DATERA_PROVISIONER_INSTALLER_LOGFILE
          value: "/log/datera-provisioner-installer.log"
        command: [ "bash", "-c", "/datera-provisioner/datera-provisioner-installer.sh" ]
        volumeMounts:
          - name: udev-rules
            mountPath: /udev-rules
          - name: scripts
            mountPath: /scripts
          - name: log
            mountPath: /log
      volumes:
        - name: udev-rules
          hostPath:
              path: /etc/udev/rules.d
        - name: scripts
          hostPath:
              path: /sbin/
        - name: log
          hostPath:
              path: /var/log
```

Create the daemon set in the ``datera`` namespace

    kubectl create –f  datera-k8s-external-provisioner.yaml

Check the driver images are pulled and pods are running

    kubectl -n datera get deploy
    NAME                       DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    datera-provisioner-agent   4         4         4         4            4           <none>          

    kubectl -n datera get pods -o wide
    NAME                           READY     STATUS    RESTARTS   AGE       IP             NODE
    datera-provisioner-agent-7bc697b8bd-mq9tj   1/1       Running       0          2h        172.19.3.199   kubew05

### Create an application using dynamic storage provisioning
Define a volume claim in the datera storage class as in the following ``sample-volume-claim.yaml`` file

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: sample-volume-claim
spec:
  storageClassName: datera-storage-class
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

and an Apache pod that is using that volume claim for its static html folder as in the following ``apache-with-datera-external-volume.yaml`` file 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apache-with-datera-external-volume
  labels:
    app: httpd
spec:
  containers:
  - name: apache
    image: centos/httpd:latest
    imagePullPolicy: IfNotPresent
    ports:
    - name: web
      containerPort: 80
      protocol: TCP
    volumeMounts:
    - mountPath: "/var/www/html"
      name: html
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: sample-volume-claim
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

Create the volume claim and the apache pod

	kubectl create -f sample-volume-claim.yaml
	kubectl create -f apache-with-datera-external-volume.yaml

Check the volume claim

	kubectl get pvc
	NAME                  STATUS    VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS           AGE
	sample-volume-claim   Bound     pvc-4af76e0f   10Gi       RWO            datera-storage-class   7m

The volume claim is bound to a dynamically created volume on the datera storage backend

	kubectl get pv
	NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                 STORAGECLASS
	pvc-4af76e0f   10Gi        RWO           Delete           Bound     apache-volume-claim   datera-storage-class

In the same way, the datera volume is dynamically removed when the claim is removed

	kubectl delete pvc sample-volume-claim
	kubectl delete pod apache-with-datera-external-volume
	kubectl get pod,pvc,pv

## Redis benchmark
In this section, we are going to use Datera block storage a backend for a Redis server. Redis is an open source, in-memory data structure store, used as a database, cache and message broker. Redis provides different levels of on-disk persistence. Redis is famous for its performances and, therefore, we are going to run a Redis benchmark having persistence on a Datera volume.

### Create a persistent volume claim
Make sure a Datera Storage Class is defined and create a claim for data as in the ``redis-data-claim.yaml`` configuration file
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: redis-data-claim
spec:
  storageClassName: datera-storage-class
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Create the claim and check the dynamic volume creation and the binding

	kubectl create -f redis-data-claim.yaml
	
	kubectl get pvc
	NAME                     STATUS    VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS                     AGE
	redis-data-claim         Bound     pvc-eaab62e3   10Gi       RWO            datera-storage-class             1m

### Create a Redis Master
Define a Redis Master deployment as in the ``redis-deployment.yaml`` configuration file
```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: redis
        role: master
      name: redis
    spec:
      containers:
        - name: redis
          image: kubernetes/redis:v1
          env:
            - name: MASTER
              value: "true"
          ports:
            - containerPort: 6379
          volumeMounts:
            - mountPath: /redis-master-data
              name: redis-data
      volumes:
        - name: redis-data
          persistentVolumeClaim:
            claimName: redis-data-claim
```

Define a Redis service as in the ``redis-service.yaml`` configuration file
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  ports:
  - port: 6379
    targetPort: 6379
    nodePort: 31079
    name: http
  type: NodePort
  selector:
    name: redis
```

Deploy the Redis Master and create the service

	kubectl create -f redis-deployment.yaml
	kubectl create -f redis-service.yaml

Wait the Redis pod is ready

	kubectl get pods -a -o wide
	NAME                                READY     STATUS      RESTARTS   AGE       IP             NODE
	redis-deployment-75466795f6-thtx4   1/1       Running     0          30s       10.38.5.62     kubew05

To verify Redis, install the ``netcat`` utility and connect to the Redis Master

	yum install -y nmap-ncat

	nc -v kubew05 31079
	Ncat: Version 6.40 
	Ncat: Connected to kubew05:31079.
	ping
	+PONG
	set greetings "Hello from Redis!"
	+OK
	get greetings
	$17
	Hello from Redis!
	logout

### Run benchmark
Define a job running the Redis performances benchmark as in the ``redis-benchmark.yaml`` configuration file
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: redis-bench
spec:
  template:
    metadata:
      name: bench
    spec:
      containers:
      - name: bench
        image: clue/redis-benchmark
      restartPolicy: Never
```

Create a batch job to run the benchmark

	kubectl create -f redis-benchmark.yaml

Wait the job completes

	kubectl get job
	NAME          DESIRED   SUCCESSFUL   AGE
	redis-bench   1         1            48s

Check the bench pod name and display the results

	kubectl get pods -a
	NAME                                        READY     STATUS      RESTARTS   AGE
	redis-bench-jgbj8                           0/1       Completed   0          1m
	redis-deployment-75466795f6-thtx4           1/1       Running     0          16m

	kubectl logs redis-bench-jgbj8


## Deploy a Consul cluster as StatefulSet
Common Kubernetes controllers as Replica Set and Daemon Set are a great way to run stateless applications, but their semantics are not so friendly for deploying all stateful applications. A better approach for deploying stateful applications on a kubernetes cluster, is to use the **Stateful Set**.

### Stateful Set
The purpose of Stateful Set is to provide a controller with better semantics for deploying stateful workloads. A Stateful Set manages the deployment and scaling of a set of pods, and provides guarantees about the ordering and uniqueness of these pods. Like a Replica Set, a Stateful Set manages pods that are based on an identical container specifications. Unlike Replica Set, a Stateful Set maintains a sticky identity for each of pod across any rescheduling.

For a Stateful Set with n replicas, when pods are deployed, they are created sequentially, in order from {0..n-1} with a sticky, unique identity in the form ``<statefulset name>-<ordinal index>``. The (i)th pod is not created until the (i-1)th is running. This ensure a predictable order of pod creation.

Deletion of pods in a stateful set follows the inverse order from {n-1..0}. However, if the order of pod creation is not strictly required, it is possible to create pods in parallel by setting the ``podManagementPolicy: Parallel`` option.

### Consul cluster
**HashiCorp Consul** is a distributed key-value database with a powerful service discovery. Consul is based on the **Raft** alghoritm for distributed consensus in a cluster of machines. In this section, we assume the reader knows hot to operate with Consul Cluster. Details about Consul and how to configure it can be found on the Consul project website. 

The most difficult part to run a Consul cluster on Kubernetes is how to form a cluster having Consul strict requirements about instance names of nodes being part of its cluster. In this section we are going to deploy a three node Consul cluster by using the stateful set controller.

A storage class needs to be created before to implement this example. We assume a persistent Datera EDF storage environment is available to the kubernetes cluster. This is because each Consul node uses a data directory where to store the status of the cluster and this directory needs to be preserved across pods deletion and restarts. 

### Bootstrap the cluster
A stateful set requires an headless service to control the domain of its pods. Here the headless service ``consul-svc.yaml`` configuration file

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: consul
  labels:
    app: consul
spec:
  ports:
  - name: rpc
    port: 8300
    targetPort: 8300
  - name: lan
    port: 8301
    targetPort: 8301
  - name: wan
    port: 8302
    targetPort: 8302
  - name: http
    port: 8500
    targetPort: 8500
  - name: dns
    targetPort: 8600
    port: 8600
  clusterIP: None
  selector:
    app: consul
```

The domain managed by this service takes the form ``$(service name).$(namespace).svc.cluster.local``.  Create the headless service

    kubectl create -f consul-svc.yaml

    kubectl get svc -o wide
    NAME    TYPE       CLUSTER-IP EXTERNAL-IP PORT(S)                                        AGE  SELECTOR
    consul  ClusterIP  None       <none>      8300/TCP,8301/TCP,8302/TCP,8500/TCP,8600/TCP   25s  app=consul

The presence of an headless service, ensure the correct node discovery for the forming Consul cluster. As each consul pod is getting created, it gets a matching service name, taking the form ``$(podname).$(service)``. This leads to a predictable service name surviving to pod deletion and restarts.

Now define a Stateful Set for the Consul cluster as in the ``consul-sts.yaml`` configuration file
```yaml
---
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  name: consul
  namespace:
  labels:
    type: statefulset
spec:
  serviceName: consul
  replicas: 3
  selector:
    matchLabels:
      app: consul
  template:
    metadata:
      labels:
        app: consul
    spec:
      containers:
      - name: consul
        image: consul:latest
        env:
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        ports:
        - name: rpc
          containerPort: 8300
        - name: lan
          containerPort: 8301
        - name: wan
          containerPort: 8302
        - name: http
          containerPort: 8500
        - name: dns
          containerPort: 8600
        volumeMounts:
        - name: consuldata
          mountPath: /consul/data
        args:
        - agent
        - -server
        - -datacenter=kubernetes
        - -data-dir=/consul/data
        - -log-level=trace
        - -client=0.0.0.0
        - -advertise=$(POD_IP)
        - -advertise-wan=127.0.0.1
        - -serf-wan-bind=127.0.0.1
        - -bootstrap-expect=3
        - -retry-join=consul-0.consul.$(POD_NAMESPACE).svc.cluster.local
        - -retry-join=consul-1.consul.$(POD_NAMESPACE).svc.cluster.local
        - -retry-join=consul-2.consul.$(POD_NAMESPACE).svc.cluster.local
        - -domain=cluster.local
        - -ui
  volumeClaimTemplates:
  - metadata:
      name: consuldata
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
      storageClassName: datera-storage-class      
```

Create a Stateful Set of three Consul nodes

    	kubectl create -f consul-sts.yaml

The Consul pods will be created in a strict order with a predictable name

   	kubectl get pods -o wide
	NAME          READY     STATUS    RESTARTS   AGE       IP             NODE
	consul-0      1/1       Running   0          35m       10.38.5.66     kubew05
	consul-1      1/1       Running   0          35m       10.38.6.14     kubew06
	consul-2      1/1       Running   0          35m       10.38.7.10     kubew07

    
Consul cluster should be formed

	kubectl exec -it consul-0 -- consul members
	Node      Address          Status  Type    Build  Protocol  DC          Segment
	consul-0  10.38.5.66:8301  alive   server  1.0.1  2         kubernetes  <all>
	consul-1  10.38.6.14:8301  alive   server  1.0.1  2         kubernetes  <all>
	consul-2  10.38.7.10:8301  alive   server  1.0.1  2         kubernetes  <all>


and ready to be used by any other pod in the kubernetes cluster.

Also each pod creates its own storage volume on the Datera backend where to store its own copy of the distributed database

	kubectl get pvc
	NAME                  STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
	consuldata-consul-0   Bound     pvc-c7a3a3ff-da9d-11e7-80f7-005056b50677   10Gi       RWO            datera-storage-class   34m
	consuldata-consul-1   Bound     pvc-ce3d9357-da9d-11e7-80f7-005056b50677   10Gi       RWO            datera-storage-class   34m
	consuldata-consul-2   Bound     pvc-d6c2912a-da9d-11e7-80f7-005056b50677   10Gi       RWO            datera-storage-class   34m

### Access the Consul cluster
Consul provides a simple HTTP graphical interface listening on port 8500 for admin access. To expose this interface to the external of the kubernetes cluster, define an service as in the ``consul-service.yaml`` configuration file
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: consul-service
  labels:
    app: consul
spec:
  type: NodePort
  ports:
  - name: ui
    port: 8500
    targetPort: 8500
    nodePort: 31085
  selector:
    app: consul
```

Create the service to expose the HTTP admin port of Consul

    kubectl create -f consul-service.yaml

Point the browser to the http://kubew05:31085/ui to access the GUI.

### Cleanup everything
Remove every object we created in the previous steps

    kubectl delete pod consul-2 consul-1 consul-0
    kubectl delete sts consul
    kubectl delete svc consul-service consul
    kubectl delete pvc consuldata-consul-0 consuldata-consul-1 consuldata-consul-2
