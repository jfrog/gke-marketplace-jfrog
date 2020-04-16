# JFrog Artifactory HA for GKE Marketplace

JFrog Artifactory HA can be installed using either of the following approaches:

* [Using the Google Cloud Platform Console](#using-install-platform-console)

* [Using the command line](#using-install-command-line)

## <a name="using-install-platform-console"></a>Using the Google Cloud Platform Marketplace

Get up and running with a few clicks! Install Artifactory Enterprise to a
Google Kubernetes Engine cluster using Google Cloud Marketplace. Follow the
[on-screen instructions](https://console.cloud.google.com/marketplace/details/jfrog/jfrog-gke).

## <a name="using-install-command-line"></a>Using the command line

### Prerequisites

#### Set up command-line tools

You'll need the following tools in your development environment:

- [gcloud](https://cloud.google.com/sdk/gcloud/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- [docker](https://docs.docker.com/install/)


Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

You can install artifactory-ha in an existing GKE cluster or create a new GKE cluster. 

* If you want to **create** a new Google GKE cluster, follow the instructions from the section [Create a GKE cluster](#create-gke-cluster) onwards.

* If you have an **existing** GKE cluster, ensure that the cluster nodes have a minimum of 2 vCPU and running k8s version 1.9 and follow the instructions from section [Install the application resource definition](#install-application-resource-definition) onwards.

#### <a name="create-gke-cluster"></a>Create a GKE cluster

Artifactory-ha requires a minimum 3 node cluster with each node having a minimum of 4 vCPU and k8s version 1.9. Available machine types can be seen [here](https://cloud.google.com/compute/docs/machine-types).

Create a new cluster from the command line:

```shell
# set the name of the Kubernetes cluster
export CLUSTER=artifactory-ha-cluster

# set the zone to launch the cluster
export ZONE=us-west1-a

# set the machine type for the cluster
export MACHINE_TYPE=n1-standard-4

# create the cluster using google command line tools
gcloud container clusters create "$CLUSTER" --zone "$ZONE" ---machine-type "$MACHINE_TYPE"
```

Configure `kubectl` to connect to the new cluster:

```shell
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
```

#### <a name="install-application-resource-definition"></a>Install the application resource definition

An application resource is a collection of individual Kubernetes components,
such as services, stateful sets, deployments, and so on, that you can manage as a group.

To set up your cluster to understand application resources, run the following command:

```shell
kubectl apply -f "https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml"
```

You need to run this command once.

The application resource is defined by the Kubernetes
[SIG-apps](https://github.com/kubernetes/community/tree/master/sig-apps)
community. The source code can be found on
[github.com/kubernetes-sigs/application](https://github.com/kubernetes-sigs/application).


#### Prerequisites for using Role-Based Access Control
You must grant your user the ability to create roles in Kubernetes by running the following command. 

```shell
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
```

You need to run this command once.

### Install the Application

#### Clone this repo

```shell
git clone https://github.com/jfrog/gke-marketplace-jfrog.git
git checkout master
```

#### Pull deployer image
Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

Pull the deployer image to your local docker registry
```shell
docker pull gcr.io/jfrog-gc-mp/jfrog-artifactory/deployer:7.4
```

#### Run installer script
Set your application instance name and the Kubernetes namespace to deploy:

```shell
# set the application instance name
export APP_INSTANCE_NAME=artifactory-ha

# set the Kubernetes namespace the application was originally installed
export NAMESPACE=<namespace>

```

Creat the namepsace
```shell
kubectl create namespace $NAMESPACE
```

Run the install script

```shell
./scripts/mpdev scripts/install  --deployer=gcr.io/jfrog-gc-mp/jfrog-artifactory/deployer:7.4   --parameters='{"name": "'$NAME'", "namespace": "'$NAMESPACE'"}'
```

Watch the deployment come up with

```shell
kubectl get pods -n $NAMESPACE --watch
```

# Delete the Application

There are two approaches to deleting the Artifactory-ha

* [Using the Google Cloud Platform Console](#using-platform-console)

* [Using the command line](#using-command-line)


## <a name="using-platform-console"></a>Using the Google Cloud Platform Console

1. In the GCP Console, open [Kubernetes Applications](https://console.cloud.google.com/kubernetes/application).

1. From the list of applications, click **artifactory-ha**.

1. On the Application Details page, click **Delete**.

## <a name="using-command-line"></a>Using the command line

Set your application instance name and the Kubernetes namespace used to deploy:

```shell
# set the application instance name
export APP_INSTANCE_NAME=artifactory-ha

# set the Kubernetes namespace the application was originally installed
export NAMESPACE=<namespace>
```

### Delete the resources

Delete the resources using types and a label:

```shell
kubectl delete statefulset,secret,service,configmap,serviceaccount,role,rolebinding,application \
  --namespace $NAMESPACE \
  --selector app.kubernetes.io/name=$APP_INSTANCE_NAME
```

### Delete the persistent volumes of your installation

By design, removal of stateful sets (used by bookie pods) in Kubernetes does not remove
the persistent volume claims that are attached to their pods. This prevents your
installations from accidentally deleting stateful data.

To remove the persistent volume claims with their attached persistent disks, run
the following `kubectl` commands:

```shell
# remove all the persistent volumes or disks
for pv in $(kubectl get pvc --namespace $NAMESPACE \
  --selector app.kubernetes.io/name=$APP_INSTANCE_NAME \
  --output jsonpath='{.items[*].spec.volumeName}');
do
  kubectl delete pv/$pv --namespace $NAMESPACE
done

# remove all the persistent volume claims
kubectl delete persistentvolumeclaims \
  --namespace $NAMESPACE \
  --selector app.kubernetes.io/name=$APP_INSTANCE_NAME
```

### Delete the GKE cluster

Optionally, if you don't need the deployed application or the GKE cluster,
delete the cluster using this command:

```shell

# replace with the cluster name that you used
export CLUSTER=artifactory-ha-cluster

# replace with the zone that you used
export ZONE=us-west1-a

# delete the cluster using gcloud command
gcloud container clusters delete "$CLUSTER" --zone "$ZONE"
```
