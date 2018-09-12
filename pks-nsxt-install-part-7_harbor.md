# Integrate PKS with Harbor Registry

## Overview

This section provides instructions for integrating PKS with VMware Harbor. 

Harbor is an open-source enterprise-class container registry that is used store and manage access to the software your Developers are creating.

Harbor is an optional but highly useful add-on to PKS. Harbor can be run on-prem to securely store and provide access to container images used by their development teams. 

## Instructions

The process of deploying Harbor involves downloading the Harbor Tile from Pivotal Network, import it into Ops Manager, and configuring and deploying Harbor.

### Step 1. Download Harbor Tile to Ops Manager.

Download the Harbor Tile from PivNet and import the file (`harbor-container-registry*.pivotal`) using Ops Manager. 

Once Harbor is imported, add the Tile into the Installation Dashboard.

Click the Harbor Tile to begin the configuration.

### Step 2. Define the network.

In this first section, define the AZ and network used to deploy the Harbor VM. 

This will be AZ-Management and pks-mgmt-network.

### Step 3. Define the hostname.

In this next section, define the DNS hostname to use for the Harbor VM. 

This DNS entry must be resolvable by your PKS management infrastructure and you can create the DNS entry after Harbor has been deployed by BOSH so that you can identify the IP Address that was selected.

### Step 4. Create a certificate. 

In this section, create a certificate for Harbor. 

Specify `*.[DNS-DOMAIN-NAME]` and click on the generate button.

### Step 5. Enter the Harbor password.

In this section, specify the admin password for Harbor.

### Step 6. Configure authentication.

In this section, define which authentication source we would like to use to allow users to connect to Harbor from their Docker clients. 

Since we have already configure UAA when we had configured the PKS Control Plane VM, use that but you have a few other options to select from.

### Step 7. Specify store location.

In this section, we specify the location to store our container images which can either be on the local file system of Harbor or using Amazon S3.

### Step 8. Complete configuration. 

The remainder 5 settings: Clair, Notary, Resource Config and Stemcell can all be left with their defaults.

### Step 9. Deploy Harbor.

At this point we have completed the Harbor configurations. It is now time to apply the changes and begin the Harbor VM deployment. Return to the Ops Manager home page and then click on the "Apply Changes" button to start.

The deployment will take some time and you can expand the verbose output to get more details. 

### Step 10. Get Harbor IP address.

Once the deployment has completed successfully, you can click on the Status tab to locate the IP Address that was selected for the Harbor VM and update DNS to now point the hostname you had specified earlier.

### Step 11. Create trust.

Before we can integrate Harbor with PKS, we need to create a trust between the two systems. 

Navigate back to the Ops Manager home page and then click on your user name at the upper right hand corner and then select Settings.

### Step 12. Download CA cert.

Select the Advanced tab and then click on "Download Root CA Cert" to download the certificate to your desktop.

### Step 13. Configure security.

Navigate to the Ops Manager home page and click the BOSH Tile. Go to the "Security" section and now copy and paste the contents of the certificate you had downloaded in the previous step into the "Trusted Certificates" box, then click Save.

Since we made a change to the BOSH Tile, we need to click on the "Apply Changes" button to update our configuration. 

### Step 14 Start using Harbor.

Open up a web browser and specify the DNS hostname of your Harbor instance which should take you to the login page. Login using the "admin" username and the password that you had configured earlier.

### Step 15. Create libarary store.

Harbor comes with a default project called "library" which you can start using to store your Docker Containers or you can create a new project. In my example, I will use the default. Next, we need to authorize users to be able to publish and consume images stored in Harbor. In the Project configuration, go to the "Members" tab and then enter a user from the authentication source you had setup in Step 6. In my case, this is the user I had created in PKS's UAA system which is lamw and it should automatically pickup the user while entering the input. You then select a role, whether thats a Project Admin, Developer or Guest. Since we want to be able to push/pull content, I will specify the Developer role.

### Step 16. Enable trust for Docker clients.

For Docker Clients to be able to push content into Harbor, they also need to setup a trust using the same certificate that was downloaded earlier in Step 12. Copy the certificate, which we can simply name ca.crt to the PKS Client VM. Next, run the following commands to install Docker Client (if you do not already have that running) and create the following directory which should map to the DNS hostname of your Harbor instance:

```
apt install -y docker.io
mkdir -p /etc/docker/certs.d/pks-harbor.primp-industries.com
```

Next, move the ca.crt file into /etc/docker/certs.d/pks-harbor.primp-industries.com by running the following command:

```
mv ca.crt /etc/docker/certs.d/pks-harbor.primp-industries.com
```

For the Docker Client to pick up the certificate, go ahead restart the service by running the following command:

```
systemctl stop docker
systemctl start docker
```

### Step 17. Login to Harbor registry via Docker.

If everything was succesful, you should be able to login to Harbor by running the following command and specifying the username/password that you have enabled:

```
docker login pks-harbor.primp-industries.com
```

### Step 18. Push images to Harbor registry.

At this point, we are now ready to push images into our private container registry. Below are four Docker Containers which I have already pulled down and we want to make these available directly within Harbor. If you have not done this, you can do so by running the following commands:

```
docker pull mreferre/yelb-ui:0.3
docker pull mreferre/yelb-appserver:0.3
docker pull mreferre/yelb-db:0.3
docker pull redis:4.0.2
```

### Step 19. Tag the containers.

We now need to tag the containers so it contains the destination of our Harbor instance, the project name and finally the name of the container. As you can see from the commands below, we are specifying the DNS hostname of our Harbor instance and because we are using the default project called "library", we need to append that along with the name of the container and version.

```
docker tag mreferre/yelb-ui:0.3 pks-harbor.primp-industries.com/library/yelb-ui:0.3
docker tag mreferre/yelb-appserver:0.3 pks-harbor.primp-industries.com/library/yelb-appserver:0.3
docker tag mreferre/yelb-db:0.3 pks-harbor.primp-industries.com/library/yelb-db:0.3
docker tag redis:4.0.2 pks-harbor.primp-industries.com/library/redis:4.0.2
```

### Step 20. Push images.

Now we are ready to push these images into our private container registry by running the following command which simply references the tag names we had used in the previous step:

docker push pks-harbor.primp-industries.com/library/yelb-ui:0.3
docker push pks-harbor.primp-industries.com/library/yelb-appserver:0.3
docker push pks-harbor.primp-industries.com/library/yelb-db:0.3
docker push pks-harbor.primp-industries.com/library/redis:4.0.2


Once the container upload has finished, we can navigate back to our Harbor UI and we should now see our four Docker Containers now residing in Harbor.

### Step 21. Deploy app from Harbor instance.

We can now re-deploy our application but rather than using an image from Docker's registry which requires internet access, we can actually refer to our Harbor instance. I have created an updated YAML called yelb-lb-harbor.yaml which you can download here that now refers to our Harbor instance for the container images. Below is a quick diff between yelb-lb.yaml and yelb-lb-harbor.yaml and you can see the difference in the location of our Docker Containers. You obviously will want to update the names based on what you have deployed in your own environment.



