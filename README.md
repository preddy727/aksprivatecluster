# Azure Kubernetes 
## Overview

Create a private Azure Kubernetes Service cluster

## Pre-requisites 
The Azure CLI version 2.0.77 or later, and the Azure CLI AKS Preview extension version 0.4.18

### Architecture Diagram
* Process flow ![alt text](https://github.com/preddy727/AzureTerraformTemplates/blob/master/Images/architecture.png)

## Goals of the Lab
1. Create a private AKS cluster.   

## Install the latest Azure CLI AKS Preview extension

# Install the aks-preview extension
```powershell
az extension add --name aks-preview

# Update the extension to make sure you have the latest version installed
```powershell
az extension update --name aks-preview

az feature register --name AKSPrivateLinkPreview --namespace Microsoft.ContainerService
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AKSPrivateLinkPreview')].{Name:name,State:properties.state}"

az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Network


## Create a private AKS cluster
```powershell
    az login
    az account set --subscription <your subscription name>

    #create resource group
    az group create --name <your rg name> --location eastus
    az aks create -n <private-cluster-name> -g <private-cluster-resource-group> --load-balancer-sku standard --enable-private-cluster
    az aks install-cli
    az aks get-credentials --resource-group <your rg name> --name <your aks name>
     
    kubectl create clusterrolebinding kubernetes-dashboard -n kube-system --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
    az aks browse --resource-group <your rg name> --name <your aks name>
   
