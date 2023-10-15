# 1. AKS Security Basics

This workshop/tutorial contains a number of different sections, each addressing a specific aspect of security in Kubernetes and specifically Azure Kubernetes Service, AKS.

You will go through the following steps to complete the workshop:

* Use Azure Portal and Azure Cloud Shell
* Setup Azure Container Registry to build and store docker images
* Create Kubernetes Cluster using AKS (Azure Kubernetes Service)
* 
* 
* 
*  ​


# 2. Prerequisites

## 2.1. Azure Portal

To make sure you are correctly setup with a working subscription, make sure you can log in to the Azure portal. Go to <https://portal.azure.com> Once logged in, feel free to browse around a little bit to get to know the surroundings!

It might be a good idea to keep a tab with the Azure Portal open during the workshop, to keep track of the Azure resources you create. We will only use CLI based tools during the workshop, but everything will be visible in the portal, and all the resources we create could also be created using the portal.

## 2.2. Azure Cloud Shell

We will use the Azure Cloud Shell throughout the workshop for all our command line needs. This is a web based shell that has all the necessary tools (like kubectl, az cli, helm, etc) pre-installed.

Start cloud shell by typing the address ````shell.azure.com```` into a web browser. If you have not used cloud shell before, you will be asked to create a storage location for cloud shell. Accept that and make sure that you run bash as your shell (not powershell).

**Protip: You can use ctrl-c to copy text in cloud shell. To paste you have to use shift-insert, or use the right mouse button -> paste. If you are on a Mac, you can use the "normal" Cmd+C/Cmd+V.**

**Protip II: Cloud Shell will time out after 20 minutes of inactivity. When you log back in, you will end up in your home directory, so be sure to ````cd```` into where you are supposed to be.**

## 2.3. Subscription
You need a valid Azure subscription. To use a specific subscription, use the ````account```` command like this (with your subscription id):
````
az account set --subscription <subscription-id>
````

# 3. Initial Setup

## 3.1. Create Resource Group

In all the commands in this section, we will use the name space name ````security-workshop````. If you choose a different name, just make sure to modify the commands accordingly

All resources in Azure exists in a *Resource Group*. The resource group is a "placeholder" for all the resources you create. 

All the resources you create in this workshop will use the same Resource Group. Use the commnd below to create the resource group.

````bash
az group create -n security-workshop -l swedencentral
````


## 3.2. Azure Kubernetes Service (AKS)

AKS is the hosted Kubernetes service on Azure.

Kubernetes provides a distributed platform for containerized applications. You build and deploy your own applications into a Kubernetes cluster, and let the cluster manage the availability and connectivity. In this step a sample application will be deployed into your own Kubernetes cluster. You will learn how to:

* Create an AKS Kubernetes Cluster
* Connect/validate towards the AKS Cluster
* Update Kubernetes manifest files
* Run an application in Kubernetes

### 3.2.1. Create AKS Cluster

In all the commands in this section, we will use the name space name ````k8s````. If you choose a different name, just make sure to modify the commands accordingly


Create an AKS cluster using ````az aks create```` command:

```azurecli
az aks create --resource-group security-workshop --name k8s --node-count 2 --node-vm-size Standard_D2s_v4 --no-ssh-key  --network-plugin azure --network-policy azure
```

#### NOTE: the ````network-plugin```` and ````--network-policy```` settings are needed for a later exercise


The creation time for the cluster should be around 4-5 minutes.

### 3.2.2. Get access to the AKS Cluster

In order to use `kubectl` you need to connect to the Kubernetes cluster, using the following command (which assumes that you named the cluster k8s):

```azurecli
az aks get-credentials --resource-group security-workshop --name k8s
```

To verify that your cluster is up and running you can try a kubectl command, like ````kubectl get nodes```` which  will show you the nodes (virtual machines) that are active in your cluster. If you followed the instructions, you should see two nodes.

````bash
kubectl get nodes
````


### Secure cluster access with Microsoft Entra ID (Azure Active Directory)

Azure Kubernetes Service (AKS) supports Azure Active Directory (AAD) integration, which allows you to control access to your cluster resources using Azure role-based access control (RBAC). 


In this section, you will learn how to:

- Update an existing AKS cluster to support AAD integration enabled.
- Create an AAD  group and assign it the Azure Kubernetes Service Cluster Admin Role.
- Create User in AAD


### Integrate AKS with Microsoft Entra ID

Update the existing AKS cluster to support Microsoft Entra ID integration, and configure a cluster admin group, and disable local admin accounts in AKS, as this will prevent anyone from using the **--admin** switch to get the cluster credentials.

````bash
az aks update -g security-workshop -n k8s --enable-azure-rbac --enable-aad --disable-local-accounts
````

At this point, if you try to communicate with the Kubernetes API it will not be permitted. For instance you can try

###TODO: validate access restriction in a good way.


Then create the security group in Azure AD for **Cluster Admin**

````bash
az ad group create --display-name admin --mail-nickname admin
````

Then grant the **Admin** group users the permissions to connect to and manage all aspects of the AKS cluster. For this you need the reource ID of your cluster and the object ID of your management group. For convenience you can put it in environment variables:

````bash
AKS_RESOURCE_ID=$(az aks show -g security-workshop -n k8s --query 'id' --output tsv)
ADMIN_GROUP_ID=$(az ad group show --group admin --query 'id' --output tsv)
````


````bash
az role assignment create --assignee $ADMIN_GROUP_ID --role "Azure Kubernetes Service RBAC Cluster Admin" --scope $AKS_RESOURCE_ID
 ````



Create the Admin user:

````bash
az ad user create --display-name admin  --user-principal-name <user-principal-name> --password Something_secure123
````

Assign the admin user to admin group for the AKS cluster.

First get the object id of the user as we will need this number to assign the user to the admin group. For convenience, you can put it in an environment variable

````bash
ADMIN_USER_ID=$(az ad user show --id <user-principal-name> --query 'id' --output tsv)
````

Assign the user to the admin security group.

````bash
az ad group member add --group admin --member-id $ADMIN_USER_ID
````

### Validate the access to the cluster.

````
az aks get-creadentials -g security-workshop -n k8s
````

This will trigger a login procedure after which you should be able to interact with the Kubernetes API 