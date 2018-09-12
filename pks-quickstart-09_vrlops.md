# Monitor PKS Infrastructure using vROps

## Overview

This section provides instructions for configuring VMware vRealize Operations Manager (vROps) to monitor PKS infrastrcuture. 

With vROps operations staff can monitor and alert on the underlying PKS infrastructure: compute, storage, and networking. In addition, through the use of vROps Management Packs, you can get visibility into the individually deployed PKS managed Kubernetes (K8S) clusters that can be useful when debugging with development teams.

Integrating PKS with vROps is done after PKS is deployed. This process can be automated by consuming both PKS and vROps APIs.

### Step 1. Download and deploy vROps. 

For detailed instructions, see the vOps documentation. 

For lab and proof of concept purposes, you can select the "Extra Small" size when deploying the vROPs appliance.

### Step 2. Configure the vSphere Adapter. 

Once vROps is running, configure the vSphere Adapter.

To do so, select the Administration tab the top and then Solutions on the left hand side and find the adapter. 

Then click on the gear box to start the configure as shown in the screenshot below. Here you will need to provide the credentials to your vCenter Server, which you can test the connection before saving. If the connection is successful, you should start seeing data being retrieved by the adapter.

### Step 3 - Import the management pack.

Next, download the vROps Management Pack for Containers which will give us additional information about our deployed K8S Clusters.

To import the management pack, under the Solutions section click on the "+" icon and then select the `*.pak` file that you had downloaded from the previous step and follow the wizard for installation instructions.

### Step 4. Monitor PKS-managed clusters.

With the Container management pack installed, we now have the ability to monitor individual PKS managed K8S Clusters that have been deployed by PKS, which is pretty slick. To do so, we will need to retrieve the credentials using the PKS CLI (similar to what we would do to provide the K8S configuration file to our developers).

Use the following command to identify the PKS Clusters you wish to add to vROPs:

```
pks clusters
```

Next, run the following command and specify the name of the PKS Cluster to generate the K8S configuration file:

```
pks get-credentials [PKS-CLUSTER-NAME]
```

At this point, you will need to extract the following four pieces of information from `~/.kube/config` which will be needed in the next step as highlighted in the screenshot below:

```
    K8S Server endpoint
    PKS Cluster Name
    K8S Username
    K8S Token
```

Note: You will need to repeat this for each PKS Cluster you plan to monitor in vROps.

### Step 5. Add PKS clusters to vROps.

Go back to vROps Solutions screen and then select the container management pack and click on the gear box to add the PKS Clusters to vROPs.

Click on the "+" icon to add a PKS Cluster. Populate the following vROps fields: 

- Display Name
- Master URL 
- Credential (use type Token)

For example:

    Display Name: k8s-cluster-01
    Master URL: https://pks-cluster-01:8443
    Credential Username: 8ebcb...
    Credential Token: eyJ...

Before saving, you can verify the settings are correct by using the "Test Connection" button. Once successful, you can then save and proceed to add other PKS Clusters into vROps.

### Step 6. Enable container dashboard for Kubernetes.

To enable our new container dashboard, click on the Dashboards tab at the top and select "Kuberentes Environment" from the drop down menu of "All Dashboards". This will add our container dashboard to the left hand panel for access.

As you can see from the screenshot below, vROps has already connected to my deployed PKS managed K8S Cluster and started to received health and configuration metrics about my deployment. It not only provides information about the K8S infrastructure but also the K8S namespaces, pods, containers and services. There are several other portlets that contain other pieces of useful information such as utilization and performance.
