# Deploy Kubernetes Clusters

## Summary: Deploy K8 cluster
- Configure BOSH for K8 deployment
	- Configure IAM policies
	- Get auth token
	- Create PKS user
	- Assign role
	- Grant OP access to cluster
	- Export BOSH envars
	- Connect to BOSH
- Login to PKS- Deploy k8 cluster
	- Create PKS cluster
	- Deploy PKS cluster
	- View cluster creation
	- Get client config file for K8 cluster (to give to devs)

## Overview

In this section you interact with PKS using the PKS CLI to request a new K8S Cluster as an Operator and then walk through a sample application deployment on top of the newly create K8S Cluster like a Developer would normally.

In this article, we will walk through the two workflows, one from the perspective of the Cloud/Platform Operator and how to create a new PKS Cluster to how it will be consumed by the Developer which is simply accessing the Kubernetes API endpoint and does not have to know anything about how it was provisioned or even access to the underlying PKS infrastructure. I think most of you have probably been waiting for this part of the series to see PKS in action and demonstrate how easy it is to manage and consume K8S Clusters.

## Instructions

### Step 1. Create UAA user.

Before a Cloud/Platform Operator can connect to PKS, we need to first create a new user within PKS's User Authentication and Authorization (UAA) system. It is possible to use the UAA system to connect and authorize users from an external directory service such as LDAP.  However, for purposes I will be using a locally created account from PKS. To do so, SSH to your PKS Client VM and configure the PKS UAA Endpoint by using the uaac CLI. If you recall in Part 5, Step 7, we defined the DNS name of our endpoint uaa.primp-industries.com which should point to our PKS Control Plane VM. Run the following command to specify the PKS UAA endpoint:

uaac target https://uaa.primp-industries.com:8443 --skip-ssl-validation


### Step 2. Retrieve UAA credentials.

Next, we need to retrieve the UAA Admin credentials so that we can request an authentication token to be authorized to create a new user. Login to the Ops Manager home page and select the PKS Tile and click on the Credentials tab. Open the "Uaa Admin Secret" link and make a note of the password which will be used in the next step.


### Step 3. Load admin token.

Run the following command and specify the credential you had retrieved from the previous step:

uaac token client get admin -s s4v38HVRFHXU0fTTFPATtT7cB5xvL-EV

### Step 4. Create PKS user.

If successful, we can now create a new PKS user by specifying the username, email and password by running the following command:

uaac user add lamw --emails lamw@primp-industries.com -p VMware1!

### Step 5. Assign role to PKS user.

PKS currently only supports two roles which can be assigned to a specific user:

    pks.clusters.admin - Can create PKS Clusters and can see all PKS Clusters created by other users
    pks.clusters.manage - Can create PKS Clusters and can only see PKS Clusters that they have created

Run the following command to assign the pks.clusters.admin role to the new user that you had just created:

uaac member add pks.clusters.admin lamw


### Step 6. Enable PKS provisioning visibility.

To provide Cloud/Platform Operators with visibility into the PKS provisioning process which can be useful for troubleshooting/debugging purposes, the BOSH CLI can be used. To gain access, you will need to authenticate against Ops Manager with the user you had logged into Ops Manager as well as referencing its certificate. Run the following command and replace the hostname or IP of your Ops Manager along with the username/password you used to login:

om --target https://pks-opsmgr.primp-industries.com -u lamw -p 'VMware1!' -k curl -p /api/v0/certificate_authorities -s | jq -r '.certificate_authorities | select(map(.active == true))[0] | .cert_pem' > /root/opsmanager.pem


If the command was successful, you should see the Ops Manager certificate stored under /root/opsmanager.pem in your PKS Client VM.

Next, run the following command to request a BOSH ticket provided with your Ops Manager credentials:

om --target https://pks-opsmgr.primp-industries.com -u lamw -p 'VMware1!' -k curl -p /api/v0/deployed/director/credentials/bosh2_commandline_credentials -s | jq -r '.credential'


If the request was successful, you should see the output like the above with list of environmental variables that will need to be set before you can use the BOSH CLI.

### Step 7. Export BOSH credentials.

Run the following commands to export the four BOSH environmental variables (replace with the ones you see in your own console output).

export BOSH_CLIENT=ops_manager
export BOSH_CLIENT_SECRET=B-KhvNkvco1oJYRThfgQEyP39BAjAbB2
export BOSH_CA_CERT=/root/opsmanager.pem
export BOSH_ENVIRONMENT=172.30.51.31

Note: For the BOSH_CA_CERT value, instead of using the one shown in your console output, make sure to update it so it points to /root/opsmanager.pem which is the Ops Manager certificate that we had downloaded from Step 6.

### Step 8. Connect to BOSH.

To verify that you can connect to BOSH, run the following command and you should see the PKS Control Plane VM as shown in the output below:

bosh vms

If you are unable to connect, you probably did not configure the correct BOSH environmental variables in Step 7 or retrieve the SSL Certificate for Ops Manager in Step 6.

### Step 10. Login to PKS endpoint.

We can now login to the PKS endpoint with the user that we had created in Step 4 by running the following command and specifying your PKS API endpoint along with the username and credentials:

pks login -a uaa.primp-industries.com -u lamw -p VMware1! -k

### Step 11. Deploy Kubernetes cluster.

At this point, we are now ready to deploy our first PKS Cluster!
PKS / K8S Cluster Deployment

In this section, we are now going walk through the workflow from the perspective of the Cloud/Platform Operator who will provision a new PKS Cluster. As part of the deployment, PKS will automatically setup a new Kubernetes (K8S) Cluster which can then be provided to their Developers for consumption.

### Step 12. Configure Kubernetes cluster.

You will need to provide a name to identify the PKS Cluster, an external hostname that will be mapped to a friendly DNS entry, a plan size and the number of K8S Worker Nodes. In our example, we will name our PKS Cluster k8s-cluster-01, the external hostname as pks-cluster-01 (which we will need to add to our DNS server once PKS completes the deployment and provides the allocated IP Address), the plan will be a small and number of worker nodes will be 3 by running the following command:

pks create-cluster k8s-cluster-01 --external-hostname pks-cluster-01 --plan small --num-nodes 3


Using the BOSH CLI, we can run the following command to watch the active tasks while our K8S Cluster is being deployed and configured:

bosh task

As you can see, for my deployment, it took 21 minutes for it to complete which included the automatic provisioning and association of NSX-T objects and deployment of a 3-Node Kubernetes Cluster. Pretty darn slick, if you ask me!? Once the deployment has completed, if can head over to our vCenter Server and you should see 4 new VMs that have provisioned which makes up our new K8S Cluster. You will have a single Master and three Worker Nodes (as defined by our deployment request). Similar to the other PKS Management VMs, you can easily identify the node type by looking at the instance_group or job Custom Attribute property for each VM.


### Step 13. View Kubernetes cluster.

To view all K8S Clusters that have been provisioned by PKS, you can run the following command:

pks clusters

We can see our k8s-cluster-01 and if we take a look at the Status field, it shows succeeded which means it is ready for our developers to consume and start deploying applications immediately.

To retrieve the K8S Master IP Address so that we can add a DNS entry for pks-cluster-01, run the following command and specify the name of your PKS Cluster:

pks cluster k8s-cluster-01


We can see that PKS has allocated an IP from our 10.10.0.0/24 range is used for the K8S Management Cluster Network. We can now add a DNS entry for pks-cluster-01 which will be the friendly name that our developers will use to connect to the K8S Cluster. In my example, pks-cluster-01.primp-industries should resolve to 10.10.0.2 and the reverse should also be true for proper forward and reverse DNS.

### Step 14. Retrieve credentials.

The last step before you hand over the K8S Cluster to your developers is by retrieving the credentials from PKS for the K8S cluster. The result is a K8S client configuration file that Developers are already familiar with and it contains information about the K8S Cluster as well as the certificate details to connect, no passwords are required which makes managing PKS Clusters extremely simple. Run the following command and specify the name of your PKS Cluster to retrieve the credentials:

pks get-credentials k8s-cluster-01


Once the command has completed, it will store and append the K8S Cluster credentials to `~/.kube/config` as shown in the screenshot above. You can then provide this configuration file to your Developers along with the hostname of the PKS Cluster (also encoded in the configuration file) which they can then use the K8S CLI (kubectl) to connect. If you accidentally deleted this file or the Developer needs to request it again, you simply re-run the above command and it will retrieve the details from PKS. There is no need to backup or persist this file as it can always be retrieved using either the PKS CLI or API.

As you can see, with just two simple PKS commands, a Cloud/Platform Operator can now quickly and easily deploy a number of PKS Clusters and make that available to their development teams to start deploying their applications.

## Application Deployment

In this section, we are now going walk through the workflow from the perspective of the Developer who will consume the new K8S Cluster and be able to start deploying his/her application using their standard tool of choice which is the K8S CLI (kubectl). Lets imagine you are a Developer and you just built the hottest Enterprise application called Yelb (actually this was a demo app built by our good friend Massimo Re Ferre' and Andrea Siviero) which contains the following four services: UI Frontend, Application Server, Database Server and Caching Service using Redis.

These have been packaged up as Docker Containers and you wan to be able to quickly deploy this hot new application to Production and not have to worry about individual container deployments and this is why you prefer K8S which is a robust container scheduler platform to do this for you. Luckily, our Cloud/Platform Operator was able to quickly provision us a new K8S Cluster and has also provided us with the K8S configuration file and endpoint so that we can deploy our application immediately.

Note: For demonstration purposes, we will continue using the same PKS Client VM and the same user account as it already has access to both the K8S client configuration file kubectl CLI. In practice, the users consuming the K8S Clusters will be your developers and he/she would be logging in from their own workstation and will NOT require any access to any of the PKS infrastructure (Ops Manager, BOSH, PKS, etc) as they simply just communicate with the deployed K8S Cluster via the K8S CLI/API.

## Step 1. Get nodes.

We can run the following command to verify that we can connect to our new K8S Cluster. We can use this command to see the number of K8S Worker Nodes that have been provisioned, which in our case is three and will be plenty for hosting our application.

kubectl get nodes -o wide


### Step 2. Create namespace.

Lets create a new K8S namespace to help separate our new project from others. The added benefit of using namespaces with PKS is that NSX-T can provide network micro-segmentation on a per-namespace level. This means Cloud/Platform Operators can isolate application/services between each other and enforce granular network and security policies provided by the integration with NSX-T. In this example, we will call our new namespace Yelb by running the following command:

kubectl create namespace yelb


### Step 3. Create deployment spec.

Now we need to create a K8S Deployment Spec describing how we want our application to be connected and deployed. I have already created a working version called yelb.yaml which you can download here and transfer it to your PKS Client VM. If you take a look at the YAML file, you can see the reference to our namespace that we had created in Step 2 along with the four Docker Containers that will be used. If you do not specify the namespace in the spec, when deploying the application, it will use the "default" namespace.

Note: Make sure that your K8S Management Cluster Network can reach the internet to be able to remotely download the required Docker Images.

### Step 4. Deploy app.

To deploy our application using our deployment spec, simply run the following command and specifying the YAML file you had downloaded from the previous step:

kubectl apply -f yelb.yaml

### Step 5. Monitor pod creation.

Once the deployment has started, we can run the following command to monitor the K8S Pod creation process denoted by the "STATUS" property. Once all four Pods have been successfully created and running, you can hit CTRL+C to exit out.

watch -n 1 kubectl get pods --namespace yelb

During the deployment, the required Docker Containers will need to be downloaded and this can take some time depending on size of the containers and the speed of your internet connection. For my setup, it took 6 minutes before all four containers were up and running.

### Step 6. Get frontend host.

Now that our application has been successfully deployed, we need to identify which of the K8S Worker Nodes is hosting our UI frontend so that we can access the application using our browser. To do so, we need to first get the unique Pod name for our yelb-ui which you can do so by running the following command:

kubectl get pods --namespace yelb

As you can see from the screenshot about, in our example the UI Pod name is yelb-ui-57ccb9d965-ntx7j (this will be different in your environment).

Next, we run the following command and specify the name of your UI Pod name to retrieve more details about this Pod:

kubectel describe pod yelb-ui-57ccb9d965-ntx7j --namespace yelb


In the output, look for the "Node" property value. In our example, it shows the IP Address of 10.10.0.5 and port defined in our K8S Deployment Spec is 30001.

### Step 7. Access the app.

To access our application, go ahead and open a web browser to the IP Address found in the previous step and using port 30001. If everything was setup and deployed correctly, we should see our new Yelb application startup as shown in the screenshot below. You can add a few votes to see the application in action as well as reloading the page to exercise other functionality that Masssimo and Andrea have built into this demo.


At this point, you have now successfully deployed your first application onto a PKS managed K8S Cluster!

However, it is not a general best practice to expose a particular K8S Worker Node for access which relies on a NAT, especially when we need to scale our application up or down based on demand. In K8S, you can specify as part of your application deployment spec, that you would like to use an External Load Balancer that would be used to sit in front of your application services, in this case our UI service. During deployment, PKS will see this request and automatically configure an NSX-T Load Balancer and add the virtual servers to the pool with appropriate firewall rules, etc. all done without having to interact with our Cloud/Platform Operator. Lets see this in action.

### Step 8. Download load balancer.

Download the load balancer K8S deployment spec called yelb-lb.yaml file here.

If we compare the two YAML files, we can see the only difference is adding the LoadBalancer type which will instruct K8S to request a Load Balancer.


### Step 9. Deploy load balancer.

Lets go ahead and deploy our new Yelb application which will include a Load Balancer by running the following command:

kubectl apply -f yelb-lb.yaml

### Step 10. Get load balancer IP.

Once the new application has been deployed. We can retrieve the Load Balancer IP Address by running the following command:

kubectl get services --namespace yelb

We can see that our yelb-ui frontend is now accessible from 10.20.0.16 which is an IP provisioned from our Load Balancer IP Pool which we had configured during the NSX-T configuration.

### Step 11. Access app via load balancer.

Lastly, lets go ahead and verify that we can now access our application by opening a browser to the load balancer IP Address, which we can see from the screenshot below. Obviously, for Production deployment, you would want to add a DNS entry for this IP Address so that your end users can reach the application with a friendly name rather than IP. Pretty awesome!

We spent some time in the shoes of our Developer and I think we can say, they are pretty happy with how easy it was to consume a PKS managed K8S Cluster. They did not have to change any of their existing workflows or tools and in fact, they even benefited from the additional network and security policies provided by the integration with NSX-T. At any given moment, there may be several dozen applications being deployed across a number of PKS managed K8S Clusters and from the Cloud/Platform Operators stand point, their job could not be any easier as they have provided a platform that is truly self service. As mentioned earlier, the creation of new on-demand K8S Pod Networks, Logical Switches, Load Balancer, Firewall Rules, etc. are all provisioned dynamically by NSX-T without any user interaction between the consumers and the platform operators.

Here are few screenshots from NSX-T showing the on-demand provisioning of our T1 Routers and Logical Switches which are broken out by individual K8S namespaces. We can also see the Load Balancer that was requested as part of the Developers application deployment spec and the respective K8S Nodes have been automatically added to the virtual sever pool as part of deployment workflow.

Another useful tool that Developers may want to use is the K8S Dashboard UI, which provides a graphical interface to a given K8S Cluster. If you wish to connect to this interface, please see the blog post below for the details.

    How to access the Kubernetes Dashboard UI for a VMware PKS Managed K8S Cluster?

## Decomissioning K8S Cluster

When Developers are done working with their K8S Cluster and wish to return the resources, it is simply one command to delete the K8S Cluster.

One thing I should note in this current release of PKS is that NSX-T objects are not automatically cleaned up after a PKS Cluster deletion. This is something that will be addressed in the next release and no additional user interaction will be required. For now, the Cloud/Platform Operator would simply use the nsx-cli.sh script which we had setup during Part 2 to clean up the NSX-T objects as part of K8S Cluster delete workflow. You could easily batch or automate this portion to simplify the process in the short term.

### Step 1. Get cluster UUID.

Once you have identified a PKS Cluster to be deleted, make a note of the UUID by running the following command:

pks cluster[NAME-OF-PKS-CLUTER]

If you do not remember or simply want to list all PKS Clusters, then run the following command:

pks clusters

### Step 2. PKS delete.

Next, we will use the PKS delete-cluster operation to de-provision the cluster and the VMs will be deleted via BOSH.

pks delete-cluster [NAME-OF-PKS-CLUTER]

### Step 3. Set NSX-T envars.

(Not required in future PKS release) - You will need to set the following environmental variables that contains credentials to your NSX-T installation. Do to so, run the following commands and replace it with the information of your own deployment:

export NSX_MANAGER_IP="nsx-mgr.primp-industries.com"
export NSX_MANAGER_USERNAME="admin"
export NSX_MANAGER_PASSWORD="VMware1!"

### Step 4. Clean up NSX-T objects.

(Not required in future PKS release) - Once the PKS Cluster has been successfully deleted, you can then run the following command and provide the UUID that you had retrieved from Step 1 to clean up all NSX-T objects related to this PKS Cluster:

./nsx-cli.sh cleanup [PKS-UUID} false


If you some how deleted a PKS Cluster but forgot to make a note of the PKS UUID, you can also find the UUID by looking at any of the NSX-T objects like T1 Routers, Logical Switch, Firewall Rules, etc. They will embed the UUID as part of the label and NSX-T will not delete an object if it is in use, so you can rest assure that you will not be deleting the wrong objects. This is just a temporarily workaround as this will be fixed in the next release of PKS and the de-provisioning of NSX-T objects will automatically be handled by PKS when you delete a PKS Cluster.


