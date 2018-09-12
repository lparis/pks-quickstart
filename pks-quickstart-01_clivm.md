# 1. Create PKS CLI VM

This section provides instructions for creating the PKS Client VM and installing various command-line tools for interacting with PKS Platform. These tools are used by platform operators responsible for managing the PKS infrastructure and developers who are consumers of the Kubernetes cluster services.

## Instructions

### 1. Create the pks-cli-vm.

a. Log in to the vSphere client for your [environment](environment).

Verify your environment meets the baseline. See [prerequisites](#prerequisites).

b. Download PKS software.

Log in to the Pivotal network <https://network.pivotal.io/products/pivotal-container-service/> and download the following software:

- pivotal-container-service-1.1.4-build.5.pivotal
- pks-linux-amd64-1.1.4-build.7
- kubectl-linux-amd64-v1.10.5

Create local folder `pks-software`.

Move the downloaded software to the `pks-software` folder.
bosh 
c. Create new Ubuntu VM named `pks-cli-vm`.

You have to install the OS. To do this, download an Ubuntu ISO, attach it as a CD to the VM and boot it.

Get the IP address for the `pks-cli-vm` from the NSX Manager. You can find allocated IPs at the Inventory -> IP Pools tab in the ip-pool-snat NAT pool. For example, the routable IP address to my CLI-VM: 10.40.14.6.

d. Log in to the Ubuntu cli-vm host.

- Ubuntu
- {standard-password}

Upgrade the software: `sudo apt-get upgrade`

e. Create directory /home/ubuntu/pks-software on the Ubuntu pks-cli-vm host.

`mkdir pks-software`

f. Use SCP to copy required software to the cli-vm.

If you are using Windows, install and use Putty. 

On the vSphere client host, run the following commands to copy the PKS software you downloaded to the pks-cli-vm:

cd c:\Program Files\PuTTY

pscp c:\pks-software\pks-linux-amd64-1.1.4-build.7 ubuntu@10.40.14.6:/home/ubuntu/pks-software

pscp c:\pks-software\kubectl-linux-amd64-v1.10.5 ubuntu@10.40.14.6:/home/ubuntu/pks-software

g. Make binaries executable and rename them: 

chmod +x pks-linux-amd64-1.1.4-build.7
chmod +x kubectl-linux-amd64-v1.10.5

sudo mv pks-linux-amd64-1.1.4-build.7 /usr/local/bin/pks
sudo mv kubectl-linux-amd64-v1.10.5 /usr/local/bin/kubectl

h. Verify CLI installation:

kubectl controls the Kubernetes cluster manager.

`pks --version`
`kubectl version`

i. View CLI help.

`pks --help`
`kubectl --help`

j. Install the UAA CLI.

The UAA CLI (UAAC) is a command line interface for the Cloud Foundry User Account and Authentication (UAA) identity server. UACC is used to manage user accounts and authorization for the PKS platform. For more information, see https://github.com/cloudfoundry/cf-uaac.

To insall UAAC on the CLI VM run these commands:

```
cd /usr/local/bin
sudo apt -y install ruby ruby-dev gcc build-essential g++
sudo gem install cf-uaac 
```

To verify:

```
uaac -v
```

Result:

UAA client 4.1.0

k. Install Operations Manager CLI.

The Ops Manager CLI is used to deploy products to Pivotal Ops Manager. 

Repository: https://github.com/pivotal-cf/om
Documentation: https://docs.pivotal.io/pivotalcf/2-1/customizing/ops-man.html

To install the Ops Manager CLI, run these commands:

```
cd /usr/local/bin
sudo wget https://github.com/pivotal-cf/om/releases/download/0.38.0/om-linux
sudo chmod +x om-linux
sudo mv om-linux /usr/local/bin/om
```

To verify installation:

```
om -v
```

Expected result:

`0.38.0`

l. Install Bosh CLI.

Bosh is used to manage PKS deployments and provides information about the VMs using its Cloud Provider Interface (CPI) which is vSphere in this case.

Documentation: https://bosh.io/docs/ and https://docs.pivotal.io/pivotalcf/2-2/customizing/deploy-bosh-om.html

Go here to find out Bosh CLI versions: https://s3.amazonaws.com/bosh-cli-artifacts/

To install the bosh cli, run the following commmands: 

```
sudo wget https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.48-linux-amd64
sudo chmod +x bosh-cli-2.0.48-linux-amd64
sudo mv bosh-cli-2.0.48-linux-amd64 /usr/local/bin/bosh
```

Verify:

```
bosh -v
```

m. Install NSX CLI.

The NSX CLI is used to clean NSX-T objects after a Kubernetes cluster has been deleted.

```
sudo apt -y install git httpie jq
sudo wget https://storage.googleapis.com/pks-releases/nsx-helper-pkg.tar.gz
sudo tar -xvzf nsx-helper-pkg.tar.gz
sudo chmod +x nsx-cli.sh
```

To verify:

nsx-cli.sh

Result:

```
ubuntu@ubuntu:/usr/local/bin$ nsx-cli.sh
bash /usr/local/bin/nsx-cli.sh [category] [method] [parameters...]
ipam
  allocate
  release [IP_ADDRESS]
nat
  create-rule [CLUSTER_UUID] [MASTER_IP] [FLOATING_IP]
  delete-rule [CLUSTER_UUID]
cleanup [CLUSTER_UUID] [DRY_RUN]
```

n. Install NSX-T cleanup script.

Run the following commands to download the script, make it executable, and rename it:

```
sudo wget https://storage.googleapis.com/pks-releases/pks_cleanup_linux
sudo chmod +x pks_cleanup_linux
sudo mv pks_cleanup_linux /usr/local/bin/pks_cleanup
```

To verify installation, run this command:

```
pks_cleanup --help
```

o. Snapshot.

At this point you have created the pks-cli-vm and installed all PKS-related CLIs to this VM. Take a snapshot.



---
Unable to get to 10.40.14.34:8443