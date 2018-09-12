# Integrate PKS with Wavefront for Cluster Metrics

Wavefront provides cluster metrics for your Kubernetes deployments, such as pod status, alerts, etc.

Wavefront monitoring is facilitated using the Wavefront proxy which runs as a container inside each pod. Heapster and something else are used to transform the data from Kubernetes API to a format wavefront can consume.

Wavefront integration is disabled by default. When you deploy PKS, you can enable it. When you do, you get several out-of-the-box preconfigured metrics and alerts. And you can define your own.

cluster metrics

pod crashes

alerts/pager duties

https://docs.wavefront.com/pks.html

Wavefront provides cloud-based metrics and analytics. 

<metricsName> <metricValue> [<timestamp>]


## Instructions

Install a Wavefront instance and obtain the URL.

Log in to PCF Ops Manager and click the PKS tile in Installation Dashboard.

Under the Settings tab, click Monitoring.

In the right pane, check Yes to enable Wavefront Integration and enter the account information:

Wavefront URL: http://YOUR_CLUSTER.wavefront.com/api
API Token: YOUR_API_TOKEN
Wavefront Alert Recipient: A list of Email addresses &/or Wavefront Target IDs
Click Save.

Navigate back to the Installation dashboard and click Apply Changes.

----

---
title: Pivotal Container Service Integration
tags: [integrations list]
permalink: pks.html
summary: Learn about the Wavefront Pivotal Container Service Integration.
---
## Pivotal Container Service Integration

Pivotal Container Service (PKS) enables operators to provision, operate, and manage enterprise-grade Kubernetes clusters on Pivotal Cloud Foundry (PCF). This integration uses [Heapster](https://github.com/kubernetes/heapster), a collector agent that runs natively in Kubernetes and [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics), a simple service that listens to the Kubernetes API server and generates metrics. The integration collects detailed metrics about the containers, namespaces, nodes, pods, deployments, services and the cluster itself and sends them to a Wavefront.

This integration explains how to configure PKS monitoring with Wavefront from the PKS tile present in PCF Ops Manager. After you've completed the integration setup, you can use Wavefront to monitor the PKS cluster.

In addition to setting up the metrics flow, this integration also installs a dashboard. Here's a preview of **Overview** and **Nodes** section of the dashboard.

{% include image.md src="images/db_overview.png" width="80" %}

## Verify Prerequisites 

PKS integration with Wavefront is supported in PKS versions 1.1 and above. Check the [PKS documentation](https://docs.vmware.com/en/VMware-Pivotal-Container-Service/index.html) for details.

## Get Access to Wavefront

To get started with Wavefront, go to https://www.wavefront.com/ and sign up, or get enabled as a user with your Wavefront instance.

## Configure Wavefront Monitoring for PKS

Use PCF Ops Manager to integrate Wavefront with PKS. You do this by enabling Wavefront monitoring in the PKS tile, providing the Wavefront API location and credentials, and creating errands for alerts. 

1. Log in to PCF Ops Manager.
2. Select the **Pivotal Container Service** tile in Installation Dashboard.
3. In the Settings tab, click **Monitoring**.
4. Select **Yes** to enable **Wavefront Integration**.
5. Enter your Wavefront account information:
   
   **Wavefront URL**				https://YOUR_CLUSTER.wavefront.com/api
   URL of your Wavefront Subscription, such as: https://vmware.wavefront.com/api
   
   **Wavefront Access Token**		`YOUR_API_TOKEN`
   API token for your Wavefront Subscription (get from Wavefront > User Profile > API Access).
   
   **Wavefront Alert Recipient**	`List of Email addresses, Wavefront Target IDs`
   Comma-separated list of e-mail addresses and/or Wavefront target IDs to which triggered alerts will be sent. 

6. Select the **Errands** tab in Ops Manaer.
7. Enable the **Create pre-defined Wavefront alerts** errand and the **Delete pre-defined Wavefront alerts** errand. 
8. Click **Save** to save the Wavefront configuration.
9. Navigate to the Installation dashboard and click **Apply Changes**. 
Wavefront monitoring will be active for any clusters created after you have saved the configuration settings and applied changes. 

{% include image.md src="images/pks-01-monitoring" width="80" %}
{% include image.md src="images/pks-02-errands" width="80" %}

## Predefined Alerts for PKS

Once configured Wavefront provides the following monitoring alerts for PKS.

**Name** | **Severity** | **resolveAfterMin**
---------|--------------|--------------------
Node Memory Usage high | WARN | 10
Node Memory Usage too high | SEVERE | 10
Node CPU Usage high | WARN | 5
Node Memory Usage too high | SEVERE | 5
Node Storage Usage high | WARN | 10
Node Storage Usage too high | SEVERE | 10
Too many Pods crashing | SEVERE | 5
Too many Containers not running | SEVERE | 5
Node unhealthy | SEVERE | 5

## JSON Examples

Node Memory Usage High

```
{
	"name": "node memory usage high",
	"target": "user@example.com",
	"condition": "(ts(pks.heapster.node.memory.usage)-ts(pks.heapster.node.memory.cache)) /ts(pks.heapster.node.memory.node_allocatable) > 0.7",
	"displayExpression": "(ts(pks.heapster.node.memory.usage) – ts(pks.heapster.node.memory.cache)) / ts(pks.heapster.node.memory.node_allocatable)",
	"minutes": 10,
	"resolveAfterMinutes": 10,
	"severity": "WARN"
}
```

Node CPU Usage Too High

```
{
	"name": "node CPU usage too high",
	"target": "user@example.com",
	"condition": "ts(heapster.node.cpu.node_utilization) > 0.9",
	"displayExpression": "ts(heapster.node.cpu.node_utilization)",
	"minutes": 5,
	"resolveAfterMinutes": 5,
	"severity": "SEVERE"  
}
```

## PKS Monitoring Dashboards

Once configured Wavefront provides several dashboards for monitoring PKS.

PKS dashboard home
{% include image.md src="images/pks-03-home" width="80" %}

Nodes
{% include image.md src="images/pks-04-nodes" width="80" %}

Namespaces
{% include image.md src="images/pks-05-namespaces" width="80" %}

Deployments
{% include image.md src="images/pks-06-deployments" width="80" %}

Pods
{% include image.md src="images/pks-07-pods" width="80" %}

Pod Containers
{% include image.md src="images/pks-08-pod-containers" width="80" %}

Services and Replication Sets
{% include image.md src="images/pks-09-services-reps" width="80" %}

## PKS-Wavefront Architecture

Wavefront operates by running a Wavefront proxy pod inside each PKS-created Kubernetes cluster.

{% include image.md src="images/pks-13-proxy" width="80" %}

There are four containers within the Wavefront proxy pod:

{% include image.md src="images/pks-14-arch" width="80" %}

## Troubleshooting PKS-Wavefront Integration

If PKS tile deployment fails at running wavefront-alert-creation errand with "401 Unauthorized," the wavefront-access-token is invalid.

If you receive a no such host/no route to host error, check network connectivity to the internet (wavefront-api-server).

To check API access:

```
curl -s 'https://vmware.wavefront.com/api/v2/source‘ \
-H 'Authorization: Bearer 1d23d456-XXXX-XXXX-XXXX-f123f12b1c21'| jq .
```

If there are no metrics in the Wavefront web UI, possible causes are:
- 401 unauthorized: check wavefront-access-token validity
- Network connectivity to wavefront-api-server 

Check the status of the wavefront-proxy pod:

```
kubectl get pods --all-namespaces
```

{% include image.md src="images/pks-10-tsa" width="80" %}


```
kubectl describe pod wavefront-proxy-pod-name -n kube-system
```

{% include image.md src="images/pks-11-tsb" width="80" %}

Check wavefront-proxy pod logs:

```
kubectl logs wavefront-proxy-pod-name -n kube-system -c wavefront-proxy
kubectl logs wavefront-proxy-pod-name -n kube-system -c heapster
kubectl logs wavefront-proxy-pod-name -n kube-system -c kube-state-metrics
```

{% include image.md src="images/pks-12-tsc" width="80" %}

----

In this blog post, we will take a look at using VMware Wavefront, a recent acquisition, that is a leading SaaS based metrics and monitoring solutions for Cloud Native Applications. Wavefront supports monitoring Kuberenetes (K8S) and many other applications, but what is really neat about Wavefront is that it not only does it give us deeper metrics about our K8S infrastructure (cluster, pod, namespaces and containers), but it can also be used to help developers instrument their application for metric collection and monitoring and provide a complete end-to-end view. The Wavefront team also recently published this blog post which has a nice video demo of their integration with VMware PKS.

Similiar to both vRLI and vROps, Wavefront integration is optional and can be configured as part of a post-deployment task as outlined above. I can imagine in the future, the workflow could be as simple as toggling checkbox to enable Wavefront configuration. Customers would only need to specify the Wavefront URL (whether that is directly to the Wavefront SaaS service or an onPrem Wavefront Proxy Appliance which can reach the service) and PKS would automatically deploy the respective Wavefront Pods and wire everything up automatically. 


Step 1 - Sign up for a free 30 day trial of Wavefront here and sign in after you receive your email invitation.


Step 2 - Once logged in, you will be taken to the "Getting Started" page where you will need to select and configure an integration type. Go ahead and click on the Kuberenetes icon which should be at the top of the screen.


Step 3 - On this page, you will be given instructions to setup a Wavefront proxy, proxy service and heapster which all runs as a Pod within the PKS managed K8S Cluster that you to monitor. Since the Wavefront proxy is running within the K8S Cluster, you will need to make sure the nodes can actually connect out to the internet and reach Wavefront. If you do not have direct internet access (which most customers do not in a Production environment), then you can setup a proxy host which does have access. For more details, please see the documentation here.

The nice thing about Wavefront K8S integration, is that it provides the YAML snippets as well as the commands needed to deploy the Pods for your setup. To be able to easily identify your K8S Cluster name within the Wavefront UI, update the clusterName (e.g. wavefront:wavefront-proxy.default.svc.cluster.local:2878?clusterName=k8s-cluster-01&includeLabels=true) when creating the Heapster Pod.


When deploying the Wavefront Heapster Pod, I found that the default K8S Cluster already had a default unused Heapster Pod instance running and you may need to run the following 3 commands to successfully get it deployed:

kubectl create -f heapster.yaml
kubectl delete -f heapster.yaml
kubectl create -f heapster.yaml

Step 4 - To verify that the Wavefront proxy has successfully connected to the Wavefront service, we can check the logs of the Wavefront Pod. First, we need to retrieve the ID by running the following command:

kubectl get pod

To view the logs, simply run the following command along with the Wavefront Pod Id:

kubectl logs [WAVEFRONT-POD-ID]

What you are looking for is the following entry in the logs which indicates a successful connection was made:

2018-04-20 22:27:07,947 INFO [agent:readOrCreateDaemonId] Proxy Id created: 3c1c3cd5-c4d4-4193-9a60-ae71468f8379

Step 5 - After we have confirmed a succesful connection, we can head back to the Wavefront UI to view our data. Click on the Dashboards tab and once metrics have been received, you should see a new link called "Kubernetes Metrics" which you can launch to view your data. This may take a few minutes for the link to show up and I had to refresh a few times before I saw it.


Here is a screenshot of the data from my K8S Cluster which you can see goes all the day down to the application that I had deployed, pretty cool!


