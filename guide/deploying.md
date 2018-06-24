# Deploying the Datera Driver for Kubernetes
Datera DSP integrates in Kubernetes through the Datera storage driver based on the FlexVolume framework. Before you install the Datera storage driver, check the Kubernetes version that you have: 

    $ kubectl version

Supported versions of Kubernetes are:

|Kubernetes versions|Datera versions|Repository|Driver versions|
|-------------------|---------------|----------|---------------|
|1.5 or prior|2.0|docker.io/dateraio/kube-dev|1.5|
|1.6 or higher|2.2.6 or higher|docker.io/dateraio/kube-dev|latest|

## Worker nodes prerequisites
Datera DSP uses the industry standard iSCSI protocol to communicate to the worker nodes.  As such the main client requirement is a network connection to the Datera and iSCSI support.  Along with iSCSI support there are a few linux packages required for a successful deployment:

  * **sg3_utils** – collection of tools for SCSI devices.  Check with Operating System guides for installation instructions.
  * **mkfs** – standard tool for the creation of file system.  Please ensure the tools for each file system type are installed.
  * **udev** – Udev is an implementation of devfs/devfsd in userspace.  Datera requires an udev.rule update, deployed as part of the Datera system.

## Verify Workers with Datera DDCT
The Datera Deployment Checking Tool (DDCT) checks the current state of the client to determine if it is correctly set up for usage with the Datera DSP cluster for usage by any ecosystem (for example, Kubernetes) or bare metal.

A non-comprehensive list of checks made on the client are below.

Settings for:

  * ARP
  * IRQ
  * CPU Frequency
  * Block Devices
  * Multipath
  * MTU
  * UDEV

As well as connection tests for:

  * Management Plane
  * Data Plane
  * MTU

These checks are performed to ensure the greatest chance of success during the deployment of a Datera DSP system.
DDCT can be downloaded directly from [github](http://github.com/datera/ddct).

### Basic usage of the DDCT:

Clone the repository

    $ git clone http://github.com/Datera/ddct
    $ cd ddct

Installation is currently only supported on Ubuntu/CentOS systems

    $ ./install.py

DDCT is now installed.  Use ``/home/ubuntu/ddct/ddct`` to run DDCT. The generated config file is located at ``/home/ubuntu/ddct/ddct.json``. Edit the generated config file and replace the IP addresses and credentials with those of your Datera DSP cluster.

```json
{
    "mgmt_ip": "172.19.1.41",
    "password": "password",
    "username": "admin",
    "vip1_ip": "172.28.41.9",
    "vip2_ip": "172.29.41.9"
}
```

Finally, run the tool

    $ ./ddct check ddct.json	

A report will be generated indicating the current status of each check and what needs to be modified for the check to pass.  An example output can be seen on the Github page. Additional documentation for DDCT can be found on the Github page. Note: Datera recommends running the tool multiple times until all checks show “SUCCESS”.

## Check the kubelet configuration
Datera driver requires kubelet to disable attacher-detacher, which is set to “enabled” by default. This is needed as flex volume framework callouts in attach mode, do not supply the necessary flexVolume options that are needed at Datera driver to configure the storage correctly and with appropriate labels (policies). These options convey the storage personality and defines necessary policies at Datera Storage System level. Kubelet configuration needs to have this option set as ``—enable-controller-attach-detach=false`` on all the worker nodes.

## Install the Datera Driver
Install the Datera Driver as a daemonset:

    $ kubectl create -f http://datera.io/assets/files/datera-k8s-installer.yaml

This will create a new namespace called ``datera`` then deploy a Kubernetes daemonset.  

Verify that the Datera installer agents are running on all the worker nodes.

    $ kubectl get pods –-namespace=datera

    NAMESPACE     NAME                                  READY       STATUS    RESTARTS AGE
    datera        datera-installer-agent-0h0               1/1       Running      3          17d
    datera        datera-installer-agent-5lsds             1/1       Running      3          17d
    datera        datera-installer-agent-98brv             1/1       Running      4          18d
    datera        datera-installer-agent-9xd1x             1/1       Running      3          17d
    datera        datera-installer-agent-f0fxs             1/1       Running      1          18d
    datera        datera-installer-agent-fdghn             1/1       Running      3          17d
    datera        datera-installer-agent-ttstb             1/1       Running      3          17d

In addition, in the case of any worker node addition or deletion, the Datera driver will automatically get deployed or terminated on the node without any user intervention. The Datera driver is fully integrated daemon-set in the Kubernetes framework. 

## Placement and Communication of Key Datera Driver Components
The diagram below shows the placement of the Datera Dynamic provisioner on the Kubernetes Master nodes, the placement of the Datera driver daemon set on the Kubernetes worker nodes, and a high-level network topography on how the components communicate.

![](./img/datera-kube-placement.png?raw=true)
