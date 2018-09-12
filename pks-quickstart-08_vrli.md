# Configure Logging for PKS Using vRLI

## Overview

This section provides instructions for configuring PKS components, including vSphere (vCenter Server & ESXi), NSX-T (Manager, Controllers & Edges), BOSH and PKS Control Plane, to forward their logs to VMware vRealize Log Insight (vRLI). vRLI includes 25 free OSI licenses for any vSphere customer.

## Requirements

Integrating PKS with vRLI for logging requires some additional steps post-PKS deployment. It is assumed you have successfully installed PKS and deployed one or more Kubernetes clusters as described in this documentation.

## Instructions

### Step 1. Download and deploy vRLI. 

For lab and proof of concept purposes, you can select the "Extra Small" size when deploying the vRLI appliance.

For more in-depth deployment options, see the vRLI documentation. 

### Step 2. Enable vSphere integration.

Once vRLI is up and running, enable vSphere Integration which enables logging from both vCenter Server as well as the ESXi hosts that it manages. This can be useful for troubleshooting and/or for auditing purposes logging all requests made to the underlying vSphere platform. 

If you only want to forward vCenter Server Events (VM Create, Delete, Host Add, etc), which is only possible when using the VCSA, see this blog post here to configure this using the VAMI UI interface (https://[VCSA]:5480)

### Step 3. Install NSX-T Content Pack.

Install the NSX-T Content Pack which provides several useful dashboards specific to an NSX-T deployment.

You can access the content pack page by going to the upper right hand corner and click on logged on username and then clicking on the "3-dashes" icon.

You can also navigate to the following URL: https://[VRLI/contentpack. Under the Marketplace, select the NSX-T plugin and click Install.

### Step 4. Forward logs to vRLI instance.

To send NSX-T logs to the vRLI instance, use the NSX CLI that is available when you SSH to each of the NSX systems: Manager, Controllers and Edge Nodes and run the following command (replace with the IP of your vRLI instance):

```
set logging-server 172.30.0.102 proto udp level info
```

You can verify the configuration by running `get logging-server`. To clear the configuration, run `clear logging-server`. A restart of services is not required for the changes to go into effect.

### Step 5. Configure BOSH log forwarding.

Login to the Ops Manager UI and click on the BOSH Tile. 

Select the Syslog tab and specify the address of your vRLI instance along with the desired port/protocol. 

For testing purposes, UDP is acceptable. For more reliable or secure logging, use TCP and/or TLS. 

### Step 6. Configure PKS logging.

Configure PKS logging by selecting the PKS Tile. 

Select the Syslog tab and specify the address of your vRLI instance along with the desired port/protocol. 

For testing purposes, UDP is acceptable. For more reliable or secure logging, use TCP and/or TLS. 

### Step 7. Save and apply changes.

Once you have saved your changes, navigate back to the Ops Manager home page and click on the Apply Changes button on the upper right hand side to deploy the updated configurations.

### Step 8. Verify log forwarding.

Once logging configuration is complete, when deploying new PKS Clusters, logs from each of the configured components will be centrally available within vRLI for further processing.

If you click on the Dashboard view, you can select the NSX-T Dashboards to see some of the default views that are available as part of that Content Pack. 

Other Dashboards include VSAN and vSphere.

If you want to query and see individual log entries, simply click on the Interactive Analytics tab at the top.



