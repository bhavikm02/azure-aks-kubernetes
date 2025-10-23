# User Azure Database for MySQL Flexible for AKS Workloads

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Problems with MySQL Pod + Azure Disk"
        Problems[Issues with Pod-based MySQL]
        Problems --> P1[Single Point of Failure<br/>Pod crash = Downtime]
        Problems --> P2[Manual Scaling<br/>No auto-scaling]
        Problems --> P3[Manual Backups<br/>Complex setup]
        Problems --> P4[Limited HA<br/>No multi-zone replica]
        Problems --> P5[Maintenance Overhead<br/>Patching, updates]
    end
    
    subgraph "Azure MySQL Flexible Server Solution"
        AzureMySQL[Azure Database for MySQL<br/>Flexible Server]
        AzureMySQL --> Feature1[Built-in High Availability<br/>Zone-redundant]
        AzureMySQL --> Feature2[Automatic Backups<br/>Point-in-time restore]
        AzureMySQL --> Feature3[Auto-scaling<br/>Compute & storage]
        AzureMySQL --> Feature4[Managed Patching<br/>Azure handles updates]
        AzureMySQL --> Feature5[Enterprise Security<br/>SSL, Firewall, VNet]
    end
    
    subgraph "Azure MySQL Setup"
        CreateDB[Create MySQL Flexible Server<br/>akswebappdb201.mysql.database.azure.com]
        CreateDB --> DBConfig[Configuration:<br/>MySQL 8.0<br/>dbadmin user<br/>Public access enabled]
        DBConfig --> FirewallRule[Firewall Rule:<br/>Allow Azure services]
        DBConfig --> SecurityParam[require_secure_transport: OFF<br/>For development]
        
        FirewallRule --> CreateSchema[Create Database:<br/>webappdb]
    end
    
    subgraph "Kubernetes Integration via ExternalName"
        ExternalNameSvc[01-MySQL-externalName-Service.yml<br/>kind: Service<br/>type: ExternalName]
        
        ExternalNameSvc --> SvcSpec[spec:<br/>  type: ExternalName<br/>  externalName:<br/>    akswebappdb201.mysql.database.azure.com]
        
        SvcSpec --> DNS[Kubernetes DNS<br/>mysql → External FQDN]
    end
    
    subgraph "Web App Deployment"
        WebAppDeploy[02-UserMgmtWebApp-Deployment.yml<br/>Spring Boot Application]
        
        WebAppDeploy --> WebAppEnv[Environment Variables:<br/>DB_HOSTNAME: mysql<br/>DB_USERNAME: dbadmin<br/>DB_PASSWORD: Redhat1449<br/>DB_NAME: webappdb]
        
        WebAppEnv --> ConnectDB[App connects to "mysql"<br/>Resolves to Azure MySQL FQDN]
        ConnectDB --> DNS
        DNS --> AzureMySQL
        
        WebAppDeploy --> LBSvc[LoadBalancer Service<br/>External Access]
    end
    
    subgraph "Traffic Flow"
        User[Internet User] --> WebAppLB[Load Balancer<br/>Public IP]
        WebAppLB --> WebAppPod[Web App Pod]
        WebAppPod --> ExternalNameSvc
        ExternalNameSvc --> InternetEgress[Internet Egress<br/>from AKS]
        InternetEgress --> AzureMySQL
        AzureMySQL --> QueryResponse[Query Response]
    end
    
    subgraph "ExternalName Service Benefits"
        Benefits[Why ExternalName?]
        Benefits --> B1[No code changes<br/>Still uses "mysql" hostname]
        Benefits --> B2[Easy migration<br/>Pod MySQL → Azure MySQL]
        Benefits --> B3[Service abstraction<br/>Change backend easily]
        Benefits --> B4[Consistent naming<br/>Dev, staging, prod]
    end
    
    style Problems fill:#ff6b6b
    style AzureMySQL fill:#0078d4
    style ExternalNameSvc fill:#9370db
    style Benefits fill:#28a745
```

### Understanding the Diagram

- **Pod MySQL Limitations**: Running MySQL as a **Pod with Azure Disk** lacks **built-in HA**, **automatic backups**, and requires **manual operational overhead**
- **Azure MySQL Flexible Server**: **Fully managed** database service with **automatic backups**, **high availability**, **security**, and **auto-scaling** capabilities
- **Zone-Redundant HA**: Azure MySQL can deploy **replicas across availability zones** for automatic failover during zone failures
- **Firewall Configuration**: Setting **Allow Azure services** permits AKS Pods to connect to MySQL without exposing database to entire internet
- **require_secure_transport OFF**: Disables SSL requirement for **development/testing**; **enable SSL in production** for encrypted connections
- **ExternalName Service**: Special service type that creates a **DNS CNAME record** mapping internal service name to external FQDN
- **No IP Address**: ExternalName Services have **no ClusterIP**; they return the external FQDN directly via Kubernetes DNS
- **Seamless Migration**: Applications continue using **"mysql" hostname**; Kubernetes DNS resolves it to **Azure MySQL FQDN** automatically
- **Environment Variable Update**: Change **DB_USERNAME** from `root` to `dbadmin` (Azure MySQL admin user) and update **DB_PASSWORD** accordingly
- **Cost vs Capability Trade-off**: Azure MySQL **costs more** than Pod + Disk but provides **enterprise features**, **reduced ops burden**, and **better reliability**

---

## Step-01: Introduction
- What are the problems with MySQL Pod & Azure Disks? 
- How we are going to solve them using Azure Database for MySQL Flexible?

## Step-02: Create Azure Database for MySQL flexible servers
- Go to Service **Azure Database for MySQL flexible servers**
- Click on **Create** -> Flexible Server
- **Basics**
- **Project details**
  - Subscription: SUBSCRIPTION-NAME
  - Resource Group: aks-rg1
- **Server Details**
  - Server name: akswebappdb201 (This name is based on availability - in your case it might be something else)
  - Region: (US) East US
  - MySQL Version: 8.0 (default)
  - Workload type: For development or hobby projects
  - **Compute + Storage:** Leave to defaults
  - Availability Zone: No prederence
- **High availability** 
  - leave to defaults (NO HA)  
- **Authentication**     
  - Authentication method: mysql authentication only
  - Admin username: dbadmin
  - Password: Redhat1449
  - Confirm password: Redhat1449
  - Click on **Next Networking**
- **Network connectivity**
  - Connectivity method: Public access (allowed IP addresses) and Private endpoint
  - Public access: CHECKED
  - Firewall rules: 
    - Allow public access from any Azure service within Azure to this server: CHECKED
  - Private endpoint: DONT DEFINE 
  - Click on **Next Security**
- **Security**
  - Leave to defaults        
- **Tags**
  - Leave to defaults        
- **Review + Create**  
- It will take close to 15 minutes to create the database. 

## Step-03: Update Security Settings for Database
- Go to **Azure Database for MySQL flexible servers** -> **akswebappdb201**
- **Settings -> Server Parameters**
  - Change **require_secure_transport: OFF**
  - Click on **Save**
- It will take close to 5 to 10 minutes for changes to take place. 


## Step-04:  Connect to Azure MySQL Database 
### Step-04-01: Using Azure Cloud Shell
- As we enabled the setting **Allow public access from any Azure service within Azure to this server** in `Networking Tab` it does the below. 
- This option configures the firewall to allow connections from IP addresses allocated to any Azure service or asset, including connections from the subscriptions of other customers.
- Go to Azure Database for MySQL flexible servers** -> akswebappdb201 -> Settings -> **Connect**
```t
# DB Connect Command
mysql -h akswebappdb201.mysql.database.azure.com -u dbadmin -p

mysql> show schemas;
mysql> create database webappdb;
mysql> show schemas;
mysql> exit
```
### Step-04-02: Using kubectl and create usermgmt schema/db
```t
# Template
kubectl run -it --rm --image=mysql:8.0 --restart=Never mysql-client -- mysql -h <AZURE-MYSQ-DB-HOSTNAME> -u <USER_NAME> -p<PASSWORD>

# Replace Host Name of Azure MySQL Database and Username and Password
kubectl run -it --rm --image=mysql:8.0 --restart=Never mysql-client -- mysql -h akswebappdb201.mysql.database.azure.com -u dbadmin -pRedhat1449

mysql> show schemas;
mysql> create database webappdb;
mysql> show schemas;
mysql> exit
```

## Step-05: Update Kubernetes Manifests
### Step-05-01: Create Kubernetes externalName service Manifest and Deploy
- Create mysql externalName Service
- **01-MySQL-externalName-Service.yml**
```yml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  type: ExternalName
  externalName: akswebappdb201.mysql.database.azure.com
```

### Step-05-02: In User Management WebApp deployment file change username from `root` to 
`dbadmin`
- **02-UserMgmtWebApp-Deployment.yml**
```yml
# Change From
          - name: DB_USERNAME
            value: "root"
          - name: DB_PASSWORD
            value: "dbpassword11"               

# Change To dbadmin
            - name: DB_USERNAME
              value: "dbadmin"            
            - name: DB_PASSWORD
              value: "Redhat1449"                              
```

## Step-06: Deploy User Management WebApp and Test
```t
# Deploy all Manifests
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Stream pod logs to verify DB Connection is successful from SpringBoot Application
kubectl logs -f <pod-name>
```
## Step-07: Access Application
```t
# Get Public IP
kubectl get svc

# Access Application
http://<External-IP-from-get-service-output>
Username: admin101
Password: password101
```

## Step-08: Clean Up 
```t
# Delete all Objects created
kubectl delete -f kube-manifests/

# Verify current Kubernetes Objects
kubectl get all

# Delete Azure MySQL Database
- Go to Azure Database for MySQL flexible servers -> akswebappdb201 -> Delete
```
