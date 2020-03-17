# Azure Managed Identities

## Create a user assigned managed identity
```bash
# Azure login using cli
az login
# Create a resource group
az group create --name rg-aks-demo --location eastus --verbose
# Create a managed identity
az identity create --resource-group rg-aks-demo --name mi-acr-id
```
Get the id of the user assigned managed identity
```bash
# Get resource ID of the user-assigned identity
az identity show --resource-group rg-aks-demo --name mi-acr-id --query id --output tsv
# Get principal ID of the user-assigned identity
az identity show --resource-group rg-aks-demo --name mi-acr-id --query principalId --output tsv
```

By default the managed identity will not have any role assigned to it. It has to be done explicitly.

![Alt text](/images/mi-demo.jpg)

## AKS - managed identity configuration (Below steps are used only for preview version) 
```bash
# install the aks-preview extension
az extension add --name aks-preview
# Register the managed identity preview feature for AKS 
# Don't enable preview features on production subscriptions
az feature register --name MSIPreview --namespace Microsoft.ContainerService
# It might take several minutes for the status to show as Registered.
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/MSIPreview')].{Name:name,State:properties.state}"
# refresh the registration of the Microsoft.ContainerService resource
az provider register --namespace Microsoft.ContainerService
# It might take several minutes for the status to show as Registered.
az provider show -n Microsoft.ContainerService --query registrationState --output tsv
```

## AKS - create managed identity
The below command creates a user assigned managed identity with the name *"aks-abs-demo-agentpool"*  inside the **node resource group** *"MC_rg-aks-demo_aks-abs-demo_eastus"*.  
```bash
# create an AKS cluster with managed identity - preview mode
az aks create --resource-group rg-aks-demo --name aks-abs-demo --node-count 1 --generate-ssh-keys --enable-managed-identity --verbose
```
The managed identity is assigned Reader role by default on the **node resource group** *"MC_rg-aks-demo_aks-abs-demo_eastus"*.

![Alt text](/images/mi-aks-default.jpg)

The managed identity details can be viewed by running the commands below
```bash
# view the service principal of a managed identity using Azure CLI
az ad sp list --display-name aks-abs-demo-agentpool
# Get resource ID of the user-assigned identity
az identity show --resource-group MC_rg-aks-demo_aks-abs-demo_eastus --name aks-abs-demo-agentpool --query id --output tsv
# Get principal ID of the user-assigned identity
az identity show --resource-group MC_rg-aks-demo_aks-abs-demo_eastus --name aks-abs-demo-agentpool --query principalId --output tsv
```

## Grant the identity access to azure container registry
```bash
# Create a container registry
az acr create --resource-group rg-aks-demo --name acrabsdemo --sku Basic
# Get the resource id of the acr registry
$resourceID=(az acr show --resource-group rg-aks-demo --name acrabsdemo --query id --output tsv)
# Get principal ID of the user-assigned identity
$spID=(az identity show --resource-group MC_rg-aks-demo_aks-abs-demo_eastus --name aks-abs-demo-agentpool --query principalId --output tsv)
# Assign the role say 'acrpull' to the managed identity 'aks-abs-demo-agentpool' on the acr registry 'acrabsdemo'
az role assignment create --assignee $spID --scope $resourceID --role acrpull
# Similarly, the AcrImageSigner role can be assigned to the managed identity
az role assignment create --assignee $spID --scope $resourceID --role acrimagesigner
```

![Alt text](/images/mi-assignment.jpg)


