Install the SMB driver using the following Helm commands:
Add the specified chart repository, get the latest information for available charts, and install the specified chart archive.
```
helm repo add azurefile-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/azurefile-csi-driver/master/charts
```
```
helm repo update
```
```
helm install azurefile-csi-driver azurefile-csi-driver/azurefile-csi-driver --namespace kube-system
```


CREATE AZURE_CONTAINER_REGISTRY



To create SMB we need create Azure Container Registry.
Then we need to create file share and secret
Then we need to create pv-pvc.yaml files
After push deploy our test file docker to the ACR
After create deployment yaml pod for creating resource AzureFileShare with our file 


# STEP 1. Creating Azure Container Registry
```
az acr create --resource-group RESOURCE_GROUP --name ACR_CONATINER_REGISTRY_NAME --sku Basic
az acr update --name ACR_CONATINER_REGISTRY_NAME --resource-group RESOURCE_GROUP --admin-enabled true
```
```
ACR_USERNAME= az acr credential show -n ACR_CONATINER_REGISTRY_NAME --resource-group RESOURCE_GROUP --query username
ACR_PASSWORD= az acr credential show -n ACR_CONATINER_REGISTRY_NAME --resource-group RESOURCE_GROUP --query 'passwords[0].value'
```
# STEP 2. Creating AzureFile share
```
az storage account create --name STORAGE_ACCOUNT_NAME -g RESOURCE_GROUP -l LOCATION --sku Standard_LRS

AZURE_STORAGE_CONNECTION_STRING= az storage account show-connection-string -n STORAGE_ACCOUNT_NAME -g RESOURCE_GROUP -o tsv

az storage share create -n AZURE_FILE_SHARE_NAME --connection-string AZURE_STORAGE_CONNECTION_STRING
```

After completing the operations, we can check the storage account and the File shares we just created through the Azure Portal.

Microsoft Defender for Storage
Microsoft Defender is an advanced threat protection tool which we can use for potential security threats in both our Azure-native and hybrid environments. In order to protect our File shares service, we will benefit from Microsoft Defender’s native intelligent security layer for storages.
It is possible to activate Microsoft Defender for Storage either at the subscription level or at the resources level that we choose. Now, using the commands below, let’s activate advanced threat protection for the storage account we created.
```
az security atp storage update --resource-group RESOURCE_GROUP --storage-account STORAGE_ACCOUNT_NAME --is-enabled true
```

# STEP 3. Creating Persistent Volumes and Persistent Volume Claims
```
STORAGE_KEY= $(az storage account keys list --resource-group RESOURCE_GROUP --account-name STORAGE_ACCOUNT_NAME --query "[0].value" -o tsv)

kubectl create secret generic K8S_AZURE_SOTRAGE_SECRET_NAME --from-literal=azurestorageaccountname=STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STORAGE_KEY
```

A PersistentVolumeClaim (PVC) is a Kubernetes resource that represents a request for storage by a pod. It can specify requirements for storage, such as the size, access mode, and storage class. Kubernetes uses the PVC to find an available PV that satisfies the PVC’s requirements.

A PVC is created by a pod to request storage from a PV. Once a PVC is created, it can be mounted as a volume in a pod. The pod can then use the mounted volume to store and retrieve data.

PV_PVC_NAME.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: PV_NAME
  namespace: NAMESPACE
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  azureFile:
    secretName: K8S_AZURE_SOTRAGE_SECRET_NAME
    shareName: AZURE_FILE_SHARE_NAME
    readOnly: false
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: PVC_NAME
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 5Gi
```


```
kubectl apply -f DEPLOYMENT_NAME.yaml
kubectl get pv PV_NAME
kubectl get pvc PVC_NAME
kubectl describe pvc PVC_NAME
```

# STEP 4. CREATE DEPLOYMENT FILE

ACR_SECRET= kubectl create secret docker-registry ARC_SECRET --docker-server=https://ACR_CONATINER_REGISTRY_NAME.azurecr.io --docker-username=ACR_USERNAME --docker-password=ACR_PASSWORD --docker-email=YOUR_EMAIL

And now we are going to create simple project for C# console that will create file in Azure FileShare for us:

createfile.csproj
```
using (StreamWriter writer = File.CreateText("/mnt/azure/mytext.txt"))
  {
     await writer.WriteLineAsync("hello");
  } 
await Task.Delay(TimeSpan.FromHours(1));
```

DOCKERFILE
```
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["PATH_TO_FOLDER/createfile.csproj", "PATH_TO_FOLDER/"]
RUN dotnet restore "PATH_TO_FOLDER/createfile.csproj"
COPY ./createfile ./createfile
WORKDIR "/src/createfile"
RUN dotnet build "createfile.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "createfile.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "createfile.dll"]
```

```
az acr login --name ACR_CONATINER_REGISTRY_NAME --resource-group RESOURCE_GROUP
docker login ACR_CONATINER_REGISTRY_NAME.azurecr.io -u ACR_USERNAME -p ACR_PASSWORD
docker build -t ACR_CONATINER_REGISTRY_NAME.azurecr.io/DEPLOYMENT_APP_NAME:v1 .   
docker push ACR_CONATINER_REGISTRY_NAME.azurecr.io/DEPLOYMENT_APP_NAME:v1
```


DEPLOYMENT_NAME.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: DEPLOYMENT_NAME
  labels:
    app: DEPLOYMENT_APP_NAME
spec:
  replicas: 1
  selector:
    matchLabels:
      app: DEPLOYMENT_APP_NAME
  template:
    metadata:
      labels:
        app: DEPLOYMENT_APP_NAME
    spec:
      containers:
      - name: DEPLOYMENT_APP_NAME
        image: ACR_CONATINER_REGISTRY_NAME.azurecr.io/DEPLOYMENT_APP_NAME:v1
        volumeMounts:
        - name: azurefileshare
          mountPath: /mnt/azure
      volumes:
      - name: azurefileshare
        persistentVolumeClaim:
          claimName: PVC_NAME
      imagePullSecrets:
      - name: K8S_AZURE_SOTRAGE_SECRET_NAME
```
```
kubectl apply -f ./DEPLOYMENT_NAME.yaml
kubectl get pods
```

Ensure that the storage account key has access to File Services.
Example to Check File
```
kubectl exec -it $(kubectl get pods -l app=DEPLOYMENT_APP_NAME --output=jsonpath="{.items[0].metadata.name}") -- cat /mnt/azure/mytext.txt
```



```
kubectl exec -it $(kubectl get pods -l app=DEPLOYMENT_APP_NAME --output=jsonpath="{.items[0].metadata.name}") -- df -h
kubectl describe pod $(kubectl get pods -l app=DEPLOYMENT_APP_NAME --output=jsonpath="{.items[0].metadata.name}")
```