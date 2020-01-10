# Azure Kubernetes 
## Overview

Create a private Azure Kubernetes Service cluster

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
az aks create -n <private-cluster-name> -g aksdemo --load-balancer-sku standard --enable-private-cluster --enable-addons monitoring --kubernetes-version $version --generate-ssh-keys --location <eastus2>


az aks install-cli
az aks get-credentials --resource-group <your rg name> --name <your aks name>

kubectl create clusterrolebinding kubernetes-dashboard -n kube-system --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
az aks browse --resource-group <your rg name> --name <your aks name>
```

### Create a Private endpoint in the Bastion VNET and link vnet to private-dns 
```powershell
################Retrieve AKS Resource ID######################
aksresourceid=$(az aks show --name aksattcluswestus2 --resource-group aksdemo --query 'id' -o tsv)
################Retrieve the MC Resource Group################
noderg=$(az aks show --name aksattcluswestus2 --resource-group aksdemo --query 'nodeResourceGroup' -o tsv) 
az resource list --resource-group $noderg
##############Create subnet, disable private endpoint network policies, create private endpoint############
az network vnet subnet create --name BastionPESubnet2 --resource-group Bastion --vnet-name BastionVMVNET --address-prefixes 10.0.4.0/24
az network vnet subnet update --name BastionPESubnet2 --resource-group Bastion --vnet-name BastionVMVNET --disable-private-endpoint-network-policies true
az network private-endpoint create --name PrivateKubeApiEndpoint2 --resource-group Bastion --vnet-name BastionVMVNET --subnet BastionPESubnet2 --private-connection-resource-id $aksresourceid --group-ids management --connection-name myKubeConnection
sudo az aks install-cli
az aks get-credentials --resource-group aksdemo --name aksattcluswestus2

az network private-dns zone create -g Bastion -n 1392a07c-ad38-49fd-b65e-86f45e099abb.westus2.azmk8s.io
az network private-dns record-set a add-record -g Bastion -z 1392a07c-ad38-49fd-b65e-86f45e099abb.westus2.azmk8s.io -n aksattclus-aksdemo-c24839-686c6cd4 -a 10.0.3.4
az network private-dns link vnet create -g Bastion -n MyDNSLinktoBastion -z 1392a07c-ad38-49fd-b65e-86f45e099abb.westus2.azmk8s.io -v BastionVMVNET -e true
```



##################Setup ACR################
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
############################Deploy application###############################################
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
   
