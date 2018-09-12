# Part 2: Configure NSX-T for PKS

## Overview

In Part 2 you configure NSX-T 2.1 in preparation for installing PKS 1.1.4. 

NSX-T provides virtual networking and security services for PKS. When integrated with NSX-T, PKS gives you a single CLI command to deliver on demand provisioning of all network components required for cluster operations (including Container Network Interface (CNI), NSX-T Container Plugin (NCP) POD, NSX Node Agent POD) automatically when a new Kubernetes (K8S) Cluster is requested. In addition, PKS also provides a unique capability through its integration with NSX-T to enable network micro-segmentation at the Kubernetes namespace level, which allows cloud platform perators to manage access between application and/or tenant users at granular levels.

For more information about NSX-T, see the documentation https://docs.vmware.com/en/VMware-NSX-T/index.html. 

## Prerequisite

It is assumed that NSX-T version 2.1 is installed in your vSphere environment. You will need a basic NSX-T environment deployed which includes at lest two ESXi hosts prepped as Transport Nodes and at least 1 NSX-T Controller and 1 NSX-T Edge. 

When NSX-T is installed and the ESX hosts are registered with NSX as "hosts," a management channel between them is established. To have these ESXi hosts participate in virtual networking, they need to be converted to Transport Nodes.

Creating a Transport Node requires an uplink profile and an IP Pool:
- The Uplink profile defines the number of uplinks for the host and their teaming policy
- The IP Pool is used to assign an IP address to the Tunnel End Point(s) leveraged by the host

The Transport Node also needs to be attached to a Transport Zone. The Transport Zone defines the span of the virtual network over the physical infrastructure. Transport zone is also called an overlay network.

Finally, some logical switches will be instantiated, providing L2 connectivity between VMs on the different hypervisor Transport Nodes.
The last part of this exercise will show that there is connectivity between VMs over some Logical Switches spanning Hypervisors of different types (ESX and KVM).

When hypervisor hosts are prepared to operate with NSX-T, they are known as fabric nodes. Hosts that are fabric nodes have NSX-T modules installed and are registered with the NSX-T management plane. 

Refer to the NSX-T installation documentation for details: https://docs.vmware.com/en/VMware-NSX-T/2.1/com.vmware.nsxt.install.doc/GUID-3E0C4CEC-D593-4395-84C4-150CD6285963.html. 

For additioanal help, see: https://www.virtuallyghetto.com/2017/10/vghetto-automated-nsx-t-2-0-lab-deployment.html.

## Instructions for Configuing NSX-T for PKS

a. Verify baseline NSX-T configuration.

Log in to the NSX Manager web client UI.

Navigate to **Fabric > Nodes > Hosts**.

Select the **Managed by** network from the dropdown.

Verify that your environment meets the prerequisites. 

Insert 2.PKS-NSXT-0.png

b. Create IP pool.

The IP Pool is used to allocate Virtual IPs (VIPs) for exposed Kubernetes Services, such as the Load Balancer for application deployments. 

To do this, navigate to **Inventory > Groups > IP Pools** and provide the following:

Name: Load-Balancer-Pool
IP Range: 10.20.0.10 - 10.20.0.50
CIDR: 10.20.0.0/24

c. Create Nodes IP Block.

The **Nodes IP Block** is used by PKS to allocate /24 networks for each Kubernets cluster and associated namespaces. This IP block should be sized sufficiently to ensure you do not run out of addresses. The recommendation is to use a /16 network (non-routable). 

To do this, navigate to **DDI > IPAM** and provide the following:

Name: ip-block-pks-nodes-snat
CIDR: 172.15.0.0/16

d. Create Pods IP Block.

The **Pods IP Block** is used by PKS to allocate Pod IP addresses. Each Kubernetes cluster will use the entire /24 block. 

To do this, navigate to **DDI > IPAM** and provide the following:

Name: ip-block-pks-pods-snat
CIDR: 172.16.0.0/16

e. Create T0-Router.

The **T0-Router** is used by PKS to communicate with the external physical network.

To create a **T0 Router**, as part of the NSX-T prerequisites you must have defined an Edge Cluster with at least a single Edge node (two is recommended). If you do not have an Edge Cluster, you can create one when you define the **T0 Router**. 

To do this, navigate to **Routing > Routers**, click **ADD > Tier-0 Router** and provide the following:

Name: t0-pks
Edge Cluster: Edge-Cluster-01
High Availability Mode: Active-Standby
Preferred Member: edge-01

Note that **HA mode** must be set to Active/Standby as NAT is used by the NCP service within the Kubernetes management pod. 

f. Define static route to the T0-Router.

A static route on the T0-Router enables all traffic from the Kubernetes Management PODs to communicate outbound to Management components. This is needed for the NCP POD to talk to NSX-T for creation of new networks and/or Load Balancer services based on application deployment from the Kubernetes Clusters.

To do this, select the T0-Router you just created and navigate to **Routing > Static Routes** and provide the following:

Network: 0.0.0.0/0
Next Hop: 192.168.200.1

g. Create two logical switches.

One logical switch is used for the T0 uplink and the other logical switch is used for the Kubernetes Management Cluster (also known as the service network). 

To do this, navigate to **Switching > Switches** and add the following:

**Logical Switch 1**

Name: ls-pks-uplink
Transport Zone: tz-vlan
VLAN: 0

**Logical Switch 2**

Name: ls-pks-mgmt
Transport Zone: tz-overlay
VLAN: 0

h. Congfigure Uplink Router Port for the T0-Router.

Configure an **Uplink Router Port** and assign it an address from the intermediate network to route traffic from the T0-Router to the physical or virtual router. 

To do this, navigate to **Routing**, select the **T0-Router** you created earlier, select **Configuration > Router Ports** and provide the following:

Name: ls-pks-uplink
Type: Uplink
Transport Node: nsx-dege
Logical Switch: ls-pks-uplink
Logical Switch Port: UUID
IP Address/mask: 192.168.200.3/24

i. Create T1-Router.

Create T1-Router for use by the Kubernetes Management Cluster POD. 

To do this, navigate to **Routing > Routers** and provide the following:

Name: t1-pks-mgmt
Tier-0 Router: T0-LR
Failover Mode: Preemptive
Edge Cluster: Edge-Cluster-01
Edge Cluster Members: edge-01
Preferred Member: edge-01

j. Configure Downlink Router Port.

Configure the **Downlink Router Port** for the Kubernetes Management Cluster. This is where you define the network that NSX-T will use for Kubernetes VMs.

To do this, select the T1-Router that you created, select **Configuration > Router Ports** and provide the following:

Name: ls-pks-mgmt
Logical Switch: ls-pks-mgmt
Logical Switch Port: UUID
IP Address/mask: 172.31.0.1/24

k. Advertise T1 routes

To ensure the Kubernetes Management Cluster network is accessible from the outside, advertise the routes. 

To do this, select the T1-Router and navigate to **Routing > Route Advertisement** and enable the following:

Status: Enabled
Advertise All NSX Connected Routes: Yes
Advertise All NAT Routes: Yes

l. Define static routes.

If you are using a virtual router to connect your physical and virtual networks to the NSX-T T0-Router, you will need to define a few static routes to enable connectivity from the PKS management network as well as the networks that vCenter Server, ESXi hosts, NSX-T VMs and PKS Management VMs are hosted on to communicate with PKS is setting up a few static routes. We need to create two static routes to reach both our K8S Management Cluster Network (10.10.0.0/24) as well as K8S Load Balancer Network (10.20.0.0/24). For all traffic destined to either of these 10 networks, we will want them to be forwarded to our T0's uplink address which if you recall from Step 7 is 172.30.50.2. Depending on the physical or virtual router solution, you will need to follow your product documentation to setup either BGP or static routes.

m. Perform validation checks.

What is the best way to do this?


