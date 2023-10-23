# 3 Workload Identity


## 3.1 Introduction

Workload identity allows you to securely access Azure resources from your Kubernetes applications using Azure AD identities. This way, you can avoid storing and managing credentials in your cluster, and instead rely on the native Kubernetes mechanisms to federate with external identity providers.

Workload identity replaces the deprecated pod identity feature, and is the recommended way to manage identity for workloads. 

more information about workload identity can be found here: [Use Azure AD workload identity with Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview?tabs=dotnet)

During this activity you will:

* Create an Azure Container Registry (ACR)
* Activate OIDC issuer on AKS cluster, and attach ACR
* Create an Azure Key Vault and secret.
* Create a Microsoft Entra (Azure AD) managed identity
* Create a Kubernetes service account
* Connect the MI to a Kubernetes service account with token federation
* Deploy a workload and verify authentication to keyvault with the workload identity


## 3.2 Environment variables

First, lets create a few environment variables, for ease of use. 

Pro-tip: Save them into a file, so that you can restore them if cloudshell times out (which it will).

````
export RESOURCE_GROUP="security-workshop"
export CLUSTERNAME="k8s"
export LOCATION="northeurope"
export FRONTEND_NAMESPACE="frontend"
export BACKEND_NAMESPACE="backend"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export USER_ASSIGNED_IDENTITY_NAME="keyvaultreader"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="keyvaultfederated"
export KEYVAULT_SECRET_NAME="redissecret"
````

Then create environment variables to hold the name of your keyvault and your container registry. These need to be unique, and you could for example use your corporate signum as part of the name.
````
export KEYVAULT_NAME=<your globally unique key vault name>
export ACRNAME=<your globally unique container registry name>
````

## 3.3 Create Azure Container Registry (ACR)

Later in this section you will build an application that you will store in Azure Container Registry. To create the registry, run this command:

````
 az acr create --resource-group $RESOURCE_GROUP --name $ACRNAME --sku Standard
````

## 3.4 Update AKS cluster with OIDC issuer and container registry attachment

Now, update the AKS cluster to attach it to the newly created container registry. Also register the cluster as OIDC issuer. Enabling OIDC on the cluster means it will be allowed to use Microsoft Entra ID as an external identity provider.

The update will take a couple of minutes, so... Coffee Time?

````
az aks update -g "$RESOURCE_GROUP" -n $CLUSTERNAME --enable-oidc-issuer --enable-workload-identity --attach-acr $ACRNAME

````

## 3.5 Get the OICD issuer URL

Query the AKS cluster for the OICD issuer URL with the following command, which stores the reult in an environment variable.

````
export AKS_OIDC_ISSUER="$(az aks show -n $CLUSTERNAME -g $RESOURCE_GROUP  --query "oidcIssuerProfile.issuerUrl" -otsv)"
````


After storing the output of a command in an environment variable, its always a good idea to check the content:
````
echo $AKS_OIDC_ISSUER
````

The variable should contain the Issuer URL similar to the following:
 ````https://eastus.oic.prod-aks.azure.com/9e08065f-6106-4526-9b01-d6c64753fe02/9a518161-4400-4e57-9913-d8d82344b504/````



## 3.6 Create keyvault

Create a keyvault in the same resource group as the other resources (not neccesary for in to be in the same RG, but for clarity)

 ````
 az keyvault create --resource-group $RESOURCE_GROUP  --location $LOCATION  --name $KEYVAULT_NAME 
 ````

## 3.7 Add a secret to the vault

Create a secret in the keyvault. This is the secret that will be used by the frontend application to connect to the (redis) backend.

 ````
 az keyvault secret set --vault-name $KEYVAULT_NAME  --name $KEYVAULT_SECRET_NAME  --value 'redispassword'
 ````

## 3.8 Add the Key Vault URL to the environment variable *KEYVAULT_URL*
 ````
 export KEYVAULT_URL="$(az keyvault show -g $RESOURCE_GROUP  -n $KEYVAULT_NAME --query properties.vaultUri -o tsv)"
 ````
Check the content of the environment variable. It should look similar to this
````
echo $KEYVAULT_URL
https://globallyuniquekeyvaultname.vault.azure.net/
````


 ## 3.9 Create a managed identity and grant permissions to access the secret

Create a User Managed Identity. We will give this identity *GET access* to the keyvault, and later associate it with a Kubernetes service account. 

 ````
 az account set --subscription $SUBSCRIPTION 
 az identity create --name $USER_ASSIGNED_IDENTITY_NAME  --resource-group $RESOURCE_GROUP  --location $LOCATION  --subscription $SUBSCRIPTION 

 ````

 Set an access policy for the managed identity to access the Key Vault

 ````
 export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group $RESOURCE_GROUP  --name $USER_ASSIGNED_IDENTITY_NAME  --query 'clientId' -otsv)"

 az keyvault set-policy --name $KEYVAULT_NAME  --secret-permissions get --spn $USER_ASSIGNED_CLIENT_ID 
 ````


 ## 3.10 Create Kubernetes service account

First, connect to the cluster if not already connected
 
 ````
 az aks get-credentials -n $CLUSTERNAME -g $RESOURCE_GROUP 
 ````

### 3.10.1 Create service account

The service account should exist in the frontend namespace, because it's the frontend service that will use that service account to get the credentials to connect to the (redis) backend service.

First create the namespace

#### Note: instead of using yaml-files we use inline text, for convenience. In a more realistic scenario, you would have created yaml-manifests and stored them under version control.

````
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: $FRONTEND_NAMESPACE
  labels:
    name: $FRONTEND_NAMESPACE
EOF
````



Then create a service account in that namespace. Notice the annotation for *workload identity*
````
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: $FRONTEND_NAMESPACE
  annotations:
    azure.workload.identity/client-id: $USER_ASSIGNED_CLIENT_ID
  name: $SERVICE_ACCOUNT_NAME
EOF
````


## 3.11 Establish federated identity credential

In this step we connect the service account with the user defined managed identity, using a federated credential. 

````
az identity federated-credential create --name $FEDERATED_IDENTITY_CREDENTIAL_NAME --identity-name $USER_ASSIGNED_IDENTITY_NAME --resource-group $RESOURCE_GROUP --issuer $AKS_OIDC_ISSUER --subject system:serviceaccount:$FRONTEND_NAMESPACE:$SERVICE_ACCOUNT_NAME
````

## 3.12 Build the application

Now its time to build the application, which is a simple python frontend. The application uses a redis container as it's "database layer" and connects to Redis using a password. This password is stored as a secret in a keyvault, and the frontend uses the workload identity feature to get access to that secret (the code uses Azure Identity client libraries)


In order to build the application, first clone the applications repository:

````
git clone https://github.com/pelithne/az-vote-with-workload-identity.git
````
Then CD into the directory where the (python) application resides and issue the acr build command

````
cd az-vote-with-workload-identity
cd azure-vote 
az acr build --image azure-vote:v1 --registry $ACRNAME .    // don't forget the dot at the end

````

The last command will build a container image inside your container registry, and give it the tag ````v1````

#### NOTE: you do not have to make any changes, but feel free to have a look at the code which can be found in ````main.py```` in the ````azrure-vote/azrure-vote```` folder (sorry! Naming folders is hard :-) ).

## 3.13 Deploy the application

We want to create some separation between the frontend and backend, by deploying them into different namespaces. To add more separation you can use network policies in the cluster to allow/disallow traffic between specific namespaces.


First, create the backend namespace


````
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: backend
  labels:
    name: backend
EOF
````


Now create the Redis backend deployment. The container image will be downloaded from ````mcr.microsoft.com/oss/bitnami/redis:6.0.8```` so you don't have to push anything to your own container registry.

As you can see in the yaml, we send the password as a clear text environment variable to the redis container. This is NOT how you should to this. Instead a keyvault should be used, but in the interest of timekeeping we allow this malpractice for now.

````
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-back
  namespace: $BACKEND_NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-back
  template:
    metadata:
      labels:
        app: azure-vote-back
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: azure-vote-back
        image: mcr.microsoft.com/oss/bitnami/redis:6.0.8
        ports:
        - containerPort: 6379
          name: redis
        env:
        - name: REDIS_PASSWORD
          value: "redispassword"
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-back
  namespace: $BACKEND_NAMESPACE
spec:
  ports:
  - port: 6379
  selector:
    app: azure-vote-back
EOF
````


Then create the frontend. In this case we already created the frontend namespace in an earlier step. The container image for the frontend is the one you built in a previous step ````$ACRNAME.azurecr.io/azure-vote:v1````

````
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-front
  namespace: $FRONTEND_NAMESPACE
  labels:
    azure.workload.identity/use: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: azure-vote-front
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 5 
  template:
    metadata:
      labels:
        app: azure-vote-front
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: $SERVICE_ACCOUNT_NAME
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: azure-vote-front
        image: $ACRNAME.azurecr.io/azure-vote:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 500m
        env:
        - name: REDIS
          value: "azure-vote-back.backend"
        - name: KEYVAULT_URL
          value: $KEYVAULT_URL
        - name: SECRET_NAME
          value: $KEYVAULT_SECRET_NAME
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-front
  namespace: $FRONTEND_NAMESPACE
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: azure-vote-front

EOF
````


If the frontend application can connect to the backend redis store, you have succeded!

You can further validate (or troubleshoot) by reading the logs of the frontend pod. To do that, you need to find the name of the pod:
````
kubectl get pods --namespace frontend
````
This should give a result timilar to this
````
NAME                                READY   STATUS    RESTARTS        AGE
azure-vote-front-85d6c66c4d-pgtw9   1/1     Running   29 (7m3s ago)   3h13m
````

Now you can read the logs of the application by running this command (but with YOUR pod name)
````
kubectl logs azure-vote-front-85d6c66c4d-pgtw9 --namespace frontend
````

You should be able to find a line like this:
````
Connecting to Redis... azure-vote-back.backend
````
And then a little later:
````
Connected to Redis!
````


Finally, to view the application you deployed, you first need to find out the public IP address

````
kubectl get svc --namespace frontend
````

You should get output similar to this
````
NAME               TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
azure-vote-front   LoadBalancer   10.0.72.219   52.157.236.22   80:30187/TCP   3h45m
````

Now use a browser to navigate the the ````EXTERNAL-IP```` and start voting!
