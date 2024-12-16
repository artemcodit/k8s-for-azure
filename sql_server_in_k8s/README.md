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
Create a file named SQLSERVER_FILE_NAME.yaml with the following content:
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
      securityContext:
        fsGroup: 10001
      initContainers:
      - name: volume-permissions
        image: alpine
        command: ["sh", "-c", "sleep 10 && ls -ld /var/mssql-data && chown -R 10001:10001 /var/mssql-data && chmod -R 0777 /var/mssql-data"]
        volumeMounts:
        - name: sqlserver-volume
          mountPath: /var/mssql-data
      containers:
      - name: sqlserver
        image: mcr.microsoft.com/mssql/server:2019-latest
        ports:
        - containerPort: 1433
        env:
        - name: ACCEPT_EULA
          value: "Y"
        - name: SA_PASSWORD
          value: "Password123!@"
        - name: MSSQL_DATA_DIR
          value: "/var/mssql-data/data"
        volumeMounts:
        - name: sqlserver-volume
          mountPath: /var/mssql-data
        securityContext:
          runAsUser: 10001
      volumes:
      - name: sqlserver-volume
        hostPath:
          path: /mnt/data/sqlserver
          type: DirectoryOrCreate
```

SQL_SERVICE_NAME.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: sqlserver-service
spec:
  selector:
    app: sqlserver
  type: NodePort
  ports:
  - protocol: TCP
    port: 1433
    targetPort: 1433
    nodePort: 31433
```



2. Apply the YAML to Kubernetes
```
kubectl apply -f sqlserver-deployment.yaml
kubectl get pods
```
Look for a pod name that looks like sqlserver-deployment-XXXXXX.


Check Kubernetes Node Availability
```
kubectl get nodes
```
If STATUS is NotReady, the Kubernetes node is not ready to run pods.
To debug it, run this command to check the node details:
```
kubectl describe node docker-desktop
```
Look for issues in Conditions or MemoryPressure or DiskPressure.
If there are any issues, restart Docker Desktop.




3. Connect to SQL Server Pod
Check pod logs to identify if we have any issues:

Check the permissions on /var/opt/mssql
```
kubectl exec -it $(kubectl get pods -l app=SQL_APP_NAME --output=jsonpath="{.items[0].metadata.name}") -- ls -ld /var/opt/mssql
```

```
kubectl describe pod $(kubectl get pods -l app=SQL_APP_NAME --output=jsonpath="{.items[0].metadata.name}")
```
Check SQL Logs
```
kubectl logs $(kubectl get pods -l app=SQL_APP_NAME --output=jsonpath="{.items[0].metadata.name}")
```

Look for this message:

" SQL Server is now ready for client connections. This is an informational message; no user action is required. "





If Everything Works
Once the pod is running, you can connect to SQL Server using sqlcmd like this:
```
kubectl exec -it $(kubectl get pods -l app=SQL_APP_NAME --output=jsonpath="{.items[0].metadata.name}") -- MOUNT_PATH_NAME -S localhost -U sa -P 'SQL_PASSWORD'
```
To connect to SQL Server and run queries, use sqlcmd.
```
kubectl exec -it $(kubectl get pods -l app=SQL_APP_NAME --output=jsonpath="{.items[0].metadata.name}") -- /bin/bash
```

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




If not work we need to configure some things.
Check SQL Server Pod IP
```
kubectl get pods -o wide -l app=sqlserver

Install telnet Inside sqlcmd-client

kubectl run sqlcmd-client --image=mcr.microsoft.com/mssql-tools -it --rm --restart=Never -- /bin/bash

apt-get update

apt-get install -y telnet

telnet <IP_POD_SQL> 1433
```
 
Since telnet worked, you should now be able to connect to SQL Server using sqlcmd.

kubectl run sqlcmd-client --image=mcr.microsoft.com/mssql-tools -it --rm --restart=Never -- /bin/bash

Run the sqlcmd command to connect to SQL Server:
```
/opt/mssql-tools/bin/sqlcmd -S 10.1.12.90 -U sa -P 'Password123!@' -Q 'SELECT @@VERSION;'
```


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