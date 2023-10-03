## Deploy app with Workload Identity

* Activate OIDC issuer on AKS cluster
* Create an Azure Key Vault and secret.
* Create an Azure Active Directory (Azure AD) workload identity and  Kubernetes service account.
* Configure the managed identity for token federation.
* Deploy the workload and verify authentication with the workload identity.


````
export RESOURCE_GROUP="app-rg"
export LOCATION="westcentralus"
export SERVICE_ACCOUNT_NAMESPACE="frontend"
export FRONTEND_NAMESPACE="frontend"
export BACKEND_NAMESPACE="backend"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SUBSCRIPTION="$(az account show --query id --output tsv)"
export USER_ASSIGNED_IDENTITY_NAME="keyvaultreader"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="keyvaultfederated"
export KEYVAULT_NAME="<globally unique name>"
export KEYVAULT_SECRET_NAME="keyvaultsecret"
````


### Create reasource group
````
az group create --name $RESOURCE_GROUP --location $LOCATION
````

### Enable OIDC issuer
````

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
 az keyvault secret set --vault-name "${KEYVAULT_NAME}" --name "${KEYVAULT_SECRET_NAME}" --value 'letmein'
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

````
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: ${USER_ASSIGNED_CLIENT_ID}
  name: ${SERVICE_ACCOUNT_NAME}
  namespace: ${SERVICE_ACCOUNT_NAMESPACE}
EOF
````


### Establish federated identity credential

````
az identity federated-credential create --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name ${USER_ASSIGNED_IDENTITY_NAME} --resource-group ${RESOURCE_GROUP} --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
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
          value: "letmein"
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-back
spec:
  ports:
  - port: 6379
  selector:
    app: azure-vote-back
EOF
````


Frontend
````
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-vote-front
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
        image: pelithnepubacr.azurecr.io/azure-vote:v18
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 500m
        env:
        - name: REDIS
          value: "azure-vote-back"
        - name: KEYVAULT_URL
          value: "https://pelithnekeys2.vault.azure.net/"
        - name: SECRET_NAME
          value: "redissecret"
---
apiVersion: v1
kind: Service
metadata:
  name: azure-vote-front
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: azure-vote-front

EOF
````
