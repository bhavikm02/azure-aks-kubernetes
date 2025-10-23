# Ingress - SSL

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Cert-Manager Installation"
        InstallCertMgr[Install cert-manager<br/>via Helm Chart]
        InstallCertMgr --> CertMgrPods[cert-manager Pods:<br/>cert-manager<br/>cert-manager-webhook<br/>cert-manager-cainjector]
        InstallCertMgr --> CRDs[Custom Resource Definitions:<br/>Certificate<br/>Issuer<br/>ClusterIssuer]
    end
    
    subgraph "ClusterIssuer Configuration"
        ClusterIssuer[ClusterIssuer Resource<br/>letsencrypt-prod]
        ClusterIssuer --> ACMEServer[ACME Server:<br/>https://acme-v02.api.letsencrypt.org]
        ClusterIssuer --> Email[Email: your-email@domain.com<br/>For cert expiry notifications]
        ClusterIssuer --> HTTPChallenge[Challenge Type:<br/>HTTP-01 via Ingress]
    end
    
    subgraph "Application with SSL Ingress"
        AppDeploy[Application Deployment<br/>+ ClusterIP Service]
        
        IngressSSL[Ingress Resource<br/>with TLS config]
        IngressSSL --> TLSSection[tls:<br/>  hosts: app1.kubeoncloud.com<br/>  secretName: app1-secret-tls]
        IngressSSL --> Annotations[annotations:<br/>cert-manager.io/cluster-issuer:<br/>letsencrypt-prod]
    end
    
    subgraph "Certificate Issuance Flow"
        ApplyIngress[kubectl apply Ingress]
        ApplyIngress --> CertMgrDetects[cert-manager detects<br/>TLS annotation]
        CertMgrDetects --> CreateCertResource[Creates Certificate resource<br/>automatically]
        
        CreateCertResource --> RequestCert[Request certificate from<br/>Let's Encrypt]
        RequestCert --> ACMEChallenge[Let's Encrypt sends<br/>HTTP-01 challenge]
        ACMEChallenge --> TempIngress[cert-manager creates<br/>temporary Ingress rule]
        TempIngress --> ChallengePath[/.well-known/acme-challenge/<br/>token]
        
        ACMEChallenge --> LetsEncryptVerify[Let's Encrypt verifies:<br/>http://app1.kubeoncloud.com<br/>/.well-known/acme-challenge/token]
        LetsEncryptVerify --> ChallengeSuccess{Challenge<br/>Success?}
        ChallengeSuccess -->|Yes| IssueCert[Issue SSL Certificate]
        ChallengeSuccess -->|No| RetryChallenge[Retry or fail]
        
        IssueCert --> StoreCert[Store cert in Secret:<br/>app1-secret-tls]
        StoreCert --> IngressUsesSecret[Ingress uses Secret<br/>for TLS termination]
    end
    
    subgraph "HTTPS Traffic Flow"
        UserHTTPS[User: https://app1.kubeoncloud.com]
        UserHTTPS --> TLSHandshake[TLS Handshake]
        TLSHandshake --> IngressController[NGINX Ingress Controller<br/>presents certificate]
        IngressController --> ValidateCert[Browser validates cert:<br/>✓ Trusted CA Let's Encrypt<br/>✓ Domain matches<br/>✓ Not expired]
        ValidateCert --> SecureConnection[Encrypted Connection<br/>Established]
        SecureConnection --> RouteApp[Route to backend app]
        RouteApp --> AppDeploy
    end
    
    subgraph "Certificate Renewal"
        CertExpiry[Certificate Valid: 90 days]
        CertExpiry --> AutoRenew[cert-manager auto-renews<br/>at 30 days before expiry]
        AutoRenew --> NewChallenge[New HTTP-01 challenge]
        NewChallenge --> UpdateSecret[Update Secret with<br/>new certificate]
        UpdateSecret --> ZeroDowntime[Zero downtime renewal]
    end
    
    subgraph "SSL Redirect"
        HTTPRequest[User: http://app1.kubeoncloud.com]
        HTTPRequest --> HTTPIngress[Ingress annotation:<br/>nginx.ingress.kubernetes.io/<br/>ssl-redirect: "true"]
        HTTPIngress --> Redirect301[HTTP 301 Redirect]
        Redirect301 --> RedirectHTTPS[→ https://app1.kubeoncloud.com]
    end
    
    style CertMgrDetects fill:#9370db
    style IssueCert fill:#28a745
    style SecureConnection fill:#28a745
    style IngressController fill:#326ce5
```

### Understanding the Diagram

- **cert-manager**: Kubernetes add-on that **automates SSL/TLS certificate management**, including issuance, renewal, and storage
- **ClusterIssuer**: Cluster-wide resource defining how to obtain certificates from **Let's Encrypt ACME server** (free SSL certificates)
- **Let's Encrypt**: Free, automated, and open **Certificate Authority** providing SSL/TLS certificates trusted by all major browsers
- **HTTP-01 Challenge**: Let's Encrypt verifies domain ownership by requesting a file at **/.well-known/acme-challenge/** via HTTP
- **Automatic Certificate Creation**: cert-manager watches Ingress resources with TLS config and **automatically requests certificates** from Let's Encrypt
- **Certificate Storage**: Issued certificates are stored in Kubernetes **Secrets** (type: kubernetes.io/tls), containing cert and private key
- **TLS Termination**: NGINX Ingress Controller **terminates TLS** (decrypts HTTPS), then forwards unencrypted traffic to backend Pods
- **90-Day Validity**: Let's Encrypt certificates are valid for **90 days**, but cert-manager **auto-renews at 60 days**, ensuring continuous protection
- **Zero-Downtime Renewal**: Certificate renewal happens **automatically in the background** without service interruption or manual intervention
- **SSL Redirect**: Annotation **ssl-redirect: "true"** automatically redirects HTTP traffic to HTTPS, ensuring all traffic is encrypted

---

## Step-01: Introduction
- Implement SSL using Lets Encrypt

[![Image](https://www.stacksimplify.com/course-images/azure-aks-ingress-ssl-letsencrypt.png "Azure AKS Kubernetes - Masterclass")](https://www.udemy.com/course/aws-eks-kubernetes-masterclass-devops-microservices/?referralCode=257C9AD5B5AF8D12D1E1)

## Step-02: Install Cert Manager
```t
# Label the ingress-basic namespace to disable resource validation
kubectl label namespace ingress-basic cert-manager.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace ingress-basic \
  --version v1.13.3 \
  --set installCRDs=true

## SAMPLE OUTPUT
Kalyans-Mac-mini:azure-aks-kubernetes-masterclass-internal24 kalyanreddy$ helm install \
>   cert-manager jetstack/cert-manager \
>   --namespace ingress-basic \
>   --version v1.13.3 \
>   --set installCRDs=true
NAME: cert-manager
LAST DEPLOYED: Fri Dec 29 12:47:27 2023
NAMESPACE: ingress-basic
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.13.3 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
Kalyans-Mac-mini:azure-aks-kubernetes-masterclass-internal24 kalyanreddy$ 

# Verify Cert Manager pods
kubectl get pods --namespace ingress-basic

# Verify Cert Manager Services
kubectl get svc --namespace ingress-basic
```

## Step-06: Review or Create Cluster Issuer Kubernetes Manifest
### Review Cluster Issuer Kubernetes Manifest
- Create or Review Cert Manager Cluster Issuer Kubernetes Manigest
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: dkalyanreddy@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx            
```

### Deploy Cluster Issuer
```t
# Deploy Cluster Issuer
kubectl apply -f kube-manifests/01-CertManager-ClusterIssuer/cluster-issuer.yml

# List Cluster Issuer
kubectl get clusterissuer

# Describe Cluster Issuer
kubectl describe clusterissuer letsencrypt
```


## Step-07: Review Application NginxApp1,2 K8S Manifests
- 01-NginxApp1-Deployment.yml
- 02-NginxApp1-ClusterIP-Service.yml
- 01-NginxApp2-Deployment.yml
- 02-NginxApp2-ClusterIP-Service.yml

## Step-08: Create or Review Ingress SSL Kubernetes Manifest
- 01-Ingress-SSL.yml

## Step-09: Deploy All Manifests & Verify
- Certificate Request, Generation, Approal and Download and be ready might take from 1 hour to couple of days if we make any mistakes and also fail.
- For me it took, only 5 minutes to get the certificate from **https://letsencrypt.org/**
```t
# Deploy
kubectl apply -R -f kube-manifests/

# Verify Pods
kubectl get pods

# Verify Cert Manager Pod Logs
kubectl get pods -n ingress-basic
kubectl -n ingress-basic logs -f <cert-manager-55d65894c7-sx62f> -n ingress-basic #Replace Pod name

# Verify SSL Certificates (It should turn to True)
kubectl get certificate
```
```log
stack@Azure:~$ kubectl get certificate
NAME                      READY   SECRET                    AGE
app1-kubeoncloud-secret   True    app1-kubeoncloud-secret   45m
app2-kubeoncloud-secret   True    app2-kubeoncloud-secret   45m
stack@Azure:~$
```

```log
# Sample Success Log
I0824 13:09:00.495721       1 controller.go:129] cert-manager/controller/orders "msg"="syncing item" "key"="default/app2-kubeoncloud-secret-2792049964-67728538" 
I0824 13:09:00.495900       1 sync.go:102] cert-manager/controller/orders "msg"="Order has already been completed, cleaning up any owned Challenge resources" "resource_kind"="Order" "resource_name"="app2-kubeoncloud-secret-2792049964-67728538" "resource_namespace"="default" 
I0824 13:09:00.496904       1 controller.go:135] cert-manager/controller/orders "msg"="finished processing work item" "key"="default/app2-kubeoncloud-secret-2792049964-67728538
```

## Step-10: Access Application
```t
# URLs
http://sapp1.kubeoncloud.com/app1/index.html
http://sapp2.kubeoncloud.com/app2/index.html
```

## Step-11: Verify Ingress logs for Client IP
```t
# List Pods
kubectl -n ingress-basic get pods

# Check logs
kubectl -n ingress-basic logs -f nginx-ingress-controller-xxxxxxxxx
```
## Step-12: Clean-Up
```t
# Delete Apps
kubectl delete -R -f kube-manifests/

# Delete Ingress Controller
kubectl delete namespace ingress-basic
```

## Cert Manager
- https://cert-manager.io/docs/installation/#default-static-install
- https://cert-manager.io/docs/installation/helm/
- https://docs.cert-manager.io/
- https://cert-manager.io/docs/installation/helm/#1-add-the-jetstack-helm-repository
- https://cert-manager.io/docs/configuration/
- https://cert-manager.io/docs/tutorials/acme/nginx-ingress/#step-6---configure-a-lets-encrypt-issuer
- https://letsencrypt.org/how-it-works/

  