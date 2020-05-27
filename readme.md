# Azure Managed Identities

### Problem Statement
Managing credentials in the application code is a common challenge that is faced while building the cloud native applications. The common issues that are faced by enterprises are as follows:
* managing secrets when they expire & periodic credential rotation
* auditing the service principal (automation account) which accessed the secured service

### Solution - Managed Identity
Managed Identity - It provides Azure services with an automatically managed identity in Azure AD, which is used to authenticate to any service that supports Azure AD authentication. Azure takes care of rolling the credentials that are used by the service instance.

### How it works?
Internally, managed identities are service principals of a special type, which are locked to only be used with Azure resources. If Managed Identity is enabled for a Azure service like VM or AKS, a service principal is created in Azure AD. This is called as **system-assigned MI**. After the identity is created, the credentials are provisioned onto the Azure service instance. The identity on the resource is configured by updating the Azure Instance Metadata Service (IMDS) endpoint with the service principal client ID and certificate. The lifecycle of a system-assigned identity is directly tied to the Azure service instance that it's enabled on. If the instance is deleted, Azure automatically cleans up the credentials and the identity in Azure AD. RBACs are used to grant access to a resource & this has to be done explicitly for the system-assigned identity.

![Alt text](/images/msi.jpg)

It is also possible to create a **user-assigned MI** as a standalone azure resource. After the identity is created, the identity can be assigned to one or more Azure service instances. The lifecycle of a user-assigned identity is managed separately from the lifecycle of the Azure service instances to which it's assigned. RBACs are used to grant access to a resource & this has to be done explicitly for the user-assigned identity.

To see a Managed Identity in action with Azure WebApp (PaaS Service) & Azure Key Vault, refer this [link](https://github.com/abhinabsarkar/webapp-mi-keyvault)

## AAD Pod Identities
In AKS, pods need access to other Azure services, say Cosmos DB or Key Vault. Rather than defining the credentials in container image or injecting as kubernetes secret, the best practice is to use managed identities.
> Managed pod identities is an open source project, and as of May 1st, 2020, it is not supported by Azure technical support.

In AKS, two components are deployed by the cluster operator to allow pods to use managed identities:
* **Node Management Identity (NMI) server** - It is a pod that runs as a [DaemonSet](https://github.com/abhinabsarkar/k8s-networking/blob/master/concepts/pod-readme.md#daemonset) on each node in the AKS cluster. The NMI server listens for pod requests to Azure services.
* **Managed Identity Controller (MIC)** - It is a central pod with permissions to query the Kubernetes API server and checks for an Azure identity mapping that corresponds to a pod.

When pods request access to an Azure service, network rules redirect the traffic to the Node Management Identity (NMI) server. The NMI server identifies pods that request access to Azure services based on their remote address, and queries the Managed Identity Controller (MIC). The MIC checks for Azure identity mappings in the AKS cluster, and the NMI server then requests an access token from Azure Active Directory (AD) based on the pod's identity mapping. Azure AD provides access to the NMI server, which is returned to the pod. This access token can be used by the pod to then request access to services in Azure.

![Alt text](/images/pod-identities.jpg)

In the above example, a developer creates a pod that uses a managed identity to request access to an Azure SQL Server instance:
1. Cluster operator first creates a service account that can be used to map identities when pods request access to services.
2. The NMI server and MIC are deployed to relay any pod requests for access tokens to Azure AD.
3. A developer deploys a pod with a managed identity that requests an access token through the NMI server.
4. The token is returned to the pod and used to access an Azure SQL Server instance.

To see a Pod Identity in action with AKS & Azure Key Vault, refer this [link](https://github.com/abhinabsarkar/podidentity)

## Basic example of Managed Identity (Not Pod Identity) with AKS
### Create a user assigned managed identity
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

### AKS - managed identity configuration (Below steps are used only for preview version) 
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

### AKS - create managed identity
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

### Grant the identity access to azure container registry
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

## References
https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview  
https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-identity  
https://github.com/Azure/aad-pod-identity  
https://medium.com/@harioverhere/using-aad-podidentity-with-azure-kubernetes-service-42a53fd04006