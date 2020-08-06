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

## Create a private AKS cluster
```powershell

# Please provide your subscription id here
export APP_SUBSCRIPTION_ID=c2483929-bdde-40b3-992e-66dd68f52928
# Please provide your unique prefix to make sure that your resources are unique
export APP_PREFIX=preastus2
# Please provide your region
export LOCATION=EastUS2
export REGISTRY_LOCATION=EastUS2

export VNET_PREFIX="192.168."

export AKS_PE_DEMO_RG=$APP_PREFIX"-aksdemo-rg"
export ADO_PE_DEMO_RG=$APP_PREFIX"-adodemo-rg"
export DEMO_VNET=$APP_PREFIX"-aksdemo-vnet"
export DEMO_VNET_CIDR=$VNET_PREFIX"0.0/16"
export DEMO_VNET_APP_SUBNET=app_subnet
export DEMO_VNET_APP_SUBNET_CIDR=$VNET_PREFIX"1.0/24"

# set this to the name of your Azure Container Registry.  It must be globally unique
export MYACR=$APP_PREFIX"myContainerRegistry"
export VM_NAME=myDockerVM



az login
az account set --subscription $APP_SUBSCRIPTION_ID

#create resource group
az group create --name $AKS_PE_DEMO_RG --location $LOCATION
az group create --name $ADO_PE_DEMO_RG --location $LOCATION

#get kubernetes version 
version=$(az aks get-versions -l $LOCATION --query 'orchestrators[-1].orchestratorVersion' -o tsv)

#Create a vnet and subnet 
az network vnet create \
    --resource-group $AKS_PE_DEMO_RG \
    --name $DEMO_VNET \
    --address-prefixes $DEMO_VNET_CIDR \
    --subnet-name $DEMO_VNET_APP_SUBNET \
    --subnet-prefix $DEMO_VNET_APP_SUBNET_CIDR
    
    
# Run the following line to create an Azure Container Registry if you do not already have one
az acr create -n $MYACR -g $AKS_PE_DEMO_RG --sku basic

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
NETWORK_NAME=$(az network vnet list \
  --resource-group $ADO_PE_DEMO_RG \
  --query '[].{Name: name}' --output tsv)

SUBNET_NAME=$(az network vnet list \
  --resource-group $ADO_PE_DEMO_RG \
  --query '[].{Subnet: subnets[0].name}' --output tsv)

echo NETWORK_NAME=$NETWORK_NAME
echo SUBNET_NAME=$SUBNET_NAME

#Disable Network policies in subnet
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

#Public ip nslookup 
nskookup $MYACR.eastus2.cloudapp.azure.com
[...]
Non-authoritative answer:
Name:   myregistry.easuts2.cloudapp.azure.com
Address: 40.78.103.41

#Registry operations
az acr login --name $MYACR
echo FROM hello-world > Dockerfile
az acr build --image sample/hello-world:v1 \
  --registry $MYACR \
  --file Dockerfile .
  
az acr run --registry $MYACR \
  --cmd '$Registry/sample/hello-world:v1' /dev/null

docker pull myregistry.azurecr.io/hello-world:v1


#Create a service principal and assign permissions 
az ad sp create-for-rbac --skip-assignment
#Retrieve the resource ids of vnet and subnet 
VNET_ID=$(az network vnet show --resource-group myResourceGroup --name myAKSVnet --query id -o tsv)
SUBNET_ID=$(az network vnet subnet show --resource-group myResourceGroup --vnet-name myAKSVnet --name myAKSSubnet --query id -o tsv)
#Assign the service principal for your AKS cluster Contributor permissions on the virtual network
az role assignment create --assignee <appId> --scope $VNET_ID --role Contributor

################Create a zone redundant Private Cluster in eastus2 or westus2 using kubenet and autoscaler########################
#kubenet - a simple /24 IP address range can support up to 251 nodes in the cluster (each Azure virtual network subnet reserves the #first three IP addresses for management operations)
#This node count could support up to 27,610 pods (with a default maximum of 110 pods per node with kubenet)
#Azure CNI - that same basic /24 subnet range could only support a maximum of 8 nodes in the cluster
#This node count could only support up to 240 pods (with a default maximum of 30 pods per node with Azure CNI)

az aks create \
	-n <private-cluster-name> \
	-g <resource-group-name> \
	--load-balancer-sku standard \
	--enable-managed-identity
	--enable-private-cluster \
	--enable-addons monitoring \
	--kubernetes-version $version \
	--generate-ssh-keys \
	--location <eastus2> \
	--node-zones {1,2,3} \
	--network-plugin kubenet \
	--service-cidr 10.0.0.0/16 \
	--dns-service-ip 10.0.0.10 \
	--pod-cidr 10.244.0.0/16 \
	--docker-bridge-address 172.17.0.1/16 \
	--vnet-subnet-id <$SUBNET_ID> \
	--node-count 1 \
	--vm-set-type VirtualMachineScaleSets \
	--enable-cluster-autoscaler \
	--min-count 1 \
	--max-count 3 \
	--cluster-autoscaler-profile scan-interval=30s \
	--attach-acr $MYACR
```
### Create a Private endpoint in the Bastion VNET and link vnet to private-dns 
```powershell
################Retrieve AKS Resource ID######################
aksresourceid=$(az aks show --name <private-cluster-name> --resource-group <resource-group-name> --query 'id' -o tsv)
################Retrieve the MC Resource Group and associated private DNS zone################
noderg=$(az aks show --name <private-cluster-name>  --resource-group <resource-group-name> --query 'nodeResourceGroup' -o tsv) 

##Note the Private DNS zone name and cluster A record" 
az resource list --resource-group $noderg | grep "privateDnsZones"
"id": "/subscriptions/<subid>/resourceGroups/<MC_rg>/providers/Microsoft.Network/privateDnsZones/77cb2ebb-a082-43e7-a18e-0337bf24dfce.eastus2.azmk8s.io/virtualNetworkLinks/aksatteast-aksdemo-c24839-1e53cbe1"

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



