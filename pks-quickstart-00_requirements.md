# Requirements for Installing PKS with NSX-T

Add here: https://code.vmware.com/samples?categories=Sample&keywords=&tags=vmcode-cna-devcenter&groups=&filters=&sort=dateDesc&page=

For example: https://github.com/kacole2/pks-k8s-tests

## Overview
This guide provides instructions for installing VMware PKS in an NSX-T environment.

## Prerequisites
You will need the following baseline environment to install PKS with NSX-T as described here.

### Compute and Storage
- vSphere v6.5 U2 installed with Enterprise Plus license
- 2 ESXi hosts with 256GB RAM and 1TB disk each
- vCenter Server appliance installed and configured
- vSphere client authenticated with vCenter Server
- vSAN datastore configured with vSphere 6.5
- Project Hatchway configured for Persistent Storage Volumes

To install vSphere with vSAN, refer to the vSphere documentation suite: https://docs.vmware.com/en/VMware-vSphere/index.html. See also the video "Preparing vSphere for PKS" <https://youtu.be/WLOUfQgnbUE>.

### Networking and Security
- NSX-T 2.2 installed
- ESXi hosts prepped as transport nodes
- 1 NSX-T Manager node
- 1 NSX-T Controller node
- 2 NSX-T Edge nodes (bare metal edge nodes not suported for PKS)

To install NSX-T, refer to the NSX documentation suite: https://docs.vmware.com/en/VMware-NSX-T/. See also the video "Preparing NSX-T for PKS" <https://youtu.be/8u3Lcc1ffsY>.

## Software
This is the list of software that is installed with these instructions.

Software | Description | Link
---------|-------------|-----
vShere and vSAN 6.5 U2 | infrastructure | https://my.vmware.com/web/vmware/details?productId=673&downloadGroup=NSX-T-210
NSX-T 2.2 | Virtual networking | https://my.vmware.com/web/vmware/details?productId=673&downloadGroup=NSX-T-210
PKS Software bundle | PKS | https://network.pivotal.io/products/pivotal-container-service#/releases/43085
PKS CLI | CLI for PKS | https://network.pivotal.io/products/pivotal-container-service#/releases/43085/file_groups/848
Kubectl CLI | CLI for Kubernetes | https://network.pivotal.io/products/pivotal-container-service#/releases/43085/file_groups/847
Ops Mgr for vSphere | https://network.pivotal.io/products/ops-manager
Stemcell for vSphere |	https://network.pivotal.io/products/stemcells
Harbor | Enterprise-class container registry for Docker images |
Wavefront | Cluster analytics |
vRealize Log Insight | Log collection and search |
vRL Ops Manager | Runtime operations management |

## Contents
This section summarizes the PKS installation process for NSX-T environments. Refer to the Instructions section for details on each step.

1. Create CLI virtual machine.
The CLI VM is a Linux (Ubuntu) VM that you create in vSphere. On this VM you install all of the CLIs used to install and operate PKS, including:
- pks cli -- Used by OPS to create/delete and manage K8S Clusters
- kubectl -- Used by DEV to interact with K8S Cluster and deploy applications including scaling
- uaac -- Used by OPS to manage user accounts and authorization for the PKS platform
- bosh -- Used by OPS to manage PKS deployments and provides information about the VMs using its Cloud Provider Interface (CPI) (vSphere)
- om -- Used by OPS to manage and interact with Ops Manager
- nsx-cli.sh -- Used by OP to remove NSX-T objects after a K8S have been deleted
2. Configure NSX-T for PKS
- Verify baseline NSX-T configuration
- Create IP pool
- Create IP block
- Create T0 router
- Create static route to T0 router
- Create logical switches
- Create T0 router port
- Create T1 router
- Create downlink router
- Advertise T1 routes
- Perform validation checks
3. Install OpsManager and Deploy BOSH
- Download OPM to vSphere
- Install OPM VM
- Install BOSH VM
- Configure BOSH
	- Configure PKS AZs
	- Define networking for PKS mgmt VMs
	- Define networking for K8s mgmt cluster
	- Configure security, logging, and resources
- Deploy BOSH
4. Deploy PKS Control Plane
- Download PKS software bundle
- Insall PKS tile in BOSH
- Configure PKS
	- AZs
	- API endpiont and cert
	- Resources for K8 clusters
	- vSphere config
	- NSX-T config
	- IAM config
	- Stemcell VM config
- Deploy PKS
- Verify PKS deployment
5. Deploy Kubernetes Clusters
	- Configure IAM policies
	- Get auth token
	- Create PKS user
	- Assign role
	- Grant OP access to cluster
	- Export BOSH envars
	- Connect to BOSH
- Deploy Kubernetes cluster
    - Login to PKS
	- Create PKS cluster
	- Deploy PKS cluster
	- View cluster creation
	- Get client config file for K8 cluster (to give to devs)
- Deploy application containers
	- Get nodes using kubectl
	- Create K8 namespace
	- Create K8 deployment spec
	- Deploy app
	- Monitor K8 pod creation
	- Get UI pod name
	- Access the app
	- Add load balancer
6. Install Harbor Container Registry
7. Deploy vRealize Log Insight (vRLI) for logging and troubleshooting
8. Deploy vRealize Operpations (vRLops) for managing the infrastructure
9. Deploy Wavefront for Kubernetes cluster metrics

## Concepts
Here are some fundamental concepts to consider.

### Docker Containers
A container is a an application that has been virtualized at the software level rather than the hardware level, that is, without the use of a hypervisor.

Docker is the most common container format and runtime. Dockerfile lets developers create application templates using a simple declarative language that defines an application, its files, and dependencies. The result is an artifact known as a Docker image that can be run as a container on any host running the Docker daemon (dockerd). Docker images are hosted on public or private container registries where they can be easily pushed, pulled and run.

### Container Networking 
When a container is created, it is given a virtual Ethernet device (veth) that is attached to a virtual bridge (docker0). Using Linux namespaces, the veth is mapped to appear as eth0 in the container. The in-container veth0 interface is given an IP address from the bridge’s assigned address range carved from one of the private address blocks defined in RFC1918 (https://en.wikipedia.org/wiki/Bridging_(networking)).  

Docker containers can communicate with each other, but only if they are on the same host (and thus the same virtual bridge). Docker containers on different hosts cannot connect.

### Kubernetes
Kubernetes provides container scheduling and management services for running containerized applications at scale.

Kubernetes is deployed on nodes of two types:
- Master node: API server, key value store (etcd), scheduler, controller, watcher
- Worker node: Kubelet agent, Kube proxy, container runtime (dockerd)

A kubernetes cluster comprises master and worker nodes, as well as a key value store (etcd) and a load balancer to manage external network traffic into cluster nodes. A node is deployed to a virtual machine. In practice, a Kubernetes clusters has redundant master nodes and a distributed key-value store supporting several worker nodes with highly avialable load balancing:
- 2 master node VMs
- 3 etcd VMs
- 3 or more worker node VMs
- 2 or more load balancer VMs 

### Kubernetes Network Model
To address the problem of cross-host communication for containers, Kubernetes sets forth a network model that imposes the following rules (https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model):

- All containers can communicate with all other containers without NAT (network address translation)
- All nodes can communicate with all containers (and vice-versa) without NAT
- The IP that a container sees itself as is the same IP that others see it as

Kubernetes requires routeable IP addresses at the Pod level--not at the container level. This is known as the “IP-per-pod” model. All containers in a Pod share network namespaces and a single IP address. Containers in a Pod can reach each other’s ports on localhost. This removes the port mapping complexities that typically come with sharing a single host IP.

### Networking Considerations
When it comes to running Kubernetes and containers at scale, there are 5 areas of networking to consider:

- Container-to-container communications
- Pod-to-Service communications
- Pod-to-Pod communications
- External-to-internal communications
- Infrastructure communications

#### Container to Container
A pod is the container workload in Kubernetes. A pod runs one or more containers as a unit. All containers in a Pod behave as if they are on the same host with regard to networking. They can all reach each other's ports on localhost. 

All containers on the same pod share the same IP address. They communicate with each other via well-known port numbers (a priori). A pod is designed to run containers in a tightly coupled fashion. A containerized Wordpress app, for example, is deployed as a pod with three containers: ngnix web server, Wordpress application, MySQL database.

While the developer can control what containers belong to the same pod, he or she canonot control on what worker nodes the pod is deployed to. Kubernetes scheduling handles this.

#### Pod to Service
Kubernets provides an abstraction called a services that lets devlopers group pods under a common access policy (e.g. load-balanced). The implementation of this creates a virtual IP which clients can access and which is transparently proxied to the pods in a Service. Each worker node runs a kube-proxy process that programs iptables rules to trap access to service IPs and redirect them to the correct backends. This provides a highly-available load-balancing solution with low performance overhead by balancing client traffic from a node on that same node.

#### Pod to Pod
Because every pod gets a "real" (not machine-private) IP address, pods can communicate without proxies or translations. The pod can use HTTP IP-ADDRESS:PORT and can avoid the use of higher-level service discovery systems like DNS-SD, Consul, or Etcd. By making IP addresses and ports the same both inside and outside the pods, Kubernetes supports a NAT-less, flat address space. 

Kubernetes does not provide for this functionality. It is left to the infrastructure. Although Kubernetes provides network hooks for GCE, pod-to-pod communications and the management of pod IPs is left to the infrastructure. As such, specialized software has developed to address this need. Flanel is such a solution, as is NSX-T, discussed in more detail below.

#### External to Internal
The Kubernetes network model recogizes the obvious requirement to access a pod from outside the Kubernetes cluster network, for example, if a pod is running a web application. However, the solution is provided outside of Kubernetes. 

The way this is generally implemented is to deploy an externally-facing load balancer within the cluster to target Kubernetes Services. When traffic arrives at a node it is recognized as being part of a particular Service and routed to an appropriate backend Pod.

#### Infrastructure Networking and Security
The infrastrucutre on which Kubernetes is deployed requires networking. Kubernetes does not deploy itself. It needs to run on some infrastructure. The system resposible for deploying Kubenetes clusters must be able to communicate with the master nodes for management and worker nodes for upgrades.

### PKS with NSX-T
Because Kubernetes assumes that each container (Pod) has a unique, routable IP address inside the cluster, the Kubernetes “IP-per-pod” model and service-layer abstraction addresses the container-to-container problem (#1) and the container-to-service (#2) challenges. PKS with NSX-T addresses the other 3 challenges.

PKS with NSX-T provides the complete infrastructure and networking for deploying and running Kubernetes clusters securely at scale. 

PKS is composed of 3 VMs collectively known as the PKS Control Plane:
- PKS VM: PKS API (north Bound API to interact with PKS for Kubernetes cluster creation, deletion and resize), Proxy Server, Kubernetes master interface
- Ops Manager VM: Web Interface for deploying and managing BOSH and PKS Service VMs
- BOSH Director VM: Deploys and Manages Kubernetes Cluster

NSX-T is a virtualization solution that provides software for networking and security functions. For PKS on vSphere, NSX-T provides a comprehensive solution to satisfy the networking and security needs of the workloads running in Kubernetes.
- Pod-to-pod communications by assigning and managing Pod IP addressing using a variety of options, including routing and natting.
- Per-cluster highly available load balancer for external-to-internal communications.
- Various topology choices for running PKS infrastrucutre on or off the same NSX-T environment as the Kubernetes nodes. 

NSX-T addresses pod-to-pod communications, while also providing automatic cluster load balancing provisioning and options for different types of network topologies depending on your preference.

### NSX-T Toplogies
Using NSX-T provides various toplogies from which to choose to deploy PKS and Kubernetes. You can choose to deploy the PKS Control Plane VMs (Ops Manager, BOSH, PKS VM, Harbor) on the same NSX-T network as the Kubernetes nodes, or you can deploy PKS control plane and Kubernetes nodes on separate networks.

1. Deploy the PKS control plane and Kubernetes nodes on the same NSX-T network (No-NAT toplogy with Virtual Switch or NAT toplogy with Virtual Switch). 

With this toplogy, the PKS VMs (Ops Manager, BOSH, PKS Control Plane, Harbor) are deployed to a NSX-T Logical Switch (behind T0). Connectivity between PKS VMs, Kubernetes nodes, and the T0 Router is through a physical or virtual router. Traffic across the switch can be Routed or NAT'ed.

1a. Routed -- In a routed logical switch toplogy, PKS control plane components use corporate routable IP addresses. Likewise Kubernetes cluster master and worker nodes use corporate routable IP addresses. Both the PKS control plane components (VMs) and the Kubernetes Nodes use corporate routable IP addresses.

1b. Natted -- In a natted logical switch toplogy, PKS control plane components are located on a logical switch that has undergone Network Address Translation on T0. Likewise Kubernetes nodes are located on a logical switch that has undergone Network Address Translation on T0. PKS and Kubernetes nodes use private IP addresses which require DNAT rules to allow access to Kubernetes APIs.

2. The PKS control plane is deployed outside of the NSX-T network (No-NAT toplogy with Virtual Switch)

With this toplogy, the PKS VMs (Ops Manager, BOSH, PKS Control Plane, Harbor) are deployed to either a VSS or VDS backed portgroup. Connectivity between PKS VMs, K8S Cluster Management and T0 Router is through a physical or virtual router. NAT is only configured on T0 to provide POD networks access to associated Kubernetes Cluster Namespace.

From an operational and troubleshooting standpoint, the NAT toplogy (option 1b) is the most complex due to the number of SNAT/DNAT rules required to ensure proper communication between the vSphere Management componets (outside of NSX-T) and the PKS Control Plane and Kubernetes Cluster VMs (inside of NSX-T). 

In a NAT toplogy, the external DC firewall and the DB can distinguish tenants using the source SNAT IP that is allocated to a specific tenant. In a no-NAT toplogy, the external DC firewall and DB can distinguish tenants using the source IP subnet that is allocated to a specific tenant. The recommended approach is to implement a routable network on which the PKS VMs will reside on using either VSS/VDS or NSX-T Logical Switch (options 1a or 2 above). 

Note that from an environment perspective, the NAT topology may be the most expedient because with this toplogy you can use nested ESXi hosts and vApps to deploy PKS. These instructions implement the NAT topology.

This guide provides instructions for installing PKS in an NSX-T environment where the PKS control plane and Kubernetes cluster(s) all run in the same NSX-T virtual network using logical switches and NATing (1b above).

## Container Storage
Containers are ephemeral. When a container stops its data is deleted. To save the data across container instances you need a persistent storage volume.

Kubernetes provides persistent volume claims for connecting containers to persistent volumes. Kubernetes will bind a PV to PVC based on access
mode and storage capacity but a claim can also mention volume name, selectors and volume class for a better match. This design of PV-PVCs not only abstracts storage provisioning and consumption but also ensures security through access control.

### Static Volume Provisioning
Static Persistent Volumes require that a PKS operator manually create a (virtual disk) VMDK on a datastore, then create a Persistent Volume that abstracts the VMDK. A developer would then make use of the volume by specifying a Persistent Volume Claim.

### Dynamic Volume Provisioning
With PV and PVCs you can only provision storage statically because the PV needs to be created before a Pod can use (claim) it. Kubernetes provides the StorageClass API which enables dynamic volume provisioning. This avoids pre-provisioning of storage since storage is provisioned automatically when a user requests it. The VMDK's are also cleaned up when the Persistent Volume Claim is removed.

### PKS vSphere Support for Persistent Volumes

