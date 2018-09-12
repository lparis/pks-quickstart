# Deploy BOSH

BOSH is used by operators to deploy PKS on vSphere and NSX-T as well as other cloud platforms, including GCE and AWS.

1. Log into Ops Manager.

Enter your User Name and Password you created.

2. Select the BOSH Director for VMware vSphere tile.

Once you are logged into Ops Manager, you will see see that the Ops Manager Tile named "BOSH Director for vSphere" is imported but is un-configured (denoted by orange coloring), which means BOSH itself has not yet been deployed. 

Click the tile to begin the configuration.

(Starts at Step 3 here: https://www.virtuallyghetto.com/2018/03/getting-started-with-vmware-pivotal-container-service-pks-part-4-ops-manager-bosh.html)

Once you are logged into Ops Manager, you will see see that the BOSH Tile has already been imported but is un-configured (denoted by orange coloring) which means BOSH itself has not yet been deployed. Go ahead and click on the tile to begin the configuration.

3. Configure vCenter.

This first section is for the vCenter Server information on on where the BOSH VM will be deployed to. It is important to note that which ever vCenter Server BOSH is deployed to, it must also have inventory access to the vSphere Cluster that you will use to deploy the K8S workload. This means if you have a separate vCenter Server which manages your Management and Compute Cluster, you will need to deploy BOSH onto the Compute vCenter Server. The next part is the vSphere Datacenter (must already exists) to use as well as the disk provision type.

For vSphere Datastore configuration, you have to specify one for Ephemeral disk placement as well as Persistent disk Datastore placement. These can either be the same or different, it just depends on your environment. To deploy BOSH as well as the PKS Control Plane VMs, it will go ahead and upload a Stemcell VM (basically, a VM Template that PKS uses) and it will clone from that image for both PKS Management VMs as well as base K8S VMs. If you have a separate Management and Compute Cluster which also has non-shared Datastores that you wish to use, you will need to specify them separated by a comma delimited list. This ensures that when deploying BOSH and PKS Control Plane VM, they will be stored on the Management Cluster's Datastore and for K8S VMs, they will be stored on the Compute Cluster's Datastore.

In my environment, I have the following:

    Management Cluster Datastore: himalaya-local-SATA-re4gp4T:storage
    Compute Cluster Datastore: vsanDatastore

For the networking, just leave the default to "Standard vCenter Networking", we will not touch this if you are deploying BOSH and PKS Control Plane VM using either VSS or VDS backed portgroup. The last setting towards the bottom that you would need to tweak are the names of the VM Folders to use to store PKS VMs, PKS Templates & PKS Disks. You can specify any name like you and these directories will automatically be created for you. You should make a note of what you specify for the PKS VMs folder (first option) as that will be needed when configuring the PKS Control Plane.


    vCenter Host: The hostname of the vCenter that manages ESXi/vSphere.
    vCenter Username: A vCenter username with create and delete privileges for virtual machines (VMs) and folders.
    vCenter Password: The password for the vCenter user specified above.
    Datacenter Name: The name of the datacenter as it appears in vCenter.
    Virtual Disk Type: The Virtual Disk Type to provision for all VMs. For guidance on selecting a virtual disk type, see Provisioning a Virtual Disk in vSphere.
    Ephemeral Datastore Names (comma delimited): The names of the datastores that store ephemeral VM disks deployed by Ops Manager.
    Persistent Datastore Names (comma delimited): The names of the datastores that store persistent VM disks deployed by Ops Manager.
    VM Folder: The vSphere datacenter folder (default: pcf_vms) where Ops Manager places VMs.
    Template Folder: The vSphere datacenter folder (default: pcf_templates) where Ops Manager places VMs.
    Disk path Folder: The vSphere datastore folder (default: pcf_disk) where Ops Manager creates attached disk images. You must not nest this folder.

Select Standard vCenter Networking.

Click Save.

h. Configure Bosh Directory.

Select Director Config.

Step 5 - The next section is for BOSH itself, here you only need to specify an NTP Server as well as check the following three boxes:

    Enable VM Resurrector Plugin
    Enable Post Deploy Scripts
    Recreate all VMs

i. Create Availability Zones.

PKS Availability Zones are defined at a vSphere Cluster level. These AZs will then be used by BOSH to determine where to deploy the PKS Management VMs as well as the K8S VMs. You will need to create two AZs, one for Management and one for Compute. Resource Pools are optional but for customers who have a collapsed Management and Computer Cluster, Resource Pools can be used to guarantee resources to the PKS Management VMs so that K8S VMs will not impact them during resource contention.

In my environment, I have the following configuration:

    Name: AZ-Management
    Cluster: Primp-Cluster

    Name: AZ-Compute
    Cluster: PKS-Cluster


Step 7 - Create networks.

In this section, we will define the networks that will be used to deploy PKS Management VMs (BOSH and PKS Control Plane) as well as the K8S Management Network which is used by the K8S Management POD. For the vSphere Network, you will simply use the name of either your VSS or VDS portgroup as shown in vCenter Server. The Reserved IP Ranges is an interesting field as it is not clear at first what is actually needed. This field is basically a blacklist of all IPs within the selected network that you do NOT want BOSH to use, which should also include the gateway address. It is recommended that you use a dedicated network for PKS Management Network so that you only have to blacklist a few IPs that may be in use such as Ops Manager. If not, this can be an operational overhead when new VMs are added to this network outside of PKS, you would need to ensure that the blacklisted IPs are updated or an IP conflict may occur. The rest of the options are pretty straight forward. Make sure you specify the Management AZ which is the vSphere Cluster that BOSH and PKS Control Plane VM will be deployed to.

Here is my configuration for PKS Management Network:

    Name: pks-mgmt-network
    vSphere Network Name: dv-vlan3251
    CIDR: 172.30.51.0/24
    Reserved IP Ranges: 172.30.51.1-172.30.51.30
    DNS: 172.30.0.100
    Gateway: 172.30.51.1
    Availability Zones: AZ-Management


Step 8 - In this section, we will define the network that will be used to deploy K8S Management Cluster VMs, which will be the Logical Switch that we had created in Part 3 during our NSX-T configuration. The settings should be pretty straight forward and just remember to check the "Service Network" box for this network definition.

Here is my configuration for K8S Management Cluster Network:

    Name: k8s-mgmt-cluster-network
    Service Network: Check box
    vSphere Network Name: K8S-Mgmt-Cluster-LS
    CIDR: 10.10.0.0/24
    Reserved IP Ranges: 10.10.0.1
    DNS: 172.30.0.100
    Gateway: 10.10.0.1
    Availability Zones: AZ-Compute


Step 9 - In this section, we are defining the AZ and networking placement settings for the PKS Management VM (BOSH and PKS Control Plane) which we had defined in earlier.

    Singleton Availability Zone: AZ-Management
    Network: pks-mgmt-network


Step 10 - For the last three sections: Security, Syslog and Resource Config you can leave the defaults for testing purposes.

Step 11 - At this point, we have completed our BOSH configurations and we are now ready to deploy BOSH. Click on the Pivotal "P" icon in the upper left hand corner of the UI to return to the main Ops Manager home page. On the right hand side, you will see a pending operation for BOSH. Go ahead and click on "Apply changes" to begin the deployment.


The installation can take some time, you can get more details by expanding the logs on the right hand side. For my deployment, it took ~18minutes to complete.


If everything was deployed and configured successfully, you should see a success message and asked to return to the main home page. You will also see a new powered on VM in your vCenter Server inventory that starts with vm-[UUID] which is the BOSH VM and IP will be the first usable IP after your blacklisted entries. Ops Manager uses vSphere Custom Attributes to add additional metadata fields to identify the various VMs it can deploy, you tell what type of VM this is by simply looking at the deployment, instance_group or job property. In this case, we can see its been noted as p-bosh.


In the next blog post, we finish up our PKS deployment by installing and configuring the PKS Control Plane VM.