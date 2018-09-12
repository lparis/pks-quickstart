# Install and Configure OpsManger and Bosh Director

## 3. Install OpsManager

PKS leverages the capabilities of [Pivotal Cloud Foundry BOSH](https://bosh.io/docs/) to deploy the PKS infrastructure and clusters, including installing, upgrading, managing, provisioning, and monitoring the VMs comprising a PKS implementation, including the PKS control plane VMs and PKS cluster VMs (master and worker nodes). Pivotal Ops Manager provides the interface (UI/API/CLI) for PKS Operators to manage the complete lifecycle of both BOSH and PKS from install, patch and upgrade. In addition, you can also deploy new application services using Ops Manager Tiles such as the VMware Harbor container registry. In this case we are using BOSH to deploy PKS on vSphere and NSX-T, but BOSH also supports deploying PKS on other cloud platforms, including GCE and AWS.

- Install OPM VM
- Install BOSH VM
- Configure BOSH
	- Configure PKS AZs
	- Define networking for PKS mgmt VMs
	- Define networking for K8s mgmt cluster
	- Configure security, logging, and resources

a. Download Ops Manager for vSphere.

If you have not already done so, download Pivotal Cloud Foundry Ops Manager for vSphere relese 2.2.1 to your vSphere host.

The name of the file is `pcf-vsphere-2.2-build.296.ova`. Here are the URLs:

https://network.pivotal.io/products/ops-manager#/releases/140614
https://network.pivotal.io/api/v2/products/ops-manager/releases/140614/product_files/179467/download
http://sc-dbc1221.eng.vmware.com/nishadm/artifacts/pcf-vsphere-2.2-build.300.ova

The size of this file is 3.8GB so it will take a few minutes to download.

b. Deploy the Ops Manager VM to your Management Cluster. 

The Ops Manager for vSphere software is provided as an OVA file (`pcf-vsphere-2.2-build.296.ova`), which is a template for a VM. 

Use the vSphere Client to deploy the Ops Manager OVA (`pcf-vsphere-2.2-build.296.ova`) to your Management vSphere Cluster. Note that this can also be done using the using OVFTool or PowerCLI or the vSphere web client.  

Launch the vSphere Client UI.
Select the Management Cluster.
Right-click and select Deploy OVF Template.
Select the `pcf-vsphere-2.2-build.296.ova` tempalte.

1) Select an OVF template.

2) Select a name and folder.

3) Select a compute resource.

4) Review details.

Note the size on disk of the Ops Manager in the bottom rows of the table. In the next screen you will be asked to select storage. If you are using a vPod or vApp, or working with a non-mission critical deployment, you should select thin provisioning.

5) Select storage.

Select Thin Provisioning unless this is a production environment with enough storage for Ops Manager as displayed on the previous screen. Note that the option defaults to "Thick Provisioning." You must change this.

6) Select networks.

You must choose a standard/distributed portgroup when deploying the OVF, then change the network to the ls-pks-mgmt LS after deployment. (There is a bug in web client that wont add the VM to a LS.)

7) Customize template.

There a several OVF Properties that you will want to fill out such as the admin credentials, DNS Hostname and the desired network settings. 

You can use any IP you want for ops mgr. What I have done is to put it on 172.31.0.3. Subnet mask is 255.255.255.0. DNS is the windows console  192.168.110.10. There is a NAT rule in NSX that maps 172.31.0.3 to 10.40.14.3 because the 172 network is not accessible from the 192.168.x.x networks. Use 10.40.14.3 to access the Ops Mgr browser interface. 

- IP Address: 172.31.0.3
- Netmaks: 255.255.255.0
- Default Gateway: 172.31.0.1
- DNS: 192.168.110.10
- NTP Servers: 192.168.110.10
- Password for SSH access to VM: username = ubuntu / password = PASSWORD
- Custom Hostname: N/A (Don't enter anything here.)

When you have finished entering the information, deploy the VM.

Check for deployment success.

If you used the vSphere client, change the network to the ls-pks-mgmt LS after deployment. (There is a bug in web client that wont add the VM to a LS.) Make sure you select the correct network (Management).

To do this, select the Ops Manager VM.
Right click and select Edit Settings.
Select the Network Adapter and click Browse.
Select the ls-pks-mgmt network.

c. Start Ops Manager and connect.

Once the deployment has completed, Power On the Ops Manager VM.

Once Ops Manager is ready, connect to it using a web browser by DNS hostname and/or IP Address of the system. 

When using NSX-T, you will need to have a NAT rule in NSX that maps 172.31.0.3 to 10.40.14.3 because the 172 network is not accessible from the 192.168.x.x networks. Use 10.40.14.3 to access the Ops Mgr browser interface.

Bookmark the Ops Manager interface.

d. Select authentication type for Ops Mgr.

You are prompted to select the authentication type. 

For this scWe will use the Internal Authentication option. 

For integrating with Active Directory or other LDAP services, you can do so by connecting Ops Manager to VMware Identity Manager (vIDM) which is an external identity provider.

e. Create admin account and log into Ops Mgr.

Next, you will be prompted to create a new admin user which we will use to manage BOSH. 

opsmgr / VMware1!

VMware1!

e. Log into Ops Manager.

Once you have successfully created the user, go ahead and login with the new user account.

For some reason the UI says "Email" and "Password." Enter the User Name and Password you created.

f. Select the BOSH Director for VMware vSphere tile.

Once you are logged into Ops Manager, you will see see that the Ops Manager Tile named "BOSH Director for vSphere" is imported but is un-configured (denoted by orange coloring), which means BOSH itself has not yet been deployed. 

Click the tile to begin the configuration.

(Starts at Step 3 here: https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-4-ops-manager-bosh.html)

Once you are logged into Ops Manager, you will see see that the BOSH Tile has already been imported but is un-configured (denoted by orange coloring) which means BOSH itself has not yet been deployed. Go ahead and click on the tile to begin the configuration.

