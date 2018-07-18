# Working with Role Based Access control (RBAC) enabled
In all the previous examples, we assumed Role Based Access control (RBAC) is disabled on your Kubernetes cluster. As starting from latest Kubernetes versions, the RBAC is enabled by default on the Kubernetes cluster if you are installing it via the "kubeadm" installation tool.

The Role Based Access Control is a general-purpose authorizer that regulates access to cluster resources based on the roles of groups, individual users, and including applications through their service accounts . Please, refer to the Kubernetes documentation for details about RBAC and how it works.

## RBAC authorization for Datera dynamic storage provisioner
This guide reports an example of RBAC settings for the the Datera dynamic provisioner. Please, note that the RBAC capability of Kubernetes is vast and this is just a reference for getting Datera working on Kubernetes when RBAC is enabled. This is only an example without support from Datera.

Because of the RBAC access control, we have to define a dedicated Service Account for Datera dynamic provisioner and give it permissions in the ``datera`` namespace. Define a service account as in the ``datera-sa.yaml`` configuration file

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: datera-k8s-external-provisioner
  namespace: datera
```

Create the service account

    $ kubectl create -f datera-sa.yaml

The Datera dynamic provisioner deploy should be edited 

    $ kubectl edit deploy datera-provisioner-agent

and updated with a direct reference to use the new service account 

```
...
    spec:
      serviceAccount: datera-k8s-external-provisioner
      containers:
      - name: datera-provisioner
...
```

Define a Cluster Role set of permissions as for the following ``datera-cluster-role.yaml`` configuration file.

```yaml
# Datera Kubernetes RBAC Example
# This is an example of RBAC settings for the Datera dynamic provisioner
# The RBAC capabilities of Kubenetes is vast and this is just a reference
# Our expectation is this will provide guidance for those customers leveraging RBAC
# As this is an example with no support from Datera
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: datera-storage-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
```

A Cluster Role object can be used to grant the permissions at cluster level, instead of a single namespace. It can also be used to grant access to cluster-scoped resources, like Persistent Volume and Storage Class, or namespaced resources, like Persistent Volume Claim, across all the namespaces.

Make sure your current user has permissions to create Cluster Roles objects

    $ kubectl auth can-i create clusterroles
    yes

and create the role 

    $ kubectl create -f datera-cluster-role.yaml

The role above can be assigned, for example, to the Datera Service Account you created before by defining a Cluster Role Binding object between the Cluster Role above and the Datera Service Account. Define the binding as in the ``datera-cluster-role-binding.yaml`` configuration file

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: datera-storage-provisioner
subjects:
  - kind: ServiceAccount
    name: datera-k8s-external-provisioner
    namespace: datera
roleRef:
  kind: ClusterRole
  name: datera-storage-provisioner
  apiGroup: rbac.authorization.k8s.io
```

Create the binding

    $ kubectl create -f datera-cluster-role-binding.yaml

The Datera dynamic provisioner should now have all the permissions to work in a Kubernetes environment with RBAC enabled.

## RBAC authorization for users working with Datera
The user going to create claims bound to Datera volumes needs to access the secret used to login to the Datera system. If you created that secret in the ``datera`` namespaces, make sure the user has the permissions to read secrets in that namespace.




