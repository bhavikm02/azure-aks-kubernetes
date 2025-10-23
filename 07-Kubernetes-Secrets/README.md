# Kubernetes - Secrets

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Problem: Hardcoded Passwords"
        Problem[Hardcoded Passwords in YAML]
        Problem --> P1[env:<br/>- MYSQL_ROOT_PASSWORD: "dbpassword11"<br/>❌ Password visible in Git]
        Problem --> P2[Security Risk<br/>Anyone with repo access<br/>sees passwords]
    end
    
    subgraph "Solution: Kubernetes Secrets"
        Solution[Kubernetes Secrets]
        Solution --> CreateSecret[Create Secret:<br/>echo -n 'dbpassword11' | base64<br/>Output: ZGJwYXNzd29yZDEx]
        
        CreateSecret --> SecretManifest[Secret Manifest<br/>apiVersion: v1<br/>kind: Secret<br/>type: Opaque]
        SecretManifest --> SecretData[data:<br/>  db-password: ZGJwYXNzd29yZDEx<br/>Base64 encoded]
    end
    
    subgraph "MySQL Deployment with Secret"
        MySQLDeploy[MySQL Deployment]
        MySQLDeploy --> EnvFrom[env:<br/>- name: MYSQL_ROOT_PASSWORD]
        EnvFrom --> ValueFrom[valueFrom:<br/>  secretKeyRef:<br/>    name: mysql-db-password<br/>    key: db-password]
        ValueFrom --> DecodeSecret[Kubernetes decodes<br/>base64 automatically]
        DecodeSecret --> ContainerEnv[Container gets:<br/>MYSQL_ROOT_PASSWORD=dbpassword11]
    end
    
    subgraph "Web App Deployment with Secret"
        WebAppDeploy[Web App Deployment]
        WebAppDeploy --> WebEnvFrom[env:<br/>- name: DB_PASSWORD]
        WebEnvFrom --> WebValueFrom[valueFrom:<br/>  secretKeyRef:<br/>    name: mysql-db-password<br/>    key: db-password]
        WebValueFrom --> WebDecodeSecret[Kubernetes decodes<br/>base64 automatically]
        WebDecodeSecret --> WebContainerEnv[Container gets:<br/>DB_PASSWORD=dbpassword11]
    end
    
    subgraph "Secret Benefits"
        Benefits[Why Use Secrets?]
        Benefits --> B1[Separation of Concerns<br/>Code separate from config]
        Benefits --> B2[Single Source of Truth<br/>One Secret, multiple consumers]
        Benefits --> B3[Git-Safe<br/>Base64 obfuscation]
        Benefits --> B4[Rotation Support<br/>Update Secret independently]
        Benefits --> B5[RBAC Integration<br/>Control who reads Secrets]
    end
    
    subgraph "Secret Types & Use Cases"
        Types[Secret Types]
        Types --> Opaque[Opaque: Generic key-value<br/>Passwords, tokens]
        Types --> TLS[kubernetes.io/tls<br/>TLS certificates & keys]
        Types --> DockerCfg[kubernetes.io/dockerconfigjson<br/>Docker registry credentials]
        Types --> ServiceAccount[kubernetes.io/service-account-token<br/>Service account tokens]
    end
    
    subgraph "Important Security Notes"
        Security[Security Considerations]
        Security --> S1[Base64 is NOT encryption<br/>Just encoding]
        Security --> S2[Secrets stored in etcd<br/>Enable encryption at rest]
        Security --> S3[Use RBAC<br/>Limit Secret access]
        Security --> S4[Consider External Secrets<br/>Azure Key Vault, HashiCorp Vault]
    end
    
    style Problem fill:#ff6b6b
    style Solution fill:#326ce5
    style Benefits fill:#28a745
    style Security fill:#ffa500
```

### Understanding the Diagram

- **Problem with Hardcoded Passwords**: Storing passwords directly in YAML manifests exposes them in **Git repositories**, **CI/CD logs**, and **kubectl outputs**
- **Kubernetes Secrets**: Store sensitive data as **base64-encoded** key-value pairs in a dedicated **Secret** resource type
- **Base64 Encoding**: Not encryption - just **obfuscation** to avoid accidental exposure; anyone with cluster access can decode Secrets
- **secretKeyRef**: References a specific **key** from a **named Secret**, allowing Kubernetes to inject the value as an environment variable
- **Automatic Decoding**: Kubernetes automatically **decodes base64** when injecting Secret values into containers as environment variables
- **Single Source of Truth**: Define password **once** in Secret, reference it from **multiple Deployments** (MySQL, Web App) for consistency
- **Secret Rotation**: Update Secret value **independently** without modifying Deployment manifests, enabling password rotation without redeployment
- **RBAC Integration**: Use **Role-Based Access Control** to restrict which service accounts and users can **read** or **modify** Secrets
- **Opaque Type**: Most common Secret type for **arbitrary key-value pairs** like passwords, API keys, tokens
- **Production Security**: Enable **etcd encryption at rest**, use **external secret managers** (Azure Key Vault, HashiCorp Vault), and implement **secret rotation policies**

---

## Step-01: Introduction
- Kubernetes Secrets let you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. 
- Storing confidential information in a Secret is safer and more flexible than putting it directly in a Pod definition or in a container image. 

## Step-02: Create Secret for MySQL DB Password
### 
```
# Mac
echo -n 'dbpassword11' | base64

# URL: https://www.base64encode.org
```
### Create Kubernetes Secrets manifest
```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-db-password
#type: Opaque means that from kubernetes's point of view the contents of this Secret is unstructured.
#It can contain arbitrary key-value pairs. 
type: Opaque
data:
  # Output of echo -n 'Redhat1449' | base64
  db-password: ZGJwYXNzd29yZDEx
```
## Step-03: Update secret in MySQL Deployment for DB Password
```yml
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-db-password
                  key: db-password
```

## Step-04: Update secret in UWA Deployment
- UMS means User Management Microservice
```yml
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-db-password
                  key: db-password
```

## Step-05: Create & Test
```
# Create All Objects
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Get Public IP of Application
kubectl get svc

# Access Application
http://<External-IP-from-get-service-output>
Username: admin101
Password: password101
```

## Step-06: Clean-Up
- Delete all k8s objects created as part of this section
```
# Delete All
kubectl delete -f kube-manifests/

# List Pods
kubectl get pods

# Verify sc, pvc, pv
kubectl get sc,pvc,pv
```