Install the SMB driver using the following Helm commands:
Add the specified chart repository, get the latest information for available charts, and install the specified chart archive.
```
helm repo add csi-driver-smb https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts 
```
```
helm repo update
```
```
helm install csi-driver-smb csi-driver-smb/csi-driver-smb --namespace kube-system --version v1.15.0
```

Confirm that your SQL database is in the same network as your Arc-enabled Kubernetes cluster and SMB file share

Find and save the connection string for the SQL database that you created


# Why Do You Need an SMB File Share?

The SMB file share acts as persistent storage for the Logic App runtime to store state, workflows, triggers, and artifacts. Azure Logic App Standard uses it to maintain workflow states and logs.

# How to Set Up an SMB File Share?
If you are running Kubernetes locally, you need to set up a shared network folder that Kubernetes pods can mount. You can achieve this using Azure File Share (if you have cloud access) or local network storage (like Samba/SMB).

Here are the options to create an SMB file share:

OPTION 1: Use Local SMB File Share (for local Kubernetes)

Install Samba (SMB) Server on Windows
Go to Control Panel > Programs > Turn Windows features on or off.
Enable SMB 1.0/CIFS File Sharing Support.
Create the Folder to Share
Create a folder (for example) at C:\SMBShare.
Right-click the folder, select Properties > Sharing > Advanced Sharing.
Check Share this folder and set the share name (for example, SMBShare).
Set Permissions so that Everyone has Read/Write permissions.
Test the Share
Run this command from another system or a Docker container:
net use \\<your-local-ip>\SMBShare /user:<your-username> <your-password>

Access SMB from Kubernetes
Update your Kubernetes PersistentVolume to point to the share like this:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: smb-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  csi:
    driver: smb.csi.k8s.io
    volumeHandle: unique-volume-id
    volumeAttributes:
      source: "//host.docker.internal/SMBShare"
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - vers=3.0
```

Option 2: Use Azure File Share (if Cloud is Available)
If you have access to Azure storage account, follow these steps:
Create a Storage Account
```
az storage account create --name STORAGE_ACCOUNT_NAME --resource-group GROUP_NAME --location LOCATION --sku Standard_LRS
```

Create a File Share
```
az storage share-rm create --resource-group GROUP_NAME --storage-account STORAGE_ACCOUNT_NAME --name FILE_SHARE_NAME --quota 10
```

Get the Connection String
```
az storage account show-connection-string --name STORAGE_ACCOUNT_NAME --resource-group GROUP_NAME
```

Mount the Share in Kubernetes
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azure-file-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  azureFile:
    secretName: azure-secret
    shareName: logicappshare
    readOnly: false
```

****
To mount an SMB File Share or an Azure File Share in Kubernetes, you need to create both a PersistentVolume (PV) and a PersistentVolumeClaim (PVC). These resources will allow your Kubernetes pods to access the file share.

Here is a step-by-step guide to mounting the SMB share or Azure File share.
Access to SMB Share or Azure File Share:
```
  For SMB: You need the SMB share URL, credentials (username, password), and permissions.
```
```
  For Azure File: You need the storage account name, file share name, and storage access key.
```
  Create a Kubernetes Secret for accessing the SMB/Azure file share.
  Create PersistentVolume (PV) and PersistentVolumeClaim (PVC) for the storage.
  Mount the volume in the pod definition to make the storage available in the container.


Create the Kubernetes Secret

The secret will store the access credentials required to authenticate with the SMB or Azure file share.

For Local SMB Share

For Azure File Share

If you have an Azure Storage Account and Azure File Share, use the following command:

Create the PersistentVolume (PV)

A PersistentVolume (PV) is a storage resource in Kubernetes. Here’s how to define it for an SMB or Azure File Share.
For Local SMB Share
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: smb-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  csi:
    driver: smb.csi.k8s.io
    volumeHandle: unique-volume-id
    volumeAttributes:
      source: "//host.docker.internal/SMBShare"
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - vers=3.0
```


For Azure File Share
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azure-file-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  azureFile:
    secretName: azure-secret
    shareName: logicappshare
    readOnly: false
```

Explanation:
azureFile: This part is specific to Azure File Share.

shareName: The name of the Azure File Share you created.

secretName: This points to the Kubernetes secret you created earlier.


Create the PersistentVolumeClaim (PVC)
A PersistentVolumeClaim (PVC) is a request for storage by a Kubernetes pod. Here’s an example for local SMB share and Azure file share.
For Local SMB Share
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: smb-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

For Azure File Share
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-file-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

Apply the PV and PVC
You should see the STATUS as Bound for both PersistentVolume and PersistentVolumeClaim.
Mount the Volume in a Kubernetes Pod
```
apiVersion: v1
kind: Pod
metadata:
  name: test-smb-pod
spec:
  containers:
  - name: my-container
    image: nginx
    volumeMounts:
    - name: smb-volume
      mountPath: /mnt/smb
  volumes:
  - name: smb-volume
    persistentVolumeClaim:
      claimName: azure-file-pvc
```

Explanation:
mountPath: This is the path where the SMB/Azure share will be accessible inside the container.
volumeMounts: Attaches the volume (bound from the PVC) to the container.
s
Apply the Pod YAML
If everything works correctly, you should see the contents of the SMB share.

**If something happend:

Check your yaml twice and then:

The SMB file share is empty (this is normal if no files are uploaded).

You might want to create files inside the share to test persistence:

```
echo "Hello from Kubernetes!" > /mnt/smb/testfile.txt
Exit the pod and re-enter it to verify the file persists:
exit
```
The expectation was that /mnt/smb should be mounted as CIFS (SMB protocol), not ext4. This may indicate that the mount is not correctly pointing to your Azure File Share, and it might be referencing a local or cluster-based volume.

Check Your PVC and StorageClass
```
  volumeName: It should link to the correct PersistentVolume (PV).
  status.phase: Should be Bound.
```

How to remove Remove pod, pv,pvc,pod files