# Configuring Kubernetes with Datera storage
The Kubernetes worker nodes access the Datera storage through the iSCSI ports. iSCSI uses the same network stack for storage as does the compute layer. Care should be taken to setup the networking in accordance with Datera installation guide.

## Network Preparation
Datera network topology typically looks like the following schematic:

![](./img/datera-logical-network-topology.png?raw=true)

Multiple private networks help with Datera inter-node communications and management. The Access network is the storage network that the client nodes (e.g. the Kubernetes worker nodes) use to communicate with the storage. This access network is usually dedicated for storage; public access networks are segregated into other VLANs. In planning for Datera networking all these different network segments should be architected and deployed in their own VLANs for optimal storage performance. 

## Datera Access Network
The Kubernetes worker nodes access the Datera storage through the Access Network iSCSI ports. Each Datera node connects to the Access Network through one or two 10 Gigabit Ethernet (GbE) interfaces. The Access Network must be a traditional IP based network. The target IP addresses float during node failure, and target port rebalancing activities that occur as part of the load redistribution activities.

The Access Network uses floating IP addresses to present iSCSI LUNs to hosts. The IP address range you provide must at least be equal to the number of nodes in the Datera storage system. However, the more IP addresses available the better the load distribution for the system. Datera recommends at least 32 addresses for small systems with up to 10 nodes, 64 addresses for medium systems up to 20 nodes, and 128-250 addresses for large systems greater than 20 nodes. For more details on the latest recommendations, please refer to the Datera Installation Guide and Datera User’s Guide. 

The target IP addresses presented by the target iSCSI ports are drawn from the Access Network IP pools. The IP address in the pools are defined by the Access Network IP blocks. You must create at least one IP pool and at least one block of IP addresses to be assigned to Storage Instances. In the user interfaces, the Access Network ports are identified as Access VIP 1 and Access VIP 2.
