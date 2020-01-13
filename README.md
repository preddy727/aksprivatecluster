# Azure Kubernetes 
## Overview

Create a private Azure Kubernetes Service cluster and access kubectl commands (Control Plane) through a private endpoint.  
Deploy ACR with a service endpoint. 
Access ingress controller through private endpoint. 

## Pre-requisites 
The Azure CLI version 2.0.77 or later, and the Azure CLI AKS Preview extension version 0.4.18

### Architecture Diagram
* Process flow ![alt text](https://github.com/preddy727/aksprivatecluster/blob/master/PLSAKS.png)

## Goals of the Lab
1. Create a private AKS cluster.   

# Install the aks-preview extension
```powershell
az extension add --name aks-preview

# Update the extension to make sure you have the latest version installed

az extension update --name aks-preview

az feature register --name AKSPrivateLinkPreview --namespace Microsoft.ContainerService
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AKSPrivateLinkPreview')].{Name:name,State:properties.state}"

az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Network
```


## Create a private AKS cluster
```powershell
az login
az account set --subscription <your subscription name>

#create resource group
az group create --name <your rg name> --location <eastus2>

#get kubernetes version 
version=$(az aks get-versions -l <eastus2> --query 'orchestrators[-1].orchestratorVersion' -o tsv)
################Create Private Cluster########################
az aks create -n <private-cluster-name> -g <resource-group-name> --load-balancer-sku standard --enable-private-cluster --enable-addons monitoring --kubernetes-version $version --generate-ssh-keys --location <eastus2>
```


### Create a Private endpoint in the Bastion VNET and link vnet to private-dns 
```powershell
################Retrieve AKS Resource ID######################
aksresourceid=$(az aks show --name <private-cluster-name> --resource-group <resource-group-name> --query 'id' -o tsv)
################Retrieve the MC Resource Group and associated private DNS zone################
noderg=$(az aks show --name <private-cluster-name>  --resource-group <resource-group-name> --query 'nodeResourceGroup' -o tsv) 

##Note the Private DNS zone name and cluster A record" 
az resource list --resource-group $noderg | grep "privateDnsZones"
"id": "/subscriptions/c2483929-bdde-40b3-992e-66dd68f52928/resourceGroups/MC_aksdemo_aksatteastus2clus_eastus2/providers/Microsoft.Network/privateDnsZones/77cb2ebb-a082-43e7-a18e-0337bf24dfce.eastus2.azmk8s.io/virtualNetworkLinks/aksatteast-aksdemo-c24839-1e53cbe1"

##############Create subnet, disable private endpoint network policies, create private endpoint############
az network vnet subnet create --name BastionPESubnet2 --resource-group Bastion --vnet-name BastionVMVNET --address-prefixes 10.0.4.0/24
az network vnet subnet update --name BastionPESubnet2 --resource-group Bastion --vnet-name BastionVMVNET --disable-private-endpoint-network-policies true
az network private-endpoint create --name PrivateKubeApiEndpoint2 --resource-group Bastion --vnet-name BastionVMVNET --subnet BastionPESubnet2 --private-connection-resource-id $aksresourceid --group-ids management --connection-name myKubeConnection
##Go to the portal and get the ip address of the private-endpoint#############

##Duplicate the Private DNS zone saved earlier from the MC resource group in the Baston resource group"
az network private-dns zone create -g Bastion -n 77cb2ebb-a082-43e7-a18e-0337bf24dfce.eastus2.azmk8s.io
az network private-dns record-set a add-record -g Bastion -z 77cb2ebb-a082-43e7-a18e-0337bf24dfce.eastus2.azmk8s.io -n aksatteast-aksdemo-c24839-1e53cbe1 -a 10.0.4.4
az network private-dns link vnet create -g Bastion -n MyDNSLinktoBastion -z 77cb2ebb-a082-43e7-a18e-0337bf24dfce.eastus2.azmk8s.io -v BastionVMVNET -e true
```

## Validate connectivity to cluster
```powershell 
Run the following from the Bastion VM that has access to the endpoint created in the Bastion Subnet. 

Connect to the cluster
To manage a Kubernetes cluster, you use kubectl, the Kubernetes command-line client. If you use Azure Cloud Shell, kubectl is already installed. To install kubectl locally, use the az aks install-cli command:
Azure CLI 
az aks install-cli
To configure kubectl to connect to your Kubernetes cluster, use the az aks get-credentials command. This command downloads credentials and configures the Kubernetes CLI to use them.
Azure CLI  
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
To verify the connection to your cluster, use the kubectl get command to return a list of the cluster nodes.
Azure CLI 
kubectl get nodes
The following example output shows the single node created in the previous steps. Make sure that the status of the node is Ready:
	NAME                       STATUS   ROLES   AGE     VERSION
	aks-nodepool1-31718369-0   Ready    agent   6m44s   v1.12.8
```

## Access the cluster dashboard
```powershell
kubectl create clusterrolebinding kubernetes-dashboard -n kube-system --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
az aks browse --resource-group <your rg name> --name <your aks name>
```




### Setup ACR
```powershell
git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
cd azure-voting-app-redis
sudo /usr/local/bin/docker-compose up -d
sudo docker images
sudo docker ps
curl http://localhost:8080
sudo /usr/local/bin/docker-composer down

az acr create --resource-group aksdemo --name attacr --sku Premium --location westus
sudo az acr login --name attacr
az acr list --resource-group aksdemo --query "[].{acrLoginServer:loginServer}" --output table
sudo docker tag azure-vote-front attacr.azurecr.io/azure-vote-front:v1
sudo docker push attacr.azurecr.io/azure-vote-front:v1
az acr repository list --name attacr --output table
```
### Deploy application
```powershell
az acr create --resource-group aksdemo --name attacr --sku Standard --location westus 
AKS_RESOURCE_GROUP="aksdemo"
ACR_RESOURCE_GROUP="aksdemo"
AKS_CLUSTER_NAME="aksattcluswestus2"
Create Service endpoint from Bastion to ACR 
ACR_NAME="attacr"
 # Get the id of the service principal configured for AKS
 CLIENT_ID=$(az aks show --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" --output tsv)

 # Get the ACR registry resource id
 ACR_ID=$(az acr show --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --query "id" --output tsv)

# Create role assignment
az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID
az aks update -n aksattcluswestus2 -g aksdemo --attach-acr attacr

#Deploy the application
vi azure-vote-all-in-one-redis.yaml
kubectl apply -f azure-vote-all-in-one-redis.yaml
kubectl get service azure-vote-front --watch
```
### Create an ingress controller to an internal virtual network in Azure Kubernetes Service (AKS)

An ingress controller is a piece of software that provides reverse proxy, configurable traffic routing, and TLS termination for Kubernetes services. Kubernetes ingress resources are used to configure the ingress rules and routes for individual Kubernetes services. Using an ingress controller and ingress rules, a single IP address can be used to route traffic to multiple services in a Kubernetes cluster.

1) Install applications with Helm in Azure Kubernetes Service (AKS)

https://docs.microsoft.com/en-us/azure/aks/kubernetes-helm

2) Create an ingress controller. 

Create a file named internal-ingress.yaml using the following example manifest file. This example assigns 10.240.0.42 to the loadBalancerIP resource. Provide your own internal IP address for use with the ingress controller. Make sure that this IP address is not already in use within your virtual network.

```yaml
controller:
  service:
    loadBalancerIP: 10.240.0.42
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
 ```

