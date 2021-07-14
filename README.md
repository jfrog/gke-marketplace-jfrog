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
export MACHINE_TYPE=e2-standard-4

# create the cluster using google command line tools
gcloud container clusters create "$CLUSTER" --zone "$ZONE" --machine-type "$MACHINE_TYPE"
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
```

#### Pull deployer image
Configure `gcloud` as a Docker credential helper:

```shell
gcloud auth configure-docker
```

Pull the deployer image to your local docker registry
```shell
docker pull gcr.io/jfrog-gc-mp/jfrog-artifactory/deployer:<RT_VERSION>
```

#### Run installer script
Set your application instance name and the Kubernetes namespace to deploy:

```shell
# set the application instance name
export APP_INSTANCE_NAME=artifactory-ha

# set the Kubernetes namespace the application was originally installed
export NAMESPACE=<namespace>

```

Create the namepsace
```shell
kubectl create namespace $NAMESPACE
```

Before running the deployer, external database needs to be deployed. It will be used by Artifactory and Xray applications.
More information on coniguring databases you can find [here](https://www.jfrog.com/confluence/display/JFROG/Configuring+the+Database)
Postgres database can be deployed and configured using steps below:
```shell

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install postgres bitnami/postgresql -n $NAMESPACE --set persistence.size=500Gi

```
The steps above will create the Postgresql Pod but now we need to create and configure the database for Artifactory as well as Xray. Please follow the steps below to configure databases:

The output of command above (helm install postgres bitnami/postgresql -n $NAMESPAC --set persistence.size=500Gi,service.type=LoadBalancer) will provide steps for configuring database:
```
export POSTGRES_PASSWORD=$(kubectl get secret -–namespace $NAMESPACE postgres-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)
kubectl run postgres-postgresql-client --rm --tty -i --restart='Never' --namespace $NAMESPACE --image docker.io/bitnami/postgresql:11.9.0-debian-10-r73 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgres-postgresql -U postgres -d postgres -p 5432
```
The output of that command will take you to the Postgresql console. Use following commands for creating the Database user and also providing privileges for the database
```
CREATE USER artifactory WITH PASSWORD '<YOUR_PASSWORD>';
CREATE DATABASE artifactory WITH OWNER=artifactory ENCODING='UTF8';
GRANT ALL PRIVILEGES ON DATABASE artifactory TO artifactory;
```

Create and configure database for Xray as well:
```
CREATE DATABASE xraydb WITH OWNER=artifactory ENCODING='UTF8';
GRANT ALL PRIVILEGES ON DATABASE xraydb TO artifactory;
```
To exit Postgres console, type `\q`.

Get external IP address of Postgresql Load balancer:
```
kubectl get svc --namespace=$NAMESPACE 
```
Copy IP address from the EXTERNAL-IP column. It will be used in the DB connection string later.

Generate Master and Join keys:

```
  openssl rand -hex 32
```
Join key should be the same for Artifactory and Xray, otherwise Xray won't be able to connect to Artifactory.

Create TLS k8s secret, if needed: 

```
kubectl create secret tls unified-tls-ingress --namespace=$NAMESPACE --cert=certfile.crt --key=keyfile.key
```

Optional - add k8s license secret. You can also add a license/licenses after the installation using [Install HA cluster license API](https://www.jfrog.com/confluence/display/JFROG/Artifactory+REST+API#ArtifactoryRESTAPI-InstallHAClusterLicenses) or 
using [Artifactory UI](https://www.jfrog.com/confluence/display/JFROG/Managing+Licenses#ManagingLicenses-LicensesManagement)

```
kubectl create secret generic artifactory-license --namespace=$NAMESPACE --from-file=artifactory.cluster.license 
```
Create JSON document with installation parameters, replace <POSTGRES_CLUSTER_IP> with actual Postgresql Load balancer external IP or cluster IP address. 
Assign this document to environment variable $ARGS_JSON. 

Example of parameters to deploy **Artifactory** and **Xray** with external Postgresql:

```
export ARGS_JSON='{
  "name": "unified",
  "namespace": "<YOUR_NAMESPACE>",
  "xray.enabled": true,
  "artifactory-ha.nginx.tlsSecretName": "unified-tls-ingress",
  "artifactory-ha.nginx.service.ssloffload": true,
  "artifactory-ha.artifactory.node.replicaCount": 1,
  "artifactory-ha.artifactory.license.secret": "artifactory-license",
  "artifactory-ha.artifactory.license.dataKey": "artifactory.cluster.license",
  "artifactory-ha.database.type": "postgresql",
  "artifactory-ha.database.driver": "org.postgresql.Driver",
  "artifactory-ha.database.url": "jdbc:postgresql://<POSTGRES_CLUSTER_IP>:5432/artifactory",
  "artifactory-ha.database.user": "artifactory",
  "artifactory-ha.database.password": "<YOUR_PASSWORD>",
  "xray.database.user": "artifactory",
  "xray.database.password": "password",
  "xray.database.url": "postgres://<POSTGRES_CLUSTER_IP>:5432/xraydb?sslmode=disable",
  "xray.xray.jfrogUrl": "http://unified-nginx",
  "xray.xray.joinKey": "<ARTIFACTORY_JOIN_KEY>",
  "artifactory-ha.artifactory.joinKey": "<MASTER_KEY>"
}'
```

Example of parameters to install **Artifactory** only, with external Postgresql:
```
export ARGS_JSON='{
  "name": "unified",
  "namespace": "<YOUR_NAMESPACE>",
  "xray.enabled": false,
  "artifactory-ha.nginx.tlsSecretName": "unified-tls-ingress",
  "artifactory-ha.nginx.service.ssloffload": true,
  "artifactory-ha.artifactory.node.replicaCount": 1,
  "artifactory-ha.artifactory.license.secret": "artifactory-license",
  "artifactory-ha.artifactory.license.dataKey": "artifactory.cluster.license",
  "artifactory-ha.database.type": "postgresql",
  "artifactory-ha.database.driver": "org.postgresql.Driver",
  "artifactory-ha.database.url": "jdbc:postgresql://<POSTGRES_LB_IP>:5432/artifactory",
  "artifactory-ha.database.user": "artifactory",
  "artifactory-ha.database.password": "<YOUR_PASSWORD>",
  "xray.database.url": "postgres://<POSTGRES_CLUSTER_IP>:5432/xraydb?sslmode=disable",
  "artifactory-ha.artifactory.joinKey": "<ARTIFACTORY_JOIN_KEY>"
}'
```

Run the installation script: 

```shell
./scripts/mpdev install --deployer=gcr.io/jfrog-gc-mp/jfrog-artifactory/deployer:<RT_VERSION> --parameters=$ARGS_JSON	
```


Watch the deployment come up with

```shell
kubectl get pods -n $NAMESPACE --watch
```
Artifactory UI will be on the external load balancer IP, which can be found by running 
```shell
kubectl get svc -n $NAMESPACE
```

# Delete the Application

There are two approaches to deleting the Artifactory-ha (Unified)

* [Using the Google Cloud Platform Console](#using-platform-console)

* [Using the command line](#using-command-line)


## <a name="using-platform-console"></a>Using the Google Cloud Platform Console

1. In the GCP Console, open [Kubernetes Applications](https://console.cloud.google.com/kubernetes/application).

1. From the list of applications, click **unified**.

1. On the Application Details page, click **Delete**.

## <a name="using-command-line"></a>Using the command line

Set your application instance name and the Kubernetes namespace used to deploy:

```shell
# set the application instance name
export APP_INSTANCE_NAME=unified

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

Optionally, if you don't need the deployed application, or the GKE cluster,
delete the cluster using this command:

```shell

# replace with the cluster name that you used
export CLUSTER=artifactory-ha-cluster

# replace with the zone that you used
export ZONE=us-west1-a

# delete the cluster using gcloud command
gcloud container clusters delete "$CLUSTER" --zone "$ZONE"
```
