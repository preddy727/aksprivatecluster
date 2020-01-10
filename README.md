# Infrastructure as Code on Azure
## Overview

Deploy a sample Tomcat Application on an Azure Virtual Machine Scale Set

## Pre-requisites 
Option 1 for Ubuntu management VM  
* Create a Terraform Ubuntu virtual machine with managed identities using a marketplace template [here](https://docs.microsoft.com/en-us/azure/terraform/terraform-vm-msi)

Option 2 for CentOS management VM 



* Create a Terraform Centos virtual machine with managed identities. 
   - Source code - AzureTerraformTemplates/POCtoPattern/MgmtVmMI/
   - Populate variables.tf and run terraform apply within folder 

Setup Steps
* Contributor permission helps MSI on VM to use Terraform to create resources outside the VM resource group. You can easily achieve this action by running a script once inside the Terraform Linux vm. ~/tfEnv.sh
* The VM has a Terraform remote state back end. To enable it on your Terraform deployment, copy the remoteState.tf file from tfTemplate directory to the root of the Terraform scripts. cp ~/tfTemplate/remoteState.tf .
* Install the Packer precompiled binary on the Terraform VM [download](https://www.packer.io/intro/getting-started/install.html#precompiled-binaries)
* Clone the Github repository to the Terraform VM [download](https://github.com/preddy727/AzureTerraformTemplates.git)

## Recommended Reading
* Series of Labs for Terraform on Azure [here](https://azurecitadel.com/automation/terraform/)

### Architecture Diagram
* Process flow ![alt text](https://github.com/preddy727/AzureTerraformTemplates/blob/master/Images/architecture.png)

## Goals of the Lab
1. Create a customized Ubuntu managed image with Tomcat installed 
2. Store the image in a shared image gallery
3. Create a Key Vault enabled for disk encryption and a Key
4. Deploy a Virtual machine scale set
    * Enable service endpoint for Key Vault. 
    * Update key vault access policy to allow scale set subnet. 
    * Enable disk encryption extension and associate with key
5. Access Tomcat webpage 

## Exercises

* [Create a customized Ubuntu managed image with Tomcat installed](#Custom-Ubuntu-Tomcat-with-Packer)
* [Create a Key Vault enabled for disk encryption and a Key](#create-the-key-vault-disk-encryption-with-key)
* [Deploy a Virtual machine scale set](#deploy-a-vmss)
* [Access Tomcat webpage](#Access-the-tomcat-webpage)


## Custom Ubuntu Tomcat image with Packer
### [Back to Excercises](#exercises)

Start Here by reading the following document on how to build an Azure build pipeline 
POCtoPattern/Azure Build pipeline - Customized image in Shared Image Gallery.docx

1. Create an Azure DevOps project

2. Import the  Packer json into Azure repot

3. Install the hosted build agent into the Terraform linux vm 

4. Setup a build pipeline with tasks using the replace tokens module to populate environment variables into the json file. 

- Documentation is in Azure Build pipeline - Customized image in Shared Image Gallery.docx

5. The output is a customized managed image. 

6. Note the resource group and name of the final managed image. 


## Create the key vault disk encryption with key
### [Back to Excercises](#exercises)

1.Login to Terraform vm with a managed identity where github repository was cloned and run the following commands.

2. Change to the Source directory for key vault which is AzureTerraformTemplates/POCtoPattern/KeyVaultDiskEncryption/

3. 
export ARM_USE_MSI=true

Terraform init 

Terraform apply -out output



## Deploy a Virtual machine scale set
### [Back to Excercises](#exercises)

Create a release pipeline using the shared image gallery build artificat created in 

AzureTerraformTemplates/POCtoPattern/Azure Release pipeline - Deply Scale Set using customized tomcat image in SIG.docx

## Access Tomcat webpage
### [Back to Excercises](#exercises)
