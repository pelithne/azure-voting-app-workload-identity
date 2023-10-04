## Deploy app with Workload Identity

* Activate OIDC issuer on AKS cluster
* Create an Azure Key Vault and secret.
* Create an Azure Active Directory (Azure AD) workload identity and  Kubernetes service account.
* Configure the managed identity for token federation.
* Deploy the workload and verify authentication with the workload identity.


````
export RESOURCE_GROUP="app-rg"
export LOCATION="westeurope"
export FRONTEND_NAMESPACE="frontend"
export BACKEND_NAMESPACE="backend"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export USER_ASSIGNED_IDENTITY_NAME="keyvaultreader"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="keyvaultfederated"
export KEYVAULT_NAME="pelithnekv"
export KEYVAULT_SECRET_NAME="redissecret"
expoort ACRNAME=<globally unique name>
````


### Create reasource group
````
az group create --name $RESOURCE_GROUP --location $LOCATION
````

### Create Container Registry

````
 az acr create --resource-group $RESOURCE_GROUP --name $ACRNAME --sku Premium 
````

### Create cluster with OIDC issuer and calico network policies
````
az aks create -g "${RESOURCE_GROUP}" -n myAKSCluster --node-count 1 --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys --network-plugin kubenet --network-policy calico --attach-acr $ACRNAME

````

### Get the OICD issuer URL
````
export AKS_OIDC_ISSUER="$(az aks show -n myAKSCluster -g "${RESOURCE_GROUP}" --query "oidcIssuerProfile.issuerUrl" -otsv)"
````

The variable should contain the Issuer URL similar to the following example:
 ````https://eastus.oic.prod-aks.azure.com/00000000-0000-0000-0000-000000000000/00000000-0000-0000-0000-000000000000/````

 ### Create keyvault

 ````
 az keyvault create --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --name "${KEYVAULT_NAME}"
 ````

 ### Add a secret to the vault

 ````
 az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name "${KEYVAULT_SECRET_NAME}" --value 'redispassword'
 ````

Add the Key Vault URL to the environment variable *KEYVAULT_URL*
 ````
 export KEYVAULT_URL="$(az keyvault show -g "${RESOURCE_GROUP}" -n ${KEYVAULT_NAME} --query properties.vaultUri -o tsv)"
 ````

 ### Create a managed identity and grant permissions to access the secret

 ````
 az account set --subscription "${SUBSCRIPTION}"
 az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP}" --location "${LOCATION}" --subscription "${SUBSCRIPTION}"

 ````

 #### Set an access policy for the managed identity to access the Key Vault

 ````
 export USER_ASSIGNED_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -otsv)"

 az keyvault set-policy --name "${KEYVAULT_NAME}" --secret-permissions get --spn "${USER_ASSIGNED_CLIENT_ID}"
 ````


 ### Create Kubernetes service account

Connec to the cluster
 
 ````
 az aks get-credentials -n myAKSCluster -g "${RESOURCE_GROUP}"
 ````

Create service account

The service account should exist in the frontend namespace, because it's the frontend service that will use that service account to get the credentials to connect to the backend service, in the backend namespace.

First create the namespace

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

Then create the service account in that namespace
````
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: $FRONTEND_NAMESPACE
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
EOF
````


### Establish federated identity credential

````
az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name ${USER_ASSIGNED_IDENTITY_NAME} --resource-group ${RESOURCE_GROUP} --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${FRONTEND_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
````

### Build the application
````
git clone git@github.com:pelithne/azure-voting-app-redis.git
````

````
cd azure-voting-app-redis
cd azure-vote 
az acr build --image azure-vote:v1 --registry $CONTAINERREGISTRY .

````



### Deploy the application

We want to create some separation between the frontend and backend, by deploying them into different namespaces. Later we will add more separation by introducing network policies in the cluster to allow/disallow traffic between specific namespaces.


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

cd only neccesery if using yaml files
````
cd ..
````

Backend
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


Then create the frontend



Frontend
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
        image: pelithnepubacr.azurecr.io/azure-vote:v19
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


### Network policies
The cluster is deployed using kubenet CNI, which means we have to use Calico Network Policies. 

````



````