# capi-clusters
Cluster-API Kubernetes cluster helm charts.

## Overview

Deploy Kubernetes clusters in multiple clouds with Cluster-API. This process was performed on MacOS.

The first few steps are documented in the [Cluster-API quickstart](https://cluster-api.sigs.k8s.io/user/quick-start.html). Afterwards, the Kubernetes clusters defined as Helm charts in this repo will be deployed to AWS, GCP and Azure.

___Warning: After creating clusters for a demo or to experiment, remember to shut them all down. See the clean-up section at then end.___

## Requirements

### Local

* MacOS (modifications required for other OS's)
* kubectl
* Docker

### Cloud

* AWS credentials
* GCP credentials
* Azure credentials

### Install kind

Install cluster management tool Kind, locally. Kind is not for production use.

```
brew install kind
```

### Install clusterctl

Install clusterctl CLI tool for Cluster-API.

```
brew install clusterctl
```

Confirm clusterctl is installed and working.

```
clusterctl version
```

## Create temporary local bootstrap cluster

```
kind create cluster
```

Confirm local cluster created successfully.

```
kubectl cluster-info
```

## Initialize the management cluster

Transform the Kubernetes cluster into a management cluster by running [clusterctl init](https://cluster-api.sigs.k8s.io/user/quick-start.html#initialization-for-common-providers) for each provider.

* AWS
* GCP
* Azure

### Enable feature gates

```
export CLUSTER_TOPOLOGY=true
```

### Initialization for AWS

The [Cluster API Provider AWS](https://cluster-api-aws.sigs.k8s.io/introduction.html) (CAPA) is used to deploy Kubernetes clusters to AWS.

Download the appropriate binary for your architecture. Look for the Assets section.

* clusterawsadm-darwin-amd64
* clusterawsadm-darwin-arm64

https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases

Change to the directory where executable resides. Change file mode to executable. Substitute amd64 if running MacOS on Intel.

```
chmod +x clusterawsadm-darwin-arm64
```

Move the executable to /usr/local/bin.

```
sudo mv clusterawsadm-darwin-arm64 /usr/local/bin/clusterawsadm
```

Confirm CAPA CLI tool clusterawsadm is working.

```
clusterawsadm version
```

The clusterawsadm command line utility assists with identity and access management (IAM) for Cluster API Provider AWS.

```
export AWS_REGION=us-east-1 # This is used to help encode your environment variables
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
export AWS_SESSION_TOKEN=<session-token> # If you are using Multi-Factor Auth.

# The clusterawsadm utility takes the credentials that you set as environment
# variables and uses them to create a CloudFormation stack in your AWS account
# with the correct IAM resources.
clusterawsadm bootstrap iam create-cloudformation-stack

# Create the base64 encoded credentials using clusterawsadm.
# This command uses your environment variables and encodes
# them in a value to be stored in a Kubernetes Secret.
export AWS_B64ENCODED_CREDENTIALS=$(clusterawsadm bootstrap credentials encode-as-profile)

# Finally, initialize the management cluster
clusterctl init --infrastructure aws
```

### Initialization for GCP

Initialize the management cluster

```
# Create the base64 encoded credentials by catting your credentials json.
# This command uses your environment variables and encodes
# them in a value to be stored in a Kubernetes Secret.
export GCP_B64ENCODED_CREDENTIALS=$( cat /path/to/gcp-credentials.json | base64 | tr -d '\n' )

# Finally, initialize the management cluster
clusterctl init --infrastructure gcp
```

### Initialization for Azure

The Cluster-API Provider for Azure.

```
export AZURE_SUBSCRIPTION_ID="<SubscriptionId>"

# Create an Azure Service Principal and paste the output here
export AZURE_TENANT_ID="<Tenant>"
export AZURE_CLIENT_ID="<AppId>"
export AZURE_CLIENT_SECRET="<Password>"

# Base64 encode the variables
export AZURE_SUBSCRIPTION_ID_B64="$(echo -n "$AZURE_SUBSCRIPTION_ID" | base64 | tr -d '\n')"
export AZURE_TENANT_ID_B64="$(echo -n "$AZURE_TENANT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_ID_B64="$(echo -n "$AZURE_CLIENT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_SECRET_B64="$(echo -n "$AZURE_CLIENT_SECRET" | base64 | tr -d '\n')"

# Settings needed for AzureClusterIdentity used by the AzureCluster
export AZURE_CLUSTER_IDENTITY_SECRET_NAME="cluster-identity-secret"
export CLUSTER_IDENTITY_NAME="cluster-identity"
export AZURE_CLUSTER_IDENTITY_SECRET_NAMESPACE="default"

# Create a secret to include the password of the Service Principal identity created in Azure
# This secret will be referenced by the AzureClusterIdentity used by the AzureCluster
kubectl create secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}" --from-literal=clientSecret="${AZURE_CLIENT_SECRET}"

# Finally, initialize the management cluster
clusterctl init --infrastructure azure
```

## Configuration for workload clusters

Once the management cluster is ready, you can [create your first workload cluster](https://cluster-api.sigs.k8s.io/user/quick-start.html#create-your-first-workload-cluster). The first step is to configure the required configuration as envvars.

### Required configuration for AWS.

```
export AWS_REGION=us-east-1
export AWS_SSH_KEY_NAME=default
# Select instance types
export AWS_CONTROL_PLANE_MACHINE_TYPE=t3.large
export AWS_NODE_MACHINE_TYPE=t3.large
```

### Required configuration for GCP.

```
# Name of the GCP datacenter location. Change this value to your desired location
export GCP_REGION="<GCP_REGION>"
export GCP_PROJECT="<GCP_PROJECT>"
# Make sure to use same kubernetes version here as building the GCE image
export KUBERNETES_VERSION=1.20.9
export GCP_CONTROL_PLANE_MACHINE_TYPE=n1-standard-2
export GCP_NODE_MACHINE_TYPE=n1-standard-2
export GCP_NETWORK_NAME=<GCP_NETWORK_NAME or default>
export CLUSTER_NAME="<CLUSTER_NAME>"
```

### Required configuration for Azure.

WARNING: Make sure you choose a VM size which is available in the desired location for your subscription. To see available SKUs, use 'az vm list-skus -l <your_location> -r virtualMachines -o table'.

```
# Name of the Azure datacenter location. Change this value to your desired location.
export AZURE_LOCATION="centralus"

# Select VM types.
export AZURE_CONTROL_PLANE_MACHINE_TYPE="Standard_D2s_v3"
export AZURE_NODE_MACHINE_TYPE="Standard_D2s_v3"

# [Optional] Select resource group. The default value is ${CLUSTER_NAME}.
export AZURE_RESOURCE_GROUP="<ResourceGroupName>"
```

### Generating the cluster configuration

#### Cluster Configuration for AWS

```
clusterctl generate cluster capi-quickstart-aws --infrastructure=aws --kubernetes-version v1.23.3 --control-plane-machine-count=3 --worker-machine-count=3 > capi-quickstart-aws.yaml
```

#### Cluster Configuration for GCP

```
clusterctl generate cluster capi-quickstart-gcp --infrastructure=gcp --kubernetes-version v1.23.3 --control-plane-machine-count=3 --worker-machine-count=3 > capi-quickstart-gcp.yaml
```

#### Cluster Configuration for Azure

```
clusterctl generate cluster capi-quickstart-azure --infrastructure=azure --kubernetes-version v1.23.3 --control-plane-machine-count=3 --worker-machine-count=3 > capi-quickstart-azure.yaml
```

## Create workload clusters with CLI

#### Create Workload Cluster AWS

```
kubectl apply -f capi-quickstart-aws.yaml
```

#### Create Workload Cluster GCP

```
kubectl apply -f capi-quickstart-gcp.yaml
```

#### Create Workload Cluster Azure

```
kubectl apply -f capi-quickstart-azure.yaml
```

## Accessing the workload cluster

Access the clusters by pulling the kubeconfig files.


## Create workload clusters with ArgoCD and Helm charts

### Install ArgoCD in Management Cluster

Install ArgoCD deployment tool in the cluster.

### Deploy a CNI solution

Deploy a container network interface for the clusters.

## Clean Up

#### Clean Up AWS

```
kubectl delete cluster capi-quickstart-aws
```

#### Clean Up GCP

```
kubectl delete cluster capi-quickstart-gcp
```

#### Clean Up Azure

```
kubectl delete cluster capi-quickstart-azure
```

#### Delete management cluster.

```
kind delete cluster
```

## Sources

* https://cluster-api.sigs.k8s.io/user/quick-start.html
* [Cluster API Deep Dive - Katie Gamanji, American Express & Carlos Panato, Mattermost](https://www.youtube.com/watch?v=npFO5Fixqcc)