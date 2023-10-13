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

## 3.1. Get the code

The code for this workshop is located in the same repository that you are looking at now. To *clone* the repository to your cloud shell, do this:

```bash
git clone https://github.com/pelithne/k8s-short.git
```

Then cd into the repository directory:

````bash
cd k8s-short
````

## 3.2. View the code

Azure Cloud Shell has a built in code editor, which is based on the popular VS Code editor. To view/edit all the files in the repository, run code like this:

````bash
code .
````

You can navigate the files in the repo in the left hand menu, and edit the files in the right hand window. Use the *right mouse button* to access the various commands (e.g. ````Save```` and ````Quit```` etc).

For instance, you may want to have a look in the ````application/azure-vote-app```` directory. This is where the code for the application is located. Here you can also find the *Dockerfile* which will be used to build your docker image, in a later step.

## 3.3. Create Resource Group

All resources in Azure exists in a *Resource Group*. The resource group is a "placeholder" for all the resources you create. 

All the resources you create in this workshop will use the same Resource Group. Use the commnd below to create the resource group.

````bash
az group create -n <resource-group-name> -l westeurope
````

## 3.4. Create an Azure Container Registry (ACR)

You will use a private Azure Container Registry to *build* and *store* the docker images that you will deploy to Kubernetes. The name of the the ACR needs to be globally unique, and should consist of only lower case letters. You could for instance use your corporate signum.

The reason it needs to be unique, is that your ACR will get a Fully Qualified Domain Name (FQDN), on the form ````<Your unique ACR name>.azurecr.io````

The command below will create the container registry and place it in the Resource Group you created previously.

````bash
az acr create --name <your unique ACR name> --resource-group <resource-group-name> --sku basic
````

## 3.5. Build images using ACR

Docker images can be built in a number of different ways, for instance by using the docker CLI. Another (and sometimes easier) way is to use *Azure Container Registry Tasks*, which is the approach we will use in this workshop.

The docker image is built using a so called *Dockerfile*. The Dockerfile contains instructions for how to build the image. Feel free to have a look at the Dockerfile in the repository (once again using *code*):

````bash
code application/azure-vote-app/Dockerfile
````

As you can see, this very basic Dockerfile will use a *base image* from ````tiangolo/uwsgi-nginx-flask:python3.6-alpine3.8````.

On top of that base image, it will install [redis-py](https://pypi.org/project/redis/) and then take the contents of the directory ````./azure-vote```` and copy it into the container in the path ````/app````.

To build the docker container image, cd into the right directory, and use the ````az acr build```` command:

````bash
cd application/azure-vote-app
az acr build --image azure-vote-front:v1 --registry <your unique ACR name> --file Dockerfile .
````

## 3.6. Azure Kubernetes Service (AKS)

AKS is the hosted Kubernetes service on Azure.

Kubernetes provides a distributed platform for containerized applications. You build and deploy your own applications into a Kubernetes cluster, and let the cluster manage the availability and connectivity. In this step a sample application will be deployed into your own Kubernetes cluster. You will learn how to:

* Create an AKS Kubernetes Cluster
* Connect/validate towards the AKS Cluster
* Update Kubernetes manifest files
* Run an application in Kubernetes

### 3.6.1. Create AKS Cluster

Create an AKS cluster using ````az aks create````. Give the cluster a name, e.g.  ````k8s````, and run the command:

```azurecli
az aks create --resource-group <resource-group-name> --name k8s --node-count 2 --node-vm-size Standard_D2s_v4 --no-ssh-key --attach-acr <your unique ACR name>
```

### note: in the command above, we attach the ACR created previously. This is to allow the AKS cluster to download images from the container registry. Behind the scenes, this is using Azure Managed Identity.

The creation time for the cluster should be around 5 minutes.

### 3.6.2. Get access to the AKS Cluster

In order to use `kubectl` you need to connect to the Kubernetes cluster, using the following command (which assumes that you named the cluster k8s):

```azurecli
az aks get-credentials --resource-group <resource-group-name> --name k8s
```

To verify that your cluster is up and running you can try a kubectl command, like ````kubectl get nodes```` which  will show you the nodes (virtual machines) that are active in your cluster. If you followed the instructions, you should see two nodes.

````bash
kubectl get nodes
````