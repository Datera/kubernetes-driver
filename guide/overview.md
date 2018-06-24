# Data Services Platform Overview
The Datera® Data Services Platform (DSP™) is a software-defined data infrastructure for virtualized environments, databases, cloud stacks, DevOps, microservices and container deployments. It provides operations-free delivery and orchestration of data at scale for any application within a traditional datacenter, private cloud or hybrid cloud setting. 

DSP provides continuously available service level objective (SLO)-based data services for applications including monolithic, cloud enabled or cloud native designs. DSP may be used as a data infrastructure for enterprise storage, on or off-premises private or hybrid storage clouds, or as service provider infrastructure to enable delivery of AWS-like data services in the public cloud. 

DSP is deployed as a scale-out configuration of industry standard x86 servers containing a heterogeneous mix of commodity storage media types including persistent memory, NVMe flash, SATA flash, 3D XPoint memory and/or conventional HDDs.

![](./img/datera-dsp-overview.png?raw=true)

## Benefits of the Datera Data Services Platform
Datera DSP architecture is based on a comprehensive set of design objectives to ensure the delivery of best-in-class levels of agility, extensibility, performance and economics.  

The DSP innovative design includes:

  * Automated management of data services based on individual application SLOs
  * Multi-tenant support with resource isolation and per-tenant QoS controls 
  * Infrastructure-as-code provisioning and as-a-service resource consumption models 
  * Cloud-based monitoring and predictive analytics
  * Data security including user authentication and at-rest data encryption
  * Delivery of sub-100 microsecond read/write latencies
  * Elastic performance, latency, capacity and cost
  * Extensibility to quickly adopt next-generation technologies
  * Transparent dynamic data migration
  * Resilience to withstand media, server, network, datacenter and/or electrical grid failures
  * Deployment on industry-standard x86 hardware with commodity storage media


In addition, the policy framework allows configurability of IOPS and bandwidth at an application or volume level. DSP provides a wide range for $/GB and $/IOPS on a per volume basis, thereby providing economical elasticity for a wide variety of cloud applications. More importantly, DSP automates IT operations on the system through an adaptive data infrastructure. 

Given the wide variety of applications hosted in modern infrastructures, DSP continually optimizes price and performance by matching application SLOs with the right composition of infrastructure. DSP transparent data migration allows for data to be placed intelligently across the physical nodes, maintaining the intended service level objectives for any application workload. 


## Data Services Platform Advantages for Containers 
The Data Services Platform is perfectly matched for containerized environments:

  * **Docker, Mesos, and Kubernetes Integration**: Datera provides simple installation of a plugin to automatically integrate with all major container orchestration environments including Kubernetes

  * **Distributed Persistence**: For distributed applications, there is a need for a distributed infrastructure layer providing a common operational framework for stateful and stateless workloads. This can provide consistent scale-out access anywhere in the cluster for both ephemeral and persistent storage tiers. In addition, DSP optimizes for distributed access by using container locality, infrastructure failure domains, and resource segregation.

  * **Grow as you go Model**: Start small and be able to scale fast. Traditional storage is limited in scale by the proprietary hardware frame size and performance, while Datera delivers “heterogeneous COTS scale-out.” Software-based systems will organically evolve—grow with new hardware and decommission obsolete hardware without any disruption. Data gets rebalanced and access optimized for the runtime workloads. No explicit data migration. No forklift upgrades.

  * **Container Scale and Velocity**: Containers are at least 10x denser than virtual machines. Given the scale and velocity of transactions, the storage infrastructure has to support low latency, fast operations, and scale of storage objects. Datera provides the speed of provisioning and deployment for containers at scale. 

  * **SLO based Container Deployment**: Application specific templates define the I/O characteristics of an application environment. Templates can define broad service classes or specific requirements for each workload in an application environment. Datera maps the intent defined by the app template to the available storage when volumes are created. Each volume is unique, allowing composable, on demand storage infrastructure. The Datera system allows for storage to be placed intelligently maintaining the intended services level objectives for any application workload.

  * **DevOps Operations Model**: Datera provides application driven real-time resource consumption, evolving datacenters from slow infrastructure centric IT to agile DevOps centric IT. Application requirements are defined in SLO’s that deeply shape the fabric, accommodating failure domains, network topology, multi-tenancy, workload isolation, etc. A REST API makes the platform programmable and allows instantiating entire storage clouds with single API calls (datacenter as code). Datera transforms datacenter speed, agility and experience.

  * **Transparent DevOps to IT Ops transition**: Datera allows for application related storage provisioning to be decoupled from the physical infrastructure management. Moreover, customers can seamlessly transition from development to test to IT operations without application configuration changes but morphing the infrastructure constraints through policy overrides based on application, user or tenant.

  * **Wide Price/Performance aperture**: Given the wide variety of containerized applications, Datera provides the elastic price bands that fit the economic value for particular data. Each volume’s requirements are composable on demand and that articulates storage placement. Datera system allows for storage to be placed intelligently maintaining the intended services level objectives for any application workload, even in multi-tenant clouds.


## Application Template Provisioning 
While the DSP allows for traditional storage provisioning, such as volumes and data services, it also enables a modern application driven provisioning model. With application driven provisioning, the workload or application view of storage and its associated services are modeled through an application template (AppTemplate) 

Some common applications, such as Cassandra, MySQL, Hadoop, VMware environments and test/development workloads, are pre-modeled through default templates. Users may also build custom templates from scratch or by cloning and modifying existing templates. 

The AppTemplate acts as the interface used to express SLOs on a per-application basis, by configuring application specific characteristics such as: 

  * Host access controls (initiators or hosts accessing the data)
  * Host targets or exports
  * Data volumes
  * Performance
  * Security
  * Data durability requirements
  * Data management services, such as snapshots, and their schedule
 
![](./img/app-template-layout.png?raw=true)

AppTemplates may be used to instantiate and deploy one or more application instances (AppInstance.) An AppInstance is the provisioned entity within the system. 
