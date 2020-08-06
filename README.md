# Azure Kubernetes 
## Overview

Create a private Azure Kubernetes Service cluster and access kubectl commands (Control Plane) through a private endpoint.  
Deploy ACR with a service endpoint. 
Access ingress controller through private endpoint. 

## Pre-requisites 
The Azure CLI version 2.2.0 or later or later

### Architecture Diagram
* Process flow ![alt text](https://github.com/preddy727/aksprivatecluster/blob/master/architecture%20(1).png)

## Goals of the Lab
1. Create a private AKS cluster.   

## Setup a private Azure Container registry and create a private AKS cluster
```powershell
git clone https://github.com/preddy727/aksprivatecluster.git
# Please provide your subscription id here
export APP_SUBSCRIPTION_ID=c2483929-bdde-40b3-992e-66dd68f52928
# Please provide your unique prefix to make sure that your resources are unique


# Please provide your subscription id here
export APP_SUBSCRIPTION_ID=c2483929-bdde-40b3-992e-66dd68f52928
# Please provide your unique prefix to make sure that your resources are unique
export APP_PREFIX=preastus2
# Please provide your region
export LOCATION=EastUS2
export REGISTRY_LOCATION=EastUS2

export VNET_PREFIX="192.168."

export AKS_PE_DEMO_RG=$APP_PREFIX"-aksdemo-rg"
export AKS_PRIVATE_CLUSTER=$APP_PREFIX"-aksdemo-aks"
export ADO_PE_DEMO_RG=$APP_PREFIX"-adodemo-rg"
export DEMO_VNET=$APP_PREFIX"-aksdemo-vnet"
export DEMO_VNET_CIDR=$VNET_PREFIX"0.0/16"
export DEMO_VNET_APP_SUBNET=app_subnet
export DEMO_VNET_APP_SUBNET_CIDR=$VNET_PREFIX"1.0/24"
export AKS_PE_SUBNET=aks_pe_subnet
export AKS_PE_SUBNET_CIDR="10.0.1.0/24"

# set this to the name of your Azure Container Registry.  It must be globally unique
export MYACR=$APP_PREFIX"myContainerRegistry"
export VM_NAME=myDockerVM


az login
az account set --subscription $APP_SUBSCRIPTION_ID

#create resource group
az group create --name $AKS_PE_DEMO_RG --location $LOCATION
az group create --name $ADO_PE_DEMO_RG --location $LOCATION



#Create a vnet and subnet 
az network vnet create \
    --resource-group $AKS_PE_DEMO_RG \
    --name $DEMO_VNET \
    --address-prefixes $DEMO_VNET_CIDR \
    --subnet-name $DEMO_VNET_APP_SUBNET \
    --subnet-prefix $DEMO_VNET_APP_SUBNET_CIDR
    
    
# Run the following line to create an Azure Container Registry if you do not already have one
az acr create -n $MYACR -g $AKS_PE_DEMO_RG --sku premium

#deploy a default Ubuntu Azure virtual machine with
az vm create \
  --resource-group $ADO_PE_DEMO_RG \
  --name $VM_NAME \
  --image UbuntuLTS \
  --admin-username azureuser \
  --generate-ssh-keys

#Install Docker on the VM
ssh azureuser@publicIpAddress
sudo apt-get update
sudo apt install docker.io -y
sudo docker run -it hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
[...]
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

#Get network and subnet names


#Disable Network policies in subnet
az login

NETWORK_NAME=$(az network vnet list \
  --resource-group $ADO_PE_DEMO_RG \
  --query '[].{Name: name}' --output tsv)

SUBNET_NAME=$(az network vnet list \
  --resource-group $ADO_PE_DEMO_RG \
  --query '[].{Subnet: subnets[0].name}' --output tsv)

echo NETWORK_NAME=$NETWORK_NAME
echo SUBNET_NAME=$SUBNET_NAME

az network vnet subnet update \
 --name $SUBNET_NAME \
 --vnet-name $NETWORK_NAME \
 --resource-group $ADO_PE_DEMO_RG \
 --disable-private-endpoint-network-policies
 
 #Configure Private DNS zone 
 az network private-dns zone create \
  --resource-group $ADO_PE_DEMO_RG \
  --name "privatelink.azurecr.io"
  
 #Create an association link
 az network private-dns link vnet create \
  --resource-group $ADO_PE_DEMO_RG \
  --zone-name "privatelink.azurecr.io" \
  --name MyDNSLink \
  --virtual-network $NETWORK_NAME \
  --registration-enabled false
  
 #Create a private registry endpoint
 REGISTRY_ID=$(az acr show --name $MYACR \
  --query 'id' --output tsv)
  
 az network private-endpoint create \
    --name myPrivateEndpoint \
    --resource-group $ADO_PE_DEMO_RG \
    --vnet-name $NETWORK_NAME \
    --subnet $SUBNET_NAME \
    --private-connection-resource-id $REGISTRY_ID \
    --group-ids registry \
    --connection-name myConnection
    
 #Get private ip addresses
 NETWORK_INTERFACE_ID=$(az network private-endpoint show \
  --name myPrivateEndpoint \
  --resource-group $ADO_PE_DEMO_RG \
  --query 'networkInterfaces[0].id' \
  --output tsv)
  
 PRIVATE_IP=$(az resource show \
  --ids $NETWORK_INTERFACE_ID \
  --api-version 2019-04-01 \
  --query 'properties.ipConfigurations[1].properties.privateIPAddress' \
  --output tsv)

DATA_ENDPOINT_PRIVATE_IP=$(az resource show \
  --ids $NETWORK_INTERFACE_ID \
  --api-version 2019-04-01 \
  --query 'properties.ipConfigurations[0].properties.privateIPAddress' \
  --output tsv)
  
 #Create DNS records
 az network private-dns record-set a create \
  --name $MYACR \
  --zone-name privatelink.azurecr.io \
  --resource-group $ADO_PE_DEMO_RG

# Specify registry region in data endpoint name
az network private-dns record-set a create \
  --name ${MYACR}.${REGISTRY_LOCATION}.data \
  --zone-name privatelink.azurecr.io \
  --resource-group $ADO_PE_DEMO_RG
  
 az network private-dns record-set a add-record \
  --record-set-name $MYACR \
  --zone-name privatelink.azurecr.io \
  --resource-group $ADO_PE_DEMO_RG \
  --ipv4-address $PRIVATE_IP

# Specify registry region in data endpoint name
az network private-dns record-set a add-record \
  --record-set-name ${MYACR}.${REGISTRY_LOCATION}.data \
  --zone-name privatelink.azurecr.io \
  --resource-group $ADO_PE_DEMO_RG \
  --ipv4-address $DATA_ENDPOINT_PRIVATE_IP
  
#Validate private link connection
nslookup $MYACR.azurecr.io

#Private ip nslookup
[...]
myregistry.azurecr.io       canonical name = myregistry.privatelink.azurecr.io.
Name:   myregistry.privatelink.azurecr.io
Address: 10.0.0.6


#Registry operations
sudo az acr login --name $MYACR
echo FROM hello-world > Dockerfile
az acr build --image sample/hello-world:v1 \
  --registry $MYACR \
  --file Dockerfile .
  
az acr run --registry $MYACR \
  --cmd '$Registry/sample/hello-world:v1' /dev/null

#Create a service principal and assign permissions 

SERVICE_PRINCIPAL_NAME=prreddy-acr-service-principal

# Obtain the full registry ID for subsequent command args
ACR_REGISTRY_ID=$(az acr show --name $MYACR --query id --output tsv)

# Create the service principal with rights scoped to the registry.
# Default permissions are for docker pull access. Modify the '--role'
# argument value as desired:
# acrpull:     pull only
# acrpush:     push and pull
# owner:       push, pull, and assign roles
SP_PASSWD=$(az ad sp create-for-rbac --name http://$SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --role acrpull --query password --output tsv)
SP_APP_ID=$(az ad sp show --id http://$SERVICE_PRINCIPAL_NAME --query appId --output tsv)

# Output the service principal's credentials; use these in your services and
# applications to authenticate to the container registry.
echo "Service principal ID: $SP_APP_ID"
echo "Service principal password: $SP_PASSWD"

################Create a zone redundant Private Cluster in eastus2 or westus2 using kubenet and autoscaler########################
#kubenet - a simple /24 IP address range can support up to 251 nodes in the cluster (each Azure virtual network subnet reserves the #first three IP addresses for management operations)
#This node count could support up to 27,610 pods (with a default maximum of 110 pods per node with kubenet)
#Azure CNI - that same basic /24 subnet range could only support a maximum of 8 nodes in the cluster
#This node count could only support up to 240 pods (with a default maximum of 30 pods per node with Azure CNI)

VNET_ID=$(az network vnet show --resource-group $AKS_PE_DEMO_RG --name $DEMO_VNET --query id -o tsv)
SUBNET_ID=$(az network vnet subnet show --resource-group $AKS_PE_DEMO_RG --vnet-name $DEMO_VNET --name $DEMO_VNET_APP_SUBNET --query id -o tsv)

#get kubernetes version 
version=$(az aks get-versions -l $LOCATION --query 'orchestrators[-1].orchestratorVersion' -o tsv)
# Install the aks-preview extension
az extension add --name aks-preview

# Update the extension to make sure you have the latest version installed
az extension update --name aks-preview

az aks create \
	-n $AKS_PRIVATE_CLUSTER \
	-g $AKS_PE_DEMO_RG \
	--load-balancer-sku standard \
	--enable-managed-identity \
	--enable-private-cluster \
	--enable-addons monitoring \
	--kubernetes-version $version \
	--generate-ssh-keys \
	--location $LOCATION \
	--zones {1,2,3} \
	--network-plugin kubenet \
	--service-cidr 10.0.0.0/16 \
	--dns-service-ip 10.0.0.10 \
	--pod-cidr 10.244.0.0/16 \
	--docker-bridge-address 172.17.0.1/16 \
	--vnet-subnet-id $SUBNET_ID \
	--node-count 1 \
	--vm-set-type VirtualMachineScaleSets \
	--enable-cluster-autoscaler \
	--min-count 1 \
	--max-count 3 \
	--cluster-autoscaler-profile scan-interval=30s \
	--attach-acr $MYACR
```
### Create a Private endpoint in the ADO VNET and link vnet to private-dns 
```powershell
################Retrieve AKS Resource ID######################
aksresourceid=$(az aks show --name $AKS_PRIVATE_CLUSTER --resource-group $AKS_PE_DEMO_RG --query 'id' -o tsv)
################Retrieve the MC Resource Group and associated private DNS zone################
noderg=$(az aks show --name $AKS_PRIVATE_CLUSTER  --resource-group $AKS_PE_DEMO_RG --query 'nodeResourceGroup' -o tsv) 

##Note the Private DNS zone name and cluster A record" 
az resource list --resource-group $noderg | grep "privateDnsZones"
"id": "/subscriptions/<subid>/resourceGroups/<MC_rg>/providers/Microsoft.Network/privateDnsZones/7381ebda-9a7b-46af-b067-797fe3655227.privatelink.eastus2.azmk8s.io/virtualNetworkLinks/aksatteast-aksdemo-c24839-1e53cbe1"

export DNS_ZONE=7381ebda-9a7b-46af-b067-797fe3655227.privatelink.eastus2.azmk8s.io
export DNS_ARECORD=aksatteast-aksdemo-c24839-1e53cbe1

##############Create subnet, disable private endpoint network policies, create private endpoint############

NETWORK_NAME=$(az network vnet list \
  --resource-group $ADO_PE_DEMO_RG \
  --query '[].{Name: name}' --output tsv)

az network vnet subnet create --name $AKS_PE_SUBNET --resource-group $ADO_PE_DEMO_RG --vnet-name $NETWORK_NAME --address-prefixes $AKS_PE_SUBNET_CIDR
az network vnet subnet update --name $AKS_PE_SUBNET --resource-group $ADO_PE_DEMO_RG --vnet-name $NETWORK_NAME --disable-private-endpoint-network-policies true
az network private-endpoint create --name PrivateKubeApiEndpoint2 --resource-group $ADO_PE_DEMO_RG --vnet-name $NETWORK_NAME --subnet $AKS_PE_SUBNET --private-connection-resource-id $aksresourceid --group-ids management --connection-name myKubeConnection
##Go to the portal and get the ip address of the private-endpoint#############

##Duplicate the Private DNS zone saved earlier from the MC resource group in the Baston resource group"
az network private-dns zone create -g $ADO_PE_DEMO_RG -n $DNS_ZONE
az network private-dns record-set a add-record -g $ADO_PE_DEMO_RG  -z $DNS_ZONE -n $DNS_ARECORD  -a 10.0.1.4
az network private-dns link vnet create -g $ADO_PE_DEMO_RG -n MyDNSLinktoBastion -z $DNS_ZONE -v $NETWORK_NAME -e true
```

## Validate connectivity to cluster
```powershell 
Run the following from the ADO VM that has access to the endpoint created in the ADO Subnet. 

Connect to the cluster
To manage a Kubernetes cluster, you use kubectl, the Kubernetes command-line client. If you use Azure Cloud Shell, kubectl is already installed. To install kubectl locally, use the az aks install-cli command:
Azure CLI 
sudo az aks install-cli
To configure kubectl to connect to your Kubernetes cluster, use the az aks get-credentials command. This command downloads credentials and configures the Kubernetes CLI to use them.
Azure CLI  
az aks get-credentials --resource-group $AKS_PE_DEMO_RG --name $AKS_PRIVATE_CLUSTER
To verify the connection to your cluster, use the kubectl get command to return a list of the cluster nodes.
Azure CLI 
kubectl get nodes
The following example output shows the single node created in the previous steps. Make sure that the status of the node is Ready:
	NAME                       STATUS   ROLES   AGE     VERSION
	aks-nodepool1-31718369-0   Ready    agent   6m44s   v1.12.8
```

### Docker Compose and push to ACR
```powershell
git clone https://github.com/Azure-Samples/azure-voting-app-redis.git
cd azure-voting-app-redis
sudo apt install docker-compose 
sudo /usr/bin/docker-compose up -d
sudo docker images
sudo docker ps
curl http://localhost:8080
sudo /usr/bin/docker-compose down

sudo az acr login --name $MYACR
az acr list --resource-group $AKS_PE_DEMO_RG --query "[].{acrLoginServer:loginServer}" --output table
sudo docker tag azure-vote-front preastus2mycontainerregistry.azurecr.io/azure-vote-front:v1
sudo docker push preastus2mycontainerregistry.azurecr.io/azure-vote-front:v1
az acr repository list --name $MYACR --output table
```
### Deploy application
```powershell
 # Get the id of the service principal configured for AKS
 CLIENT_ID=$(az aks show --resource-group $AKS_PE_DEMO_RG --name $AKS_PRIVATE_CLUSTER --query "servicePrincipalProfile.clientId" --output tsv)

 # Get the ACR registry resource id
 ACR_ID=$(az acr show --name $MYACR --resource-group $AKS_PE_DEMO_RG --query "id" --output tsv)

#Deploy the application
vi azure-vote-all-in-one-redis.yaml

##############
containers:
- name: azure-vote-front
  image: preastus2mycontainerregistry.azurecr.io/azure-vote-front:v1
###############


kubectl apply -f azure-vote-all-in-one-redis.yaml
kubectl get service azure-vote-front --watch
```
### Create an ingress controller to an internal virtual network in Azure Kubernetes Service (AKS)

An ingress controller is a piece of software that provides reverse proxy, configurable traffic routing, and TLS termination for Kubernetes services. Kubernetes ingress resources are used to configure the ingress rules and routes for individual Kubernetes services. Using an ingress controller and ingress rules, a single IP address can be used to route traffic to multiple services in a Kubernetes cluster.

1) Install applications with Helm in Azure Kubernetes Service (AKS)

https://docs.microsoft.com/en-us/azure/aks/kubernetes-helm
```powershell

helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update
helm install my-nginx-ingress stable/nginx-ingress \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
    
kubectl --namespace default get services -o wide -w my-nginx-ingress-controller

```

2) Create an ingress controller. 

Create a file named internal-ingress.yaml using the following example manifest file. This example assigns 10.240.0.42 to the loadBalancerIP resource. Provide your own internal IP address for use with the ingress controller. Make sure that this IP address is not already in use within your virtual network.

```yaml
controller:
  service:
    loadBalancerIP: 10.240.0.42
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
 ```
 
 3) Create a private link service to the load balancer for the ingress. 
 
 4) Create a private endpoint pointing at the resource id of the pls 
 
 az network private-endpoint create --name PrivateingressEndpoint2 --resource-group Bastion --vnet-name BastionVMVNET --subnet BastionPESubnet2 --private-connection-resource-id <plsresourceid> --connection-name myingressConnection


### Daemonset deployment
1)	Please go through this forked git repo to look at the code for the daemonset, configmap and docker file: https://github.com/naveedzaheer/AKSNodeInstaller 
	a.	Please review this article as well: https://medium.com/@patnaikshekhar/initialize-your-aks-nodes-with-daemonsets-679fa81fd20e 
2)	Clone the repo on your machine and make changes to configmap as needed to use squid proxy: https://www.thegeekdiary.com/how-to-configure-docker-to-use-proxy/ 
3)	Connect the AKS cluster to Azure Container Registry. Please see this link: https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration#configure-acr-integration-for-existing-aks-clusters 
4)	Setup Service Endpoint for ACR but make sure to open firewall so that you can push container from VM that you are using for management: https://docs.microsoft.com/en-us/azure/container-registry/container-registry-vnet 
5)	Push the docker daemon image which will be similar to the following docker image (https://hub.docker.com/r/patnaikshekhar/node-installer ) to the ACR  https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli . 
a.	Please add the following to the daemonset spec. 
priorityClassName: system-node-critical

b.	Add all RFC 1918 and ACR addresses to no_proxy.  For reference on using wildcard *: https://docs.docker.com/config/daemon/systemd/ 
“HTTP_PROXY=proxy.local:3128" 
"HTTPS_PROXY=proxy.Local:3128"
"NO_PROXY=localhost,127.0.0.1,*.azurecr.io,gcr.io,mcr.microsoft.com,windows.net,10.*,172.16.*,172.31.*,192.168.*"

c.	You can also build your image by using the Dockerfile in the repo: https://github.com/naveedzaheer/AKSNodeInstaller. You can then push the image to ACR

6)	Change the daemonset yaml file to update the name and location of the node-installer docker image. It should be your ACR and the image that you just uploaded there.  

7)	Deploy the daemonset to the cluster. 

8)	You can use SSH to get to the AKS node to see the change made by daemonset after it is deployed https://docs.microsoft.com/en-us/azure/aks/ssh 


### Azure DevOps 

 

 
## Prerequisites
1)	A GitHub account, where you can create a repository. If you don't have one, you can create one for free.
2) 	An Azure DevOps organization. If you don't have one, you can create one for free. (An Azure DevOps organization is different from your GitHub organization. Give them the same name if you want alignment between them.)
If your team already has one, then make sure you're an administrator of the Azure DevOps project that you want to use.
3)	Allow the AKS Vnet and the jump server vnet access to Azure Container registry. Restrict access to an Azure container registry using an Azure virtual network or firewall rules
4) 	To run your jobs, you'll need at least one agent. A Linux agent can build and deploy different kinds of apps, including Java and Android apps. We support Ubuntu, Red Hat, and CentOS. Self-hosted Linux agents
5) Step by Step Guides
•	Build Images
•	Push Images
•	Deploy Manifests
•	Bake Manifests
•	Deployment strategies
o	Canary Deployment Strategy is most common
	https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/kubernetes/canary-demo?view=azure-devops

•	Build and Deploy to Azure Kubernetes Service 

