# Working with StatefulSets and Datera
To properly show StatefulSets in action with Datera DSP, we’ll implement a simple clustered data store based on HashiCorp Consul. Consul is a distributed key-value database with service discovery capabilities. Consul clustering is based on the Raft algorithm for distributed consensus about the cluster status. Details about Consul and how to configure and use it can be found on the Consul documentation. This is only an example, and you can easily extend it to the distributed datastore of your choice like MongoDB, RethinkDB or Cassandra, just to name few.

The most difficult part to run a Consul cluster on Kubernetes is how to form a cluster having Consul strict requirements about instance identities of nodes being part of it. StatefulSet is the most obvious choice for implementing Consul on Kubernetes thanks to the pod hostname predictability.

Also, we assume a persistent storage environment based on Datera DSP is available. This is because each Consul node uses a data directory where to store its data and this directory needs to be preserved across pod deletions and restarts.

![](./img/consul-cluster.png?raw=true)

In this example, we are going to deploy a three nodes Consul cluster by using the StatefulSet controller and Datera DSP as persistent storage environment. To deploy the cluster, you need to create the following objects:

  * Persistent Volumes for storing the data of each Consul instance.
  * StatefulSet controller.
  * Headless service for making Consul pods discoverable.

## Persistent Volumes
Since we are using dynamic storage provisioning on Kubernetes, you do not need to manually create the Persistent Volumes on the Datera DSP storage. Persistent Volumes are automatically created on-demand when users claim for storage.

## StatefulSet Headless Service
A headless service is required to make the Consul pods discoverable by their hostnames. Service discovery is an important feature of Kubernetes and, in most of the cases, it is based on the embedded DNS add-on. This step is required because the Consul cluster formation is made by Consul instances discovering each other by their hostnames. Before to go forward with this example, please make sure your Kubernetes setup is running an embedded DNS add-on.

Define a headless service for the Consul cluster as the ``consul-svc.yaml`` file

```yaml
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

Then, create it

    $ kubectl apply -f consul-svc.yaml

The service above exposes all the ports of the Consul instances and makes each instance discoverable by its predictable hostname in the form of ``$(statefulset name)-$(ordinal).$(service name).$(namespace).svc.cluster.local``. Thanks to the headless service, all Consul pods will get a discoverable hostname.

## Create the config map
Before you create the StatefulSet for the Consul cluster, you’ll create a ConfigMap containing a Consul configuration file. The file ``consul.json`` below contains all the settings parameters required by Consul to form a cluster of three Consul instances.

```json
{
  "datacenter": "kubernetes",
  "log_level": "DEBUG",
  "data_dir": "/consul/data",
  "server": true,
  "bootstrap_expect": 3,
  "retry_join": ["consul-0.consul.default.svc.cluster.local","consul-1.consul.default.svc.cluster.local","consul-2.consul.default.svc.cluster.local"],
  "client_addr": "0.0.0.0",
  "bind_addr": "0.0.0.0",
  "domain": "cluster.local",
  "ui": true
}
```

Create the Config Map for Consul configuration file

    $ kubectl create configmap consulconfig --from-file=consul.json

## Create the StatefulSet
Define a Stateful Set for the Consul cluster as in the ``consul-sts.yaml`` configuration file

```yaml
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
      initContainers:
      - name: consul-config-data
        image: busybox:latest
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh", "-c", "cp /readonly/consul.json /config && sed -i s/default/$(POD_NAMESPACE)/g /config/consul.json"]
        env:
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        volumeMounts:
        - name: readonly
          mountPath: /readonly
          readOnly: true
        - name: config
          mountPath: /config
          readOnly: false
      containers:
      - name: consul
        image: consul:1.0.2
        imagePullPolicy: IfNotPresent
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
        - name: data
          mountPath: /consul/data
          readOnly: false
        - name: config
          mountPath: /consul/config
          readOnly: false
        args:
        - consul
        - agent
        - -config-file=/consul/config/consul.json
      volumes:
        - name: readonly
          configMap:
            name: consulconfig
        - name: config
          emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: datera
``` 

You will notice the presence of an init container in the pod template in addition to the main Consul container. Both the containers, the init and the main container mount, as a shared volume, the same Consul configuration file (in the form of a ConfigMap you just created early).

The role of the init container is to update the consul configuration file according to the namespace where that pod is running. This is accomplished by accessing the pod metadata by the API server running in the Kubernetes Control Plane. This step is required because the discoverability of each Consul instance is tied to the namespace where the instance is running, i.e. Consul instances running in different namespaces are named differently.  

Create the StatefulSet

    $ kubectl apply -f consul-sts.yaml
 
and list the pods

    $ kubectl get pods -o wide
    NAME       READY     STATUS          RESTARTS   AGE       IP           NODE
    consul-0   1/1       Running         0          1d        10.38.1.25   kubew03
    consul-1   1/1       PodInitializing 0          1d        10.38.2.28   kubew05

The first pod has been created. The second pod will be created only after the first one is up and running, and so on. 

StatefulSets behave this way because some stateful applications can fail if two or more cluster members come up at the same time. For a StatefulSet with N replicas, when pods are deployed, they are created sequentially, in order from {0...N-1} with a sticky, unique identity in the form $(statefulset name)-$(ordinal). The (i)th pod is not created until the (i-1)th is running.

This ensure a predictable order of pod creation. However, if the order of pod creation is not strictly required, it is possible to create pods in parallel by setting the ``podManagementPolicy: Parallel`` option in the StatefulSet template.

List the pods again to see how the pod creation is progressing:

    $ kubectl get pods -o wide
    NAME       READY     STATUS          RESTARTS   AGE       IP           NODE
    consul-0   1/1       Running         0          1d        10.38.1.25   kubew03
    consul-1   1/1       Running         0          1d        10.38.2.28   kubew05
    consul-2   1/1       Running         0          1d        10.38.3.28   kubew04

Now all pods are running and forming the initial cluster.

## Check the cluster membership
To form the cluster, Consul pods will discover each other by looking up for the peers in the embedded DNS cache. Because of the headless service you created before, each pod has a predictable hostname.

```bash
for i in $(seq -w 0 2); \
do \
    kubectl exec consul-$i -- sh -c 'hostname -f'; \
done

consul-0.consul.project.svc.cluster.local
consul-1.consul.project.svc.cluster.local
consul-2.consul.project.svc.cluster.local
```

Check the cluster membership by querying one of the three Consul pods   

      $ kubectl exec -it consul-0 -- consul members

      Node      Address          Status  Type    Build  Protocol  DC          Segment
      consul-0  10.38.1.26:8301  alive   server  1.0.2  2         kubernetes  <all>
      consul-1  10.38.2.28:8301  alive   server  1.0.2  2         kubernetes  <all>
      consul-2  10.38.3.28:8301  alive   server  1.0.2  2         kubernetes  <all>

## Storing data in the cluster
The PersistentVolumeClaim template in the StatefulSet definition of our Consul cluster was used to create the PersistentVolumeClaim belonging to each Consul pod.

List the created Persistent Volume Claims.

      $ kubectl get pvc -o wide
      NAME            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
      data-consul-0   Bound     pvc-614e3324-7648-11e8-9d43-000c299341a5   1Gi        RWO            datera         1m
      data-consul-1   Bound     pvc-6ef0d057-7648-11e8-9d43-000c299341a5   1Gi        RWO            datera         1m
      data-consul-2   Bound     pvc-7a268ba6-7648-11e8-9d43-000c299341a5   1Gi        RWO            datera         1m

The names of the created Persistent Volume Claims are derived from the name of claim defined in the volumeClaimTemplate and the name of each pod.

You can also see that each PersistentVolumeClaim is now bounded to the Persistent Volumes you created before. This means that each pod is accessing a Datera storage physical volume for storing its data and this volume is strictly reserved to that pod.

To store some data in Consul, create a key/value and send a PUT request to the Consul datastore.

      $ kubectl exec -it consul-0 -- curl --request PUT --data my-data http://consul:8500/v1/kv/my-key
      true

To read data, send a GET request.

      $ kubectl exec -it consul-0 -- curl --request GET --data my-data http://consul:8500/v1/kv/my-key
      [{"LockIndex":0,"Key":"my-key","Flags":0,"Value":"bXktZGF0YQ==","CreateIndex":17314,"ModifyIndex":17314}]

## Pod failover
To simulate a fail, delete one of the pods

      $ kubectl delete pod consul-1

Checking the pod status progression

      $ kubectl get pods consul-1 -o wide --watch

      NAME       READY     STATUS           RESTARTS   AGE       IP           NODE
      consul-1   0/1       Terminating      0          13m       <none>       kubew04
      consul-1   0/1       Pending          0          0s        <none>       <none>
      consul-1   0/1       Pending          0          0s        <none>       kubew04
      consul-1   0/1       Init:0/1         0          0s        <none>       kubew04
      consul-1   0/1       PodInitializing  0          15s       10.38.5.31   kubew04
      consul-1   1/1       Running          0          16s       10.38.5.31   kubew04

As expected, a new pod is recreated with different IP address but with the same identity. Now check if the rescheduled pod is using the same previous Datera storage volume

```
$ kubectl describe pod consul-1

...
Volumes:
  data:
    Type:       PersistentVolumeClaim 
    ClaimName:  data-consul-1
    ReadOnly:   false
...
```

As you can see, the rescheduled pod is using the same Persistent Volume Claim as the previous one. Please, remember that deleting a pod in a StatefulSet does not delete the Persistent Volume Claims used by that pod. This preserves the data persistence across pod deletion and rescheduling.

## Scaling the cluster
Scaling down a StatefulSet and then scaling it up is similar to deleting a pod and waiting for the StatefulSet to recreate it. Please, remember that scaling down a StatefulSet only deletes the pods, but leaves the Persistent Volume Claims. Also, please, note that scaling down and scaling up is performed similar to how pods are created when the StatefulSet is created. When scaling down, the pod with the highest index is deleted first: only after that pod gets deleted, the pod with the second highest index is deleted, and so on.

What is the expected behavior scaling up the Consul cluster? Since the Consul cluster is based on the Raft algorithm, we have to scale up our 3 nodes cluster by 2 nodes at same time because an odd number of nodes is always required to form a healthy Consul cluster. We also expect a new Persistent Volume Claim is created for each new pod.

Scale the StatefulSet.

    $ kubectl scale sts consul --replicas=5

By listing the pods, we see our Consul cluster gets scaled up.

    $ kubectl get pods -o wide
    NAME       READY     STATUS          RESTARTS   AGE       IP           NODE
    consul-0   1/1       Running         0          1d        10.38.1.25   kubew03
    consul-1   1/1       Running         0          1d        10.38.2.28   kubew05
    consul-2   1/1       Running         0          1d        10.38.3.28   kubew04
    consul-3   1/1       Running         0          1m        10.38.1.27   kubew03
    consul-4   1/1       Running         0          1m        10.38.5.32   kubew04

Also check the dynamic Datera storage provisioner created the additional volumes.

    kubectl get pvc -o wide
    NAME            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    data-consul-0   Bound     pvc-614e3324-7648-11e8-9d43-000c299341a5   1Gi        RWO            datera         1d
    data-consul-1   Bound     pvc-6ef0d057-7648-11e8-9d43-000c299341a5   1Gi        RWO            datera         1d
    data-consul-2   Bound     pvc-7a268ba6-7648-11e8-9d43-000c299341a5   1Gi        RWO            datera         1d
    data-consul-3   Bound     pvc-c0d40c2b-77aa-11e8-9d43-000c299341a5   1Gi        RWO            datera         2m
    data-consul-4   Bound     pvc-d2ef5889-77aa-11e8-9d43-000c299341a5   1Gi        RWO            datera         1m

## Node failover
In case of node fails or becomes unreachable from Kubernetes, the StatefulSet behaves different from other replication controllers. To avoid "split-brain" effects in the case when a node becomes unreachable, the pods running on that node are marked as "unknown", and aren't rescheduled on the remaining nodes, unless the node object is forcefully deleted from Kubernetes cluster. This makes StatefulSets a great fit for distributed datastores like Consul, Cassandra or Elasticsearch.

To simulate a network partition, login to one of the worker node where Consul pods are running and stop the kubelet agent.

    $ ssh kubew04
    $ sudo systemctl stop kubelet
    $ exit

After a while, the worker node becomes unreachable

    $ kubectl get nodes
    
    NAME      STATUS     ROLES     AGE       VERSION
    kubem00   Ready      master    24d       v1.10.3
    kubew03   Ready      worker    22d       v1.10.3
    kubew04   NotReady   worker    22d       v1.10.3
    kubew05   Ready      worker    22d       v1.10.3

And pods running on that nodes are marked as unknown after expiration of a timeout. It will take 5 minutes by design and it is controlled by the ``--pod-eviction-timeout`` parameter in the controller manager.

    $ kubectl get pods -o wide

    NAME       READY     STATUS    RESTARTS   AGE       IP           NODE
    consul-0   1/1       Running   0          3h        10.38.1.26   kubew03
    consul-1   1/1       Unknown   0          3h        10.38.5.31   kubew04
    consul-2   1/1       Running   0          3h        10.38.2.29   kubew05
    consul-3   1/1       Running   0          20m       10.38.1.27   kubew03
    consul-4   1/1       Unknown   0          19m       10.38.5.32   kubew04

Because Kubernetes lost the communication with the worker node, it cannot say nothing about the state of the worker node and pods running on it. However, the Consul cluster is still running:

    $ kubectl exec -it consul-0 -- consul members

    Node      Address          Status  Type    Build  Protocol  DC          Segment
    consul-0  10.38.1.26:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-1  10.38.5.31:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-2  10.38.2.29:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-3  10.38.1.27:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-4  10.38.5.32:8301  alive   server  1.0.2  2         kubernetes  <all>

To avoid split-brain side effects on the Consul cluster, Kuberentes is not going to reschedule the unknown pods until the node object is deleted.

    $ kubectl delete node kubew04

Then kubernetes will reschedule the unknown pods on the remaining nodes

    $ kubectl get pods -o wide

    NAME       READY     STATUS    RESTARTS   AGE       IP           NODE
    consul-0   1/1       Running   0          3h        10.38.1.26   kubew03
    consul-1   1/1       Running   0          3m        10.38.2.31   kubew05
    consul-2   1/1       Running   0          3h        10.38.2.29   kubew05
    consul-3   1/1       Running   0          36m       10.38.1.27   kubew03
    consul-4   1/1       Running   0          3m        10.38.1.28   kubew03

This leads to an update of the Consul cluster

    $ kubectl exec -it consul-0 -- consul members
    
    Node      Address          Status  Type    Build  Protocol  DC          Segment
    consul-0  10.38.1.26:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-1  10.38.2.31:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-2  10.38.2.29:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-3  10.38.1.27:8301  alive   server  1.0.2  2         kubernetes  <all>
    consul-4  10.38.1.28:8301  alive   server  1.0.2  2         kubernetes  <all>

## Exposing the cluster
Consul provides a simple HTTP graphical interface on port 8500 for interact with it. To expose this interface to the external of the kubernetes cluster, define a service as in the ``consul-ui.yaml`` configuration file

```yaml
apiVersion: v1
kind: Service
metadata:
  name: consul-ui
  labels:
    app: consul
spec:
  type: ClusterIP
  ports:
  - name: ui
    port: 8500
    targetPort: 8500
  selector:
    app: consul
```

Assuming we have an Ingress Controller in place, define the ingress as in the ``consul-ingress.yaml`` configuration file

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: consul
spec:
  rules:
  - host: consul.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: consul-ui
          servicePort: 8500
```

Create the service and expose it

    $ kubectl create -f consul-ui.yaml
    $ kubectl create -f consul-ingress.yaml

Point the browser to the [consul.example.com/ui](http://consul.example.com/ui) to access the GUI.

## Deleting the cluster
To delete the Consul cluster, delete the StatefulSet in cascade mode. This will delete both the StatefulSet objects and all pods. Please remember that the order of pods deletion will be from the pod having the highest index. Also, you will have to delete the Persistent Volume Claims and then all Persistent Volumes but only in case of manual storage provisioning.

In our case, since we used the dynamic storage provisioner, all the Persistent Volumes will be deleted or retained according to the reclaim policy specified in the Storage Class.

    $ kubectl delete pod consul-4 consul-3 consul-2 consul-1 consul-0
    $ kubectl delete sts consul
    $ kubectl delete svc consul consul-ui
    $ kubectl delete ingress consul
    $ kubectl delete pvc consuldata-consul-0 consuldata-consul-1 consuldata-consul-2 consuldata-consul-3 consuldata-consul-4
