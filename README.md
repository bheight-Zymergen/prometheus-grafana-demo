# Prometheus

Prometheus is an open source monitoring and alerting toolkit made for monitoring applications in clusters.
Inspired by Google’s Borgmon, it was designed from day one for the Kubernetes model of assigning and managing
units of work. Prometheus also has beta support for using the Kubernetes API itself to discover cluster
services and monitoring targets.

Kubernetes features built in support for Prometheus metrics and labels as well, and Prometheus support and
integration continues to advance on both sides.

This write-up shows how to use Prometheus to monitor the essential workers running in your Kubernetes cluster,
more specifically it'll show how to set this up in a minikube environment for local testing.

# The Setup

Like the jobs it monitors, Prometheus runs as a pod in your Kubernetes cluster.
Only two things are needed to deploy Prometheus:

 - A Deployment for the Prometheus pod itself
 - A ConfigMap that tells Prometheus how to connect to your cluster.

The ConfigMap will contain the configuration that Prometheus will use. Which will
allow us to separate the configuration from the image itself.

NOTE: Going with the approach of a configuration in a ConfigMap for Prometheus is highly
controversial but for the sake of a demo it'll be fine.

First, verify you are running in the context/cluster that you expect to deploy to. In the case of this demo
we are using Minikube.

    $ minikube-status
    minikube: Running
    cluster: Running
    kubectl: Correctly Configured: pointing to minikube-vm at 192.168.64.48

Run kubectl cluster-info and you should see something like:

    kubernetes master is running at https://192.168.64.2:8443
    kubernetes-dashboard is running at https://192.168.64.2:8443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard

To select the minikube local cluster for kubectl - useful if you have multiple clusters like we do here at Zymergen - then run

    kubectl config use-context minikube


Create a monitoring Namespace for us to deploy Prometheus and Grafana into:

    $ kubectl apply -f monitoring-namespace.yaml

Now lets apply the Prometheus configuration (via ConfigMap)

    $ kubectl apply -f prometheus-config.yaml

# Deploying the Prometheus Pod

A single Prometheus pod will be used in this demostration. We will be using a Kubernetes Deployment resource.
The deployment file itself is prometheus-deployment.yaml and defines several different kubernetes resources:

  - ClusterRole
  - ServiceAccount
  - ClusterRoleBinding (which binds a ServiceAccount to a ClusterRole)
  - Deployment

Apply it:

    $ kubectl apply -f prometheus-deployment.yaml

Verify the deployment exists:

    $ kubectl get deployments --namespace=monitoring

# Setup the Prometheus Kubernetes Service

So now the prometheus pod exists in the monitoring namespace. Now lets expose the Prometheus UI using a Kubernetes Service resource.

The prometheus-service.yaml contains a few things to note:

  - The label selector searches for pods that have been labeled with "name: prometheus" (as seen in prometherus-deployment.yaml)
  - We are exposing port 9090 of the running pods.
  - We are using a "NodePort". This means Kubernetes will open a port on each node in our Cluster. You can query the API to get this port.

Okay, now lets create the Service resource.

    $ kubectl apply -f prometheus-service.yaml

And verify it has been created:

    $ kubectl get services -n monitoring prometheus -o yaml

NOTE: You'll likely notice that the nodePort in the output will show something like "nodePort:30827" -- That is the port on the node that maps to the prometheus port on 9090.

With minikube we can use the built-in helper to access this service:

    $ minikube service --namespace=monitoring prometheus

And it will open a browser accessing the service for you.

From the Prometheus console, you can explore the metrics is it collecting and do some basic graphing.
You can also view the configuration and the targets.

Click Status->Targets and you should see the Kubernetes cluster and nodes. You should also see that Prometheus discovered itself under kubernetes-pods

# The Grafana Setup

Very much in the same way that we deployed Prometheus we will also do to deploy Grafana. 

We will deploy a Deployment resource, and a Service resource into our environment:

    $ kubectl apply -f grafana-deployment.yaml

    $ kubectl apply -f grafana-service.yaml

And verify:

    $ kubectl get deployments -n monitoring

    $ kubectl get services -n monitoring


And access Grafana:

    $ minikube service --namespace=monitoring grafana

The default user/pass is admin/admin for grafana.

Now we should add Prometheus as a datasource:

  - Click on the icon in the upper left of grafana and go to "Data Sources".
  - Click "Add data source".
  - For name, just use "prometheus"
  - Select "Prometheus" as the type
  - For the URL, we will actual use Kubernetes DNS service discovery. So, just enter http://prometheus:9090. This means that grafana will lookup the prometheus service running in the same namespace as it on port 9090.

Create a New dashboard by clicking on the upper-left icon and selecting Dashboard->New. Click the green control and add a graph panel. Under metrics, select "prometheus" as the datasource. For the query, use sum(container_memory_usage_bytes) by (pod_name). Click save. This graphs the memory used per pod.


