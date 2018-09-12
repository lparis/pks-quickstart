# Deploy PKS Control Plane

## Summary
Deploy PKS Control Plane.
- Download PKS software bundle
- Insall the tile in BOSH
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

## Overview

In this section you deploy the PKS Control Plane which provides a frontend API that will be used by Operators to interact with PKS for provisioning and managing (create, delete, list, scale up/down) Kubernetes Clusters. Once a Kubernetes Cluster is deployed through PKS, operators simply provide their developers the external hostname of the cluster and the kubectl configuration file. These developers can immediately start deploying applications without knowing anything about PKS. If an application that a developer is deploying requires an external load balancer service, they can easily specify that in their application deployment YAML file and behind the scenes, PKS will automatically provision on-demand an NSX-T Load Balancer to service the application and this is completely seamless and does not require any additional assistance from the operator.

## Instructions

### Step 1. Import PKS Tile.

If you have not already downloaded PKS (`pivotal-container-service-*.pivotal`), please see Part 1 for the download URL.

To import the PKS Tile, go to the home page of Ops Manager and click "Import a Product" and select the PKS package to begin.

Once the PKS Tile is imported, click the "plus" symbol to add the PKS Tile which will make it available for configuring. 

Select the PKS Tile to begin the configuration.

### Step 2. Define AZ and networks.

This first section defines the AZ and Networks that will be used to deploy the PKS Control Plane VM as well as the K8S Management PODs. These were all previously defined when we had configured BOSH.

    Singleton Jobs: AZ-Management
    Balance Jobs: AZ-Management
    Network: pks-mgmt-network
    Service Network: k8s-mgmt-cluster-network


### Step 3. Configure PKS API endpoint.

This next section is for the PKS API endpoint and a certificate will be generated based on your DNS domain. In my environment, the domain is primp-industries.com and you will need to add wildcard in front as shown in the screenshot below.


### Step 4. Configure resource plans for Kubernetes clusters.

The Plan 1, Plan 2, and Plan 3 sections are used to configure the size and resources used for each of the VM types for kubernetes clusters. During cluster deployment, you can specify a plan and decide how big a given VM instance is for different deployment scenarios. 

For now, you can leave the defaults.

Assign the AZ for placement, which in our case is AZ-Compute which you will need to do for both Plans.


### Step 5. Configure vSphere as the Iaas.

In this section, you specify "vSphere" as the IaaS, provide your credentials to the vCenter Server, and enter the datastore in which persistent disks will be deployed to by PKS. Behind the scenes, when an application requests persistent disks (default is ephemeral), the Project Hatchway plugin intercepts the request and will use these credentials to create a persistent VMDK and make that available back to the application. 

For the "Stored VM Folder" field, be sure to use the same value that you had specified during the BOSH deployment.

### Step 6. Configure NSX-T networking.

In this next section you provide the NSX-T configurations for the networks created earlier and which will be used by kubernetes clusters. 

Start off by selecting "NSX-T" as the network type and then provide credentials to your NSX-T Manager. 

If you have replaced the NSX-T SSL Certificate, you will need to provide that or you can disable SSL verification which should only be done for testing purposes. 

Next, you will need to provide the name of the Compute vSphere Cluster which has been prepped for NSX-T, in this environment it is PKS-Cluster.

For next three fields, you will need to the NSX-T UI (this can also be programmatically queried through NSX-T REST API ) to obtain the UUID for the T0 Router, IP Block and Load Balance IP Pool.

    T0 Router ID - Navigate to Routing->Routers, select T0-LR and click on the ID to retrieve the UUID as shown in the screenshot below
    IP Block ID - Navigate to DDI->IPAM, select PKS-IP-Block and click on the ID to retrieve the UUID as shown in the screenshot below
    Floating IP Pool ID - Navigate to Inventory->Groups->IP Pools, select Load-Balancer-Pool and click on the ID to retrieve the UUID as shown in the screenshot below


Note: Pre-check validation for correct NSX-T objects UUIDs will be done when you click save, so if you made a mistake, the UI will alert you.

### Step 7. Configure authentication.

In this section, you configure the User Account and Authorization endpoint which we will use to manage users for PKS. 

You need to provide a DNS entry and ensure that it is mapped to the same DNS domain that you had configured earlier as the certificate generated will need to match. 

In the example, uaa.pks-example.com and once the PKS VM has been deployed, you can update your DNS Server to make sure this hostname points back to the IP selected for PKS Control Plane VM or you can update your /etc/hosts file on the PKS Client VM for testing purposes.


### Step 8. Enable NSX-T validation.

In this section, enable NSX-T validation and we can leave the rest as defaults.

### Step 9. Import updated stemcell for PKS.

In this section, if prompted import an updated Stemcell VM, otherwise move on to the next step.

### Step 10. Deploy PKS Control Plane.

To begin the PKS Control Plane VM deployment, navigate to the Ops Manager home page and click "Apply Changes" to start the deployment.

### Step 11. Verify PKS control plane deployment.

If everything is successfully deployed, in vCenter Server and you should see another new VM named vm-[UUID] to denote the PKS Control Plane VM. Similiar to the BOSH VM, we can look at the instance_group Custom Attribute to tell the role for this particular VM.

Another way you can easily identify either PKS Control Plane or BOSH VM is simply clicking on the tile in Ops Manager and then select the "Status" tab which will not only give you the VM Display Name in vCenter Server but also the IP Address that was automatically allocated from the PKS Management Network that we had specified within BOSH.

You can take the IP Address from below and create a DNS entry which we will be using in the next article to setup a new PKS user. If you do not have DNS in your environment, you can also add an entry to your /etc/hosts on the PKS Client VM as an alternative.
