# Stateful Applications in Kubernetes
Common Kubernetes controllers as Replica Sets and Daemon Sets are a great way to run stateless applications, but their semantics are not friendly for deploying all stateful applications. A better approach for deploying stateful applications and distributed systems on Kubernetes, is to use the Stateful Sets. 

StatefulSets are valuable for applications that require one or more of the following:

  * Stable, unique network identifiers.
  * Stable, persistent storage.
  * Ordered, graceful deployment and scaling.
  * Ordered, graceful deletion and termination.

In this scenario, stable is synonymous with persistence across Pod (re)scheduling. If an application doesn’t require any stable identifiers or ordered deployment, deletion, or scaling, you should deploy your application with a controller that provides a set of stateless replicas.

## Pod Identity
StatefulSet Pods have a unique identity that is comprised of an ordinal, a stable network identity, and stable storage. The identity sticks to the Pod, regardless of which node it’s (re)scheduled on. Each pod created by a StatefulSet is assigned an ordinal index (starting from zero), which is then used to derive the pod’s name and hostname, and to attach stable storage to the pod. 	

## Ordinal Index
For a StatefulSet with N replicas, each Pod in the StatefulSet will be assigned an integer ordinal, in the range [0,N), that is unique over the Set. 

## Stable Network 
Each Pod in a StatefulSet derives its hostname from the name of the StatefulSet and the ordinal index of the Pod. The names of the pods are thus predictable as $(statefulset name)-$(ordinal). 

## Rescheduling Pods
The identity sticks to the Pod, regardless of which node it’s (re)scheduled on. When a pod instance managed by a StatefulSet disappears, because the node the pod was running on has failed, it was evicted from the node, or the user manually deleted the pod, the StatefulSet makes sure it is replaced with a new replica, but the replacement pod gets the same name and hostname as the replica that has disappeared. The new pod isn’t necessarily scheduled to the same node, but it will still be available and reachable with the same name and hostname as before. 

## Scaling a StatefulSet
Scaling a StatefulSet refers to increasing (up) or decreasing (down) the number of replicas. This is accomplished by updating the replicas field. You can use either kubectl scale or kubectl patch to scale a Stateful Set. Scaling up the StatefulSet creates a new pod instance with the next unused ordinal index. For example, if you scale up from two to three instances (having index 0 and 1), then the new instance will get index 2.

The interesting part about scaling down a StatefulSet is that you always know the pod that will be removed. This is in contrast to scaling down a ReplicaSet, where do you have no control on what instance will be deleted, and you cannot specify which replica you want to remove first. Scaling down a StatefulSet always removes the instances with the highest ordinal index first. StatefulSets make the effects of a scaling up and scaling down a predictable process.

StatefulSets scale up and down only one replica at time. This makes StatefulSets very suitable to manage certain stateful applications, as, for example, distributed databases as Consul, MongoDB, and Cassandra. These kind of applications may lose their data if multiple nodes go down at the same time. For this reason, StatfulSets never permits scaling down if any of the pod is unhealthy.

## Persistent Storage
Each pod of a StatefulSet needs to use its own storage, and when the pod is eventually replaced with a new instance, the new instance must have the same storage attached to it. Also, the storage for a pod in a StatefulSet, needs to be persistent and decoupled from the pod itself. Kubernetes implements this by means of the Persistent Volumes and Persistent Volume Claims model which allows persistent storage to be attached to a pod by referencing the Persistent Volume Claim in the pod definition. 

## Persistent Volume Claims Template
In StatefulSets, each pod needs to point to a different Persistent Volume Claim to have its own separate Persistent Volume where to store its data. For this reason, a StatefulSet definition will have a pod template and one or more Persistent Volume Claims templates, which maps the pod with its own Persistent Volume Claims. The Persistent Volumes for those claims can either be statically provisioned on the Datera backend by the cluster administrator or dynamically through the usage of the dynamic Datera storage provisioner. Datera is supporting both the options.

## Persistent Volume Claims creation and deletion
By scaling up a StatefulSet, multiple objects are created: the pod and one or more Persistent Volume Claims referenced by the pod in the pod template. On the other side, scaling down deletes only the pod, leaving the Persistent Volume Claims. This is because if a claim is deleted, the Persistent Volume it was bound to gets recycled, retained, or deleted and its contents are lost. For this reason, users are required to delete Persistent Volume Claims manually to release the bound Persistent Volume.

## Deleting StatefulSets
StatefulSet supports both Non-Cascading and Cascading deletion. In a Non-Cascading Delete, the StatefulSet’s Pods are not deleted when the Stateful Set is deleted. In a Cascading Delete, both the StatefulSet and its Pods are deleted. One will have to delete the StatefulSet followed by the Persistent Volume Claim and then Persistent Volume. 
