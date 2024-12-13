# Create SQL Server storage provider
Set up any of the following SQL Server editions:
•	SQL Server on premises

•	Azure SQL Database 
https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview

•	Azure SQL Managed Instance
https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/sql-managed-instance-paas-overview

•	SQL Server enabled by Azure Arc
https://learn.microsoft.com/en-us/sql/sql-server/azure-arc/overview


CREATING LOCAL DATABASE 

Prerequisites
•	Docker Desktop installed and running.

•	Kubernetes enabled on Docker Desktop.

•	Sufficient system resources (2 CPUs, 4GB RAM) to run SQL Server in a container.

# How to Run SQL Server in Kubernetes?

If you want SQL Server inside Kubernetes, you can create it using the following YAML.

1. Create a SQL Server Deployment in Kubernetes
Create a file named sqlserver-deployment.yaml with the following content:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sqlserver-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sqlserver
  template:
    metadata:
      labels:
        app: sqlserver
    spec:
      containers:
      - name: sqlserver
        image: mcr.microsoft.com/mssql/server:2019-latest
        ports:
        - containerPort: 1433
        env:
        - name: ACCEPT_EULA
          value: "Y"
        - name: SA_PASSWORD
          value: "YourStrong!Passw0rd"
        volumeMounts:
        - mountPath: /mnt/smb
          name: smb-volume
      volumes:
      - name: smb-volume
        persistentVolumeClaim:
          claimName: azure-file-pvc
```
2. Apply the YAML to Kubernetes
```
kubectl apply -f sqlserver-deployment.yaml
```
Wait for the pod to be created. You can check the status with:
```
kubectl get pods
```
Look for a pod name that looks like sqlserver-deployment-XXXXXX.

3. Connect to SQL Server Pod
Once you have the pod name, connect to it like this:

How to Verify the SQL Server Connection?
```
1> SELECT @@VERSION; 
2> GO
exit
```
And now
1.	Open SQL Server Management Studio (SSMS).

2.	Connect to SQL Server:

o	Server name: localhost,1433

o	Authentication: SQL Server Authentication

o	Login: sa

o	Password: YourStrong!Passw0rd

3.	Click Connect.


# Create a Database
After connecting to SQL Server, you can create a new database.

Option 1: Use SSMS or Azure Data Studio

1.	Open SQL Server Management Studio (SSMS) or Azure Data Studio.

2.	Right-click Databases in the Object Explorer.

3.	Select New Database.

4.	Enter DATABASENAME as the name of the new database.

5.	Click OK.


# Common Issues and Fixes

| **Issue**                        | **Solution**                                           |
|-----------------------------------|-------------------------------------------------------|
| Port 1433 is already in use       | Stop the existing process using port 1433, or change to `-p 1434:1433`. |
| Password does not meet requirements | Use a stronger password (e.g., `YourStrong!Passw0rd`). |


This means that the SQL Server pod has access to the SMB share and can read/write files to/from it.
Your SQL Server and SMB connection is successful. 🎉🎉🎉