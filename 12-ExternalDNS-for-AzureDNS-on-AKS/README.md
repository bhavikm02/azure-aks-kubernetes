# Kubernetes ExternalDNS to create Record Sets in Azure DNS from AKS

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "ExternalDNS Setup"
        MSI[Azure Managed Service Identity<br/>Created for AKS VMSS]
        MSI --> Permissions[Assign Permissions:<br/>DNS Zone Contributor<br/>on dns-zones resource group]
        
        ConfigFile[azure.json ConfigMap<br/>tenantId, subscriptionId<br/>resourceGroup: dns-zones<br/>useManagedIdentityExtension: true]
        
        ExternalDNSDeploy[ExternalDNS Deployment<br/>bitnami/external-dns image]
        ExternalDNSDeploy --> ServiceAccount[ServiceAccount: external-dns<br/>with RBAC permissions]
        ServiceAccount --> ClusterRole[ClusterRole: Read Services<br/>Ingress, Nodes]
    end
    
    subgraph "Application Deployment"
        App[Application Deployment<br/>with Service or Ingress]
        App --> Annotation[Annotation:<br/>external-dns.alpha.kubernetes.io/hostname<br/>= app1.kubeoncloud.com]
    end
    
    subgraph "ExternalDNS Workflow"
        ExternalDNSPod[ExternalDNS Pod Running]
        ExternalDNSPod --> Watch[Watch Kubernetes API<br/>for Services & Ingress]
        
        Watch --> DetectAnnotation{Detect<br/>hostname<br/>annotation?}
        DetectAnnotation -->|Yes| ExtractHostname[Extract hostname:<br/>app1.kubeoncloud.com]
        DetectAnnotation -->|No| Skip[Skip - No DNS needed]
        
        ExtractHostname --> GetIP[Get Service/Ingress<br/>External IP:<br/>20.112.45.89]
        GetIP --> AzureDNSAPI[Call Azure DNS API<br/>using MSI credentials]
    end
    
    subgraph "Azure DNS Zone"
        AzureDNSAPI --> CheckRecord{DNS Record<br/>Exists?}
        CheckRecord -->|No| CreateRecord[Create A Record:<br/>app1.kubeoncloud.com → 20.112.45.89]
        CheckRecord -->|Yes - Different IP| UpdateRecord[Update A Record:<br/>to new IP]
        CheckRecord -->|Yes - Same IP| NoAction[No action needed]
        
        CreateRecord --> DNSZone[Azure DNS Zone:<br/>kubeoncloud.com]
        UpdateRecord --> DNSZone
        
        DNSZone --> RecordSets[Record Sets:<br/>A: app1.kubeoncloud.com<br/>A: app2.kubeoncloud.com<br/>A: usermgmt.kubeoncloud.com]
    end
    
    subgraph "DNS Resolution"
        User[Internet User] --> DNSQuery[Query: app1.kubeoncloud.com]
        DNSQuery --> AzureNS[Azure Nameservers]
        AzureNS --> DNSZone
        DNSZone --> ReturnIP[Return IP: 20.112.45.89]
        ReturnIP --> UserAccess[User accesses application]
    end
    
    subgraph "Service Lifecycle Management"
        Lifecycle[Lifecycle Events]
        Lifecycle --> ServiceCreated[Service Created<br/>→ Create DNS Record]
        Lifecycle --> ServiceUpdated[Service IP Changed<br/>→ Update DNS Record]
        Lifecycle --> ServiceDeleted[Service Deleted<br/>→ Delete DNS Record]
    end
    
    subgraph "MSI vs Service Principal"
        Comparison[Authentication Methods]
        Comparison --> MSIMethod[MSI Managed Identity:<br/>✓ No password management<br/>✓ Automatic credential rotation<br/>✓ Azure-native security<br/>✓ Recommended approach]
        Comparison --> SPMethod[Service Principal:<br/>✓ Manual credential management<br/>✗ Password/secret expiration<br/>✗ Security overhead]
    end
    
    style ExternalDNSPod fill:#326ce5
    style DNSZone fill:#0078d4
    style CreateRecord fill:#28a745
    style MSIMethod fill:#28a745
```

### Understanding the Diagram

- **ExternalDNS Purpose**: Automatically creates, updates, and deletes **DNS records** in Azure DNS based on **Kubernetes Services** and **Ingress resources**
- **Managed Service Identity (MSI)**: Azure-native authentication method that eliminates **password management** and provides **automatic credential rotation**
- **DNS Zone Contributor**: ExternalDNS needs **DNS Zone Contributor** role on the resource group containing Azure DNS Zones to manage records
- **Hostname Annotation**: Add **external-dns.alpha.kubernetes.io/hostname** annotation to Services/Ingress to trigger automatic DNS record creation
- **Watch & Sync**: ExternalDNS **continuously watches** Kubernetes API for annotated Services/Ingress and **synchronizes** with Azure DNS
- **Automatic Record Management**: When Service gets external IP, ExternalDNS **creates A record**; when IP changes, it **updates**; when deleted, it **removes** the record
- **azure.json ConfigMap**: Contains Azure **tenant ID, subscription ID, resource group** and MSI configuration for Azure API authentication
- **RBAC Permissions**: ServiceAccount with **ClusterRole** allows ExternalDNS to read Services, Ingress, Nodes from Kubernetes API
- **Lifecycle Automation**: Handles complete **DNS lifecycle** - no manual DNS record management needed for Kubernetes workloads
- **Multiple Records**: Single ExternalDNS deployment manages **all annotated Services** across all namespaces, creating multiple DNS records as needed

---

## Step-01: Introduction
- Create External DNS Manifest
- Provide Access to DNZ Zones using **Azure Managed Service Identity** for External DNS pod to create **Record Sets** in Azure DNS Zones
- Review Application & Ingress Manifests
- Deploy and Test
[![Image](https://www.stacksimplify.com/course-images/azure-aks-ingress-external-dns.png "Azure AKS Kubernetes - Masterclass")](https://www.udemy.com/course/aws-eks-kubernetes-masterclass-devops-microservices/?referralCode=257C9AD5B5AF8D12D1E1)

## Step-02: Create External DNS Manifests
- External-DNS needs permissions to Azure DNS to modify (Add, Update, Delete DNS Record Sets)
- We can provide permissions to External-DNS pod in two ways in Azure 
  - Using Azure Service Principal
  - Using Azure Managed Service Identity (MSI)
- We are going to use `MSI` for providing necessary permissions here which is latest and greatest in Azure as on today. 


### Gather Information Required for azure.json file
```t
# To get Azure Tenant ID
az account show --query "tenantId"

# To get Azure Subscription ID
az account show --query "id"
```

### Create azure.json file
```json
{
  "tenantId": "c81f465b-99f9-42d3-a169-8082d61c677a",
  "subscriptionId": "82808767-144c-4c66-a320-b30791668b0a",
  "resourceGroup": "dns-zones", 
  "useManagedIdentityExtension": true,
  "userAssignedIdentityID": "404b0cc1-ba04-4933-bcea-7d002d184436"  
}
```

### Review external-dns.yml manifest
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods", "nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.0
          args:
            - --source=service
            - --source=ingress
            #- --domain-filter=example.com # (optional) limit to only example.com domains; change to match the zone created above.
            - --provider=azure
            #- --azure-resource-group=MyDnsResourceGroup # (optional) use the DNS zones from the tutorial's resource group
            - --txt-prefix=externaldns-
          volumeMounts:
            - name: azure-config-file
              mountPath: /etc/kubernetes
              readOnly: true
      volumes:
        - name: azure-config-file
          secret:
            secretName: azure-config-file
```

## Step-03: Create MSI - Managed Service Identity for External DNS to access Azure DNS Zones

### Create Manged Service Identity (MSI)
- Go to All Services -> Managed Identities -> Add
- Resource Name: aksdemo1-externaldns-access-to-dnszones
- Subscription: Pay-as-you-go
- Resource group: aks-rg1
- Location: Central US
- Click on **Create**

### Add Azure Role Assignment in MSI
- Opem MSI -> aksdemo1-externaldns-access-to-dnszones 
- Click on **Azure Role Assignments** -> **Add role assignment**
- Scope: Resource group
- Subscription: Pay-as-you-go
- Resource group: dns-zones
- Role: Contributor

### Make a note of Client Id and update in azure.json
- Go to **Overview** -> Make a note of **Client ID"
- Update in **azure.json** value for **userAssignedIdentityID**
```
  "userAssignedIdentityID": "de836e14-b1ba-467b-aec2-93f31c027ab7"
```

## Step-04: Associate MSI in AKS Cluster VMSS
- Go to All Services -> Virtual Machine Scale Sets (VMSS) -> Open aksdemo1 related VMSS (aks-agentpool-27193923-vmss)
- Go to Settings -> Identity -> User assigned -> Add -> aksdemo1-externaldns-access-to-dnszones 



## Step-05: Create Kubernetes Secret and Deploy ExternalDNS
```t
# Create Secret
cd kube-manifests/01-ExteranlDNS
kubectl create secret generic azure-config-file --from-file=azure.json

# List Secrets
kubectl get secrets

# Deploy ExternalDNS 
cd kube-manifests/01-ExteranlDNS
kubectl apply -f external-dns.yml

# Verify ExternalDNS Logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

```log
# Error Type: 400
time="2020-08-24T11:25:04Z" level=error msg="azure.BearerAuthorizer#WithAuthorization: Failed to refresh the Token for request to https://management.azure.com/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/dns-zones/providers/Microsoft.Network/dnsZones?api-version=2018-05-01: StatusCode=400 -- Original Error: adal: Refresh request failed. Status Code = '400'. Response body: {\"error\":\"invalid_request\",\"error_description\":\"Identity not found\"}"

# Error Type: 403
Notes: Error 403 will come when our Managed Service Identity dont have access to respective destination resource 

# When all good, we should get log as below
time="2020-08-24T11:27:59Z" level=info msg="Resolving to user assigned identity, client id is 404b0cc1-ba04-4933-bcea-7d002d184436."
```


## Step-06: Deploy Application and Test
- When dns record set got created in DNS Zone, the log in external-dns should look as below.

### Deploy Application
```t
# Deploy Application
kubectl apply -f kube-manifests/02-NginxApp1

# Verify Pods and Services
kubectl get po,svc

# Verify Ingress
kubectl get ingress
```

### Verify logs in External DNS Pod
- Wait for 3 to 5 minutes for Record Set update in DNZ Zones
```t
# Verify ExternalDNS Logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```
- External DNS Pod Logs
```log
time="2020-08-24T11:30:54Z" level=info msg="Updating A record named 'eapp1' to '20.37.141.33' for Azure DNS zone 'kubeoncloud.com'."
time="2020-08-24T11:30:55Z" level=info msg="Updating TXT record named 'eapp1' to '\"heritage=external-dns,external-dns/owner=default,external-dns/resource=ingress/default/nginxapp1-ingress-service\"' for Azure DNS zone 'kubeoncloud.com'."
```

### Verify Record Set in DNS Zones -> kubeoncloud.com
- Go to All Services -> DNS Zones -> kubeoncloud.com
- Verify if we have `eapp1.kubeoncloud.com` created
```t
# Template Command
az network dns record-set a list -g <Resource-Group-dnz-zones> -z <yourdomain.com>

# Replace DNS Zones Resource Group and yourdomain
az network dns record-set a list -g dns-zones -z kubeoncloud.com
```
- Perform `nslookup` test
```t
# nslookup Test
Kalyans-MacBook-Pro:01-ExternalDNS kdaida$ nslookup eapp1.kubeoncloud.com
Server:		192.168.0.1
Address:	192.168.0.1#53

Non-authoritative answer:
Name:	eapp1.kubeoncloud.com
Address: 20.37.141.33

Kalyans-MacBook-Pro:01-ExternalDNS kdaida$ 
```

### Access Application and Test
```t
# Access Application
http://eapp1.kubeoncloud.com
http://eapp1.kubeoncloud.com/app1/index.html

# Note: Replace kubeoncloud.com with your domain name
```

## Step-07: Clean-Up
```t
# Delete Application
kubectl delete -f kube-manifests/02-NginxApp1

# Verify External DNS pod to ensure record set got deleted
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')

# Verify Record set got automatically deleted in DNS Zones
# Template Command
az network dns record-set a list -g <Resource-Group-dnz-zones> -z <yourdomain.com>

# Replace DNS Zones Resource Group and yourdomain
az network dns record-set a list -g dns-zones -z kubeoncloud.com
```

```log
time="2020-08-24T12:08:52Z" level=info msg="Deleting A record named 'eapp1' for Azure DNS zone 'kubeoncloud.com'."
time="2020-08-24T12:08:53Z" level=info msg="Deleting TXT record named 'eapp1' for Azure DNS zone 'kubeoncloud.com'."
```

## References
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md
- Open Issue and Break fix: https://github.com/kubernetes-sigs/external-dns/issues/1548
- https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx#configuration
- https://github.com/kubernetes/ingress-nginx/blob/master/charts/ingress-nginx/values.yaml
- https://kubernetes.github.io/ingress-nginx/

## External DNS References
- https://github.com/kubernetes-sigs/external-dns
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md