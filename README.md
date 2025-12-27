# .NET Application Kubernetes Deployment with GitOps

This repository contains a **complete production-grade Kubernetes setup** for deploying a **.NET application on Azure Kubernetes Service (AKS)** with **GitOps using Argo CD**.

Everything is structured for **one-click copy** and immediate deployment.

---

## 🎯 What This Setup Includes (Full Stack)

✔ Namespace isolation  
✔ ConfigMaps & Secrets  
✔ Deployment (.NET app)  
✔ Service  
✔ Ingress (Application Gateway / AGIC)  
✔ Autoscaling (HPA)  
✔ Blue-Green Deployment  
✔ Resource limits  
✔ Probes (liveness/readiness)  
✔ Network policies  
✔ GitOps-ready structure  

---

## 📐 Architecture

```
GitHub (Source of Truth)
  |
  v
Argo CD (Sync Engine)
  |
  v
AKS Cluster
  |
  ├── .NET Application
  ├── ConfigMaps & Secrets
  ├── HPA (Auto-scaling)
  ├── Network Policies
  └── Azure Application Gateway (Ingress)
```

---

## 📁 Repository Structure (Mandatory for GitOps)

```
k8s-infra/
├── base/
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── networkpolicy.yaml
│   └── kustomization.yaml
└── overlays/
    └── prod/
        └── kustomization.yaml
```

This is exactly how Argo CD expects it.

---

## 🚀 Quick Start

### Prerequisites
- Azure CLI installed and authenticated
- kubectl configured with AKS cluster access
- Argo CD installed on your cluster

### Deployment Steps

1. **Clone the repository**
```bash
git clone https://github.com/YOUR_USERNAME/dotnet-k8s-gitops
cd dotnet-k8s-gitops
```

2. **Apply manifests directly (without Argo CD)**
```bash
kubectl apply -k k8s-infra/base/
```

3. **OR deploy with Argo CD (recommended)**
```bash
kubectl apply -f argocd-application.yaml
```

4. **Verify deployment**
```bash
kubectl get pods -n prod
kubectl get svc -n prod
kubectl get ingress -n prod
```

---

## ☸️ Kubernetes Manifests

### 1️⃣ Namespace (Isolation)

**`k8s-infra/base/namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

---

### 2️⃣ ConfigMap (Non-Secret Config)

**`k8s-infra/base/configmap.yaml`**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dotnet-config
  namespace: prod
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Logging__LogLevel__Default: "Information"
```

---

### 3️⃣ Secret (Sensitive Data)

**`k8s-infra/base/secret.yaml`**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dotnet-secrets
  namespace: prod
type: Opaque
stringData:
  DB_CONNECTION: "Server=tcp:sql.database.windows.net;Database=db;"
  API_KEY: "your-api-key-here"
```

> ⚠️ **Security Note**: In production, use **Azure Key Vault CSI Driver** instead of storing secrets in Git.

---

### 4️⃣ Deployment (.NET App - Production Grade)

**`k8s-infra/base/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app
  namespace: prod
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: dotnet
  template:
    metadata:
      labels:
        app: dotnet
    spec:
      containers:
      - name: dotnet
        image: acrname.azurecr.io/dotnet-app:1.0.0
        ports:
        - containerPort: 80
        
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        
        envFrom:
        - configMapRef:
            name: dotnet-config
        - secretRef:
            name: dotnet-secrets
        
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
```

**Features:**
- ✔ Prevents bad pods from receiving traffic (readinessProbe)
- ✔ Auto-restarts unhealthy containers (livenessProbe)
- ✔ Rolling updates with zero downtime
- ✔ Resource limits for cost optimization

---

### 5️⃣ Service (Internal Load Balancer)

**`k8s-infra/base/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: dotnet-service
  namespace: prod
spec:
  type: ClusterIP
  selector:
    app: dotnet
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
```

---

### 6️⃣ Ingress (Application Gateway / AGIC)

**`k8s-infra/base/ingress.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dotnet-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: dotnet-service
            port:
              number: 80
```

**Features:**
- ✔ Uses Azure Application Gateway + WAF
- ✔ Enterprise-standard ingress
- ✔ SSL/TLS termination support

---

### 7️⃣ Autoscaling (HPA)

**`k8s-infra/base/hpa.yaml`**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: dotnet-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dotnet-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**Features:**
- ✔ Auto-scales pods based on CPU/Memory
- ✔ Works with AKS Cluster Autoscaler
- ✔ Handles traffic spikes automatically

---

### 8️⃣ Network Policy (Security)

**`k8s-infra/base/networkpolicy.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: dotnet
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 80
```

**Features:**
- ✔ Blocks unwanted pod-to-pod traffic
- ✔ Allows only ingress traffic on port 80
- ✔ Enhances cluster security

---

### 9️⃣ Kustomization (GitOps Entry Point)

**`k8s-infra/base/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- namespace.yaml
- configmap.yaml
- secret.yaml
- deployment.yaml
- service.yaml
- ingress.yaml
- hpa.yaml
- networkpolicy.yaml

commonLabels:
  app: dotnet
  environment: production
```

**`k8s-infra/overlays/prod/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

namespace: prod

patchesStrategicMerge:
- |-
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: dotnet-app
  spec:
    replicas: 3
```

---

## 🔵🟢 Blue-Green Deployment Strategy

### Blue Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app-blue
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dotnet
      version: blue
  template:
    metadata:
      labels:
        app: dotnet
        version: blue
    spec:
      containers:
      - name: dotnet
        image: acrname.azurecr.io/dotnet-app:1.0.0
        # ... rest of config
```

### Green Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app-green
  namespace: prod
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dotnet
      version: green
  template:
    metadata:
      labels:
        app: dotnet
        version: green
    spec:
      containers:
      - name: dotnet
        image: acrname.azurecr.io/dotnet-app:2.0.0
        # ... rest of config
```

### Switch Traffic (Update Service)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: dotnet-service
  namespace: prod
spec:
  selector:
    app: dotnet
    version: green  # Change from 'blue' to 'green'
  ports:
  - port: 80
    targetPort: 80
```

**Benefits:**
- ✔ Zero downtime deployments
- ✔ Instant rollback capability
- ✔ Git-controlled traffic switching

**Advanced:** Use [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) for canary and progressive delivery.

---

## 🔄 Argo CD Application

**`argocd-application.yaml`**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dotnet-app-prod
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/YOUR_USERNAME/dotnet-k8s-gitops
    targetRevision: main
    path: k8s-infra/base
  
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### How Argo CD Deploys This

1. You push YAML to Git
2. Argo CD detects change
3. Syncs automatically to cluster
4. AKS state == Git state (always)
5. Drift → auto-healed

---

## 🔧 Configuration

### Update ACR Image

After creating your Azure Container Registry (ACR):

```bash
# Get ACR login server
az acr show --name <your-acr-name> --query loginServer -o tsv

# Update deployment.yaml with actual ACR name
# Replace: acrname.azurecr.io/dotnet-app:1.0.0
# With: <your-acr-name>.azurecr.io/dotnet-app:1.0.0
```

### Build and Push .NET Docker Image

```bash
# Login to ACR
az acr login --name <your-acr-name>

# Build .NET Docker image
docker build -t <your-acr-name>.azurecr.io/dotnet-app:1.0.0 .

# Push image
docker push <your-acr-name>.azurecr.io/dotnet-app:1.0.0
```

### Sample Dockerfile for .NET

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["YourApp.csproj", "./"]
RUN dotnet restore "YourApp.csproj"
COPY . .
RUN dotnet build "YourApp.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "YourApp.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "YourApp.dll"]
```

---

## 🔍 Monitoring & Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n prod
kubectl describe pod <pod-name> -n prod
```

### View Logs
```bash
kubectl logs -n prod -l app=dotnet -f
kubectl logs -n prod <pod-name> --previous
```

### Check HPA Status
```bash
kubectl get hpa -n prod
kubectl describe hpa dotnet-hpa -n prod
```

### Check Service and Endpoints
```bash
kubectl get svc -n prod
kubectl get endpoints -n prod
```

### Check Ingress
```bash
kubectl get ingress -n prod
kubectl describe ingress dotnet-ingress -n prod
```

### Argo CD Sync Status
```bash
kubectl get application -n argocd
argocd app get dotnet-app-prod
argocd app sync dotnet-app-prod
```

### Test Health Endpoint
```bash
kubectl port-forward -n prod svc/dotnet-service 8080:80
curl http://localhost:8080/health
```

---

## 🔒 Security Best Practices

### 1. Use Azure Key Vault for Secrets

Install Azure Key Vault CSI Driver:
```bash
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name
```

### 2. Enable Pod Security Standards
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 3. Use Network Policies
Already included in `networkpolicy.yaml`

### 4. Scan Images for Vulnerabilities
```bash
# Using Trivy
trivy image <your-acr-name>.azurecr.io/dotnet-app:1.0.0
```

---

## 📊 Resource Management

### Monitor Resource Usage
```bash
kubectl top pods -n prod
kubectl top nodes
```

### Adjust Resource Limits
Edit `deployment.yaml` and update:
```yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "1Gi"
```

---

## 🧹 Cleanup

### Delete via Kustomize
```bash
kubectl delete -k k8s-infra/base/
```

### Delete via Argo CD
```bash
argocd app delete dotnet-app-prod
```

### Delete Namespace (removes everything)
```bash
kubectl delete namespace prod
```

---

## 📝 Notes

- Update the GitHub repository URL in `argocd-application.yaml`
- Replace ACR name in deployment manifest after creating ACR
- Use Azure Key Vault CSI Driver for secrets in production
- Enable Azure Policy for AKS compliance
- Configure RBAC according to your security requirements
- Set up Azure Monitor for container insights
- Enable Azure Defender for Kubernetes

---

## 🎓 Additional Resources

- [.NET on Kubernetes Best Practices](https://learn.microsoft.com/en-us/dotnet/architecture/containerized-lifecycle/)
- [Azure Kubernetes Service Documentation](https://docs.microsoft.com/azure/aks/)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)

---

## 📄 License

MIT License - Feel free to use this in your projects!

---

**Happy Deploying! 🚀**
