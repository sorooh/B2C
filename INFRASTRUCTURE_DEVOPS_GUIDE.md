# ☁️ GlobalMarket: Infrastructure & DevOps Guide
## Complete Azure Infrastructure & CI/CD Pipeline

**Version:** 1.0  
**Date:** November 28, 2025  
**Cloud Provider:** Microsoft Azure  
**Container Orchestration:** Kubernetes (AKS)  

---

## 📋 Table of Contents

1. [Infrastructure Overview](#infrastructure-overview)
2. [Azure Architecture](#azure-architecture)
3. [Kubernetes Setup](#kubernetes)
4. [CI/CD Pipeline](#cicd)
5. [Monitoring & Logging](#monitoring)
6. [Disaster Recovery](#disaster-recovery)
7. [Cost Optimization](#cost-optimization)

---

<a name="infrastructure-overview"></a>
## 🌐 1. Infrastructure Overview

### 1.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    CloudFlare CDN + WAF                      │
│                  (DDoS Protection, SSL/TLS)                  │
└────────────────────────┬────────────────────────────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
    ┌───────▼────────┐       ┌───────▼────────┐
    │  Azure Region  │       │  Azure Region  │
    │   West Europe  │       │   UAE North    │
    │  (Amsterdam)   │       │    (Dubai)     │
    └───────┬────────┘       └───────┬────────┘
            │                         │
    ┌───────▼─────────────────────────▼────────┐
    │     Azure Kubernetes Service (AKS)       │
    │                                           │
    │  ┌────────────────────────────────────┐  │
    │  │       Ingress Controller           │  │
    │  │         (NGINX)                    │  │
    │  └────────────┬───────────────────────┘  │
    │               │                           │
    │  ┌────────────▼───────────────────────┐  │
    │  │      API Gateway (Kong)            │  │
    │  └────────────┬───────────────────────┘  │
    │               │                           │
    │  ┌────────────▼───────────────────────┐  │
    │  │     Microservices (Pods)           │  │
    │  │  ┌─────────┬─────────┬──────────┐  │  │
    │  │  │ Django  │FastAPI  │Background│  │  │
    │  │  │   API   │Services │ Workers  │  │  │
    │  │  └─────────┴─────────┴──────────┘  │  │
    │  └────────────────────────────────────┘  │
    └───────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
   ┌────▼────┐  ┌────▼────┐  ┌───▼────┐
   │Cosmos DB│  │PostgreSQL│  │ Redis  │
   │         │  │          │  │Cluster │
   │ (NoSQL) │  │  (RDBMS) │  │(Cache) │
   └─────────┘  └──────────┘  └────────┘
```

### 1.2 Technology Stack

```yaml
Infrastructure:
  Cloud Provider: Microsoft Azure
  Container Orchestration: Azure Kubernetes Service (AKS)
  CDN & Security: CloudFlare
  DNS: CloudFlare DNS
  Load Balancer: Azure Load Balancer
  
Compute:
  Kubernetes Nodes: Standard_D4s_v3 (4 vCPU, 16 GB RAM)
  Auto-scaling: Enabled (3-10 nodes per region)
  Container Runtime: containerd
  
Databases:
  Primary: Azure Cosmos DB (NoSQL)
  Secondary: Azure Database for PostgreSQL
  Cache: Azure Cache for Redis
  Search: Elasticsearch (Self-hosted on AKS)
  
Storage:
  Blob Storage: Azure Blob Storage (Hot tier)
  File Share: Azure Files (for shared volumes)
  Backup: Azure Backup
  
Networking:
  VNet: Azure Virtual Network
  Subnets: Public, Private, Database
  NSG: Network Security Groups
  Private Link: Enabled for databases
  
Monitoring:
  APM: Datadog
  Logs: Azure Monitor + Datadog
  Metrics: Prometheus + Grafana
  Errors: Sentry
  Uptime: Pingdom
  
CI/CD:
  Source Control: GitHub
  CI/CD: GitHub Actions
  Container Registry: Azure Container Registry (ACR)
  Secrets: Azure Key Vault
```

---

<a name="azure-architecture"></a>
## ☁️ 2. Azure Architecture

### 2.1 Resource Groups

```yaml
Resource Group Structure:

1. globalmarket-eu-prod:
   Purpose: Europe production environment
   Location: West Europe (Amsterdam)
   Resources:
     - AKS Cluster (globalmarket-eu-aks)
     - Cosmos DB (globalmarket-eu-cosmos)
     - PostgreSQL (globalmarket-eu-postgres)
     - Redis (globalmarket-eu-redis)
     - Storage Account (globalmarketeu)
     - Key Vault (globalmarket-eu-kv)
     - Container Registry (globalmarketeu)

2. globalmarket-mena-prod:
   Purpose: MENA production environment
   Location: UAE North (Dubai)
   Resources:
     - AKS Cluster (globalmarket-mena-aks)
     - Cosmos DB (globalmarket-mena-cosmos)
     - PostgreSQL (globalmarket-mena-postgres)
     - Redis (globalmarket-mena-redis)
     - Storage Account (globalmarketmena)
     - Key Vault (globalmarket-mena-kv)

3. globalmarket-shared:
   Purpose: Shared resources
   Location: West Europe
   Resources:
     - GitHub Container Registry mirror
     - Shared monitoring dashboards
     - Backup vaults

4. globalmarket-staging:
   Purpose: Staging environment
   Location: West Europe
   Resources: (Same as prod but smaller)
```

### 2.2 Azure Cosmos DB Setup

```hcl
# terraform/cosmos_db.tf

resource "azurerm_cosmosdb_account" "globalmarket_eu" {
  name                = "globalmarket-eu-cosmos"
  location            = "westeurope"
  resource_group_name = azurerm_resource_group.eu_prod.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"
  
  consistency_policy {
    consistency_level       = "Session"
    max_interval_in_seconds = 5
    max_staleness_prefix    = 100
  }
  
  geo_location {
    location          = "westeurope"
    failover_priority = 0
    zone_redundant    = true
  }
  
  geo_location {
    location          = "northeurope"
    failover_priority = 1
    zone_redundant    = true
  }
  
  capabilities {
    name = "EnableServerless"  # Use Serverless for cost optimization
  }
  
  # Enable automatic failover
  enable_automatic_failover = true
  
  # Enable multiple write locations
  enable_multiple_write_locations = true
  
  # Enable analytical storage for historical data
  analytical_storage_enabled = true
  
  # Backup
  backup {
    type                = "Continuous"
    interval_in_minutes = 240
    retention_in_hours  = 720  # 30 days
  }
  
  tags = {
    Environment = "Production"
    Tenant      = "Europe"
    ManagedBy   = "Terraform"
  }
}

# Cosmos DB Database
resource "azurerm_cosmosdb_sql_database" "products" {
  name                = "globalmarket_eu"
  resource_group_name = azurerm_resource_group.eu_prod.name
  account_name        = azurerm_cosmosdb_account.globalmarket_eu.name
  
  # Serverless - no throughput needed
}

# Products Collection
resource "azurerm_cosmosdb_sql_container" "products" {
  name                  = "products"
  resource_group_name   = azurerm_resource_group.eu_prod.name
  account_name          = azurerm_cosmosdb_account.globalmarket_eu.name
  database_name         = azurerm_cosmosdb_sql_database.products.name
  partition_key_path    = "/tenant_id/vendor_id"
  partition_key_version = 2  # Hierarchical partition keys
  
  # Indexing policy
  indexing_policy {
    indexing_mode = "consistent"
    
    included_path {
      path = "/*"
    }
    
    excluded_path {
      path = "/images/*"
    }
    
    excluded_path {
      path = "/description/*"
    }
  }
  
  # TTL for automatic cleanup
  default_ttl = -1  # Disabled by default
}
```

### 2.3 Azure Kubernetes Service (AKS)

```hcl
# terraform/aks.tf

resource "azurerm_kubernetes_cluster" "globalmarket_eu" {
  name                = "globalmarket-eu-aks"
  location            = "westeurope"
  resource_group_name = azurerm_resource_group.eu_prod.name
  dns_prefix          = "globalmarket-eu"
  kubernetes_version  = "1.28.3"
  
  # Default node pool (system)
  default_node_pool {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D2s_v3"
    type                = "VirtualMachineScaleSets"
    availability_zones  = ["1", "2", "3"]
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 5
    
    # OS disk
    os_disk_size_gb = 100
    os_disk_type    = "Managed"
    
    # Network
    vnet_subnet_id = azurerm_subnet.aks.id
    
    # Labels
    node_labels = {
      "node-type" = "system"
    }
  }
  
  # Identity
  identity {
    type = "SystemAssigned"
  }
  
  # Network profile
  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    load_balancer_sku = "standard"
    outbound_type     = "loadBalancer"
    
    service_cidr   = "10.0.0.0/16"
    dns_service_ip = "10.0.0.10"
  }
  
  # Azure Monitor
  addon_profile {
    oms_agent {
      enabled                    = true
      log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
    }
    
    azure_policy {
      enabled = true
    }
    
    kube_dashboard {
      enabled = false  # Security best practice
    }
  }
  
  # RBAC
  role_based_access_control {
    enabled = true
    
    azure_active_directory {
      managed                = true
      admin_group_object_ids = [var.admin_group_id]
      azure_rbac_enabled     = true
    }
  }
  
  # Auto-upgrade
  automatic_channel_upgrade = "patch"
  
  tags = {
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

# Application node pool
resource "azurerm_kubernetes_cluster_node_pool" "apps" {
  name                  = "apps"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.globalmarket_eu.id
  vm_size               = "Standard_D4s_v3"
  node_count            = 3
  
  enable_auto_scaling = true
  min_count           = 3
  max_count           = 10
  
  availability_zones = ["1", "2", "3"]
  
  node_labels = {
    "node-type" = "application"
  }
  
  node_taints = []
  
  tags = {
    Purpose = "Application Workloads"
  }
}
```

---

<a name="kubernetes"></a>
## ⚙️ 3. Kubernetes Configuration

### 3.1 Namespace Structure

```yaml
# k8s/namespaces.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: globalmarket-prod
  labels:
    name: globalmarket-prod
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: globalmarket-staging
  labels:
    name: globalmarket-staging
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
    purpose: observability
```

### 3.2 Django API Deployment

```yaml
# k8s/deployments/django-api.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-api
  namespace: globalmarket-prod
  labels:
    app: django-api
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: django-api
  template:
    metadata:
      labels:
        app: django-api
        version: v1.0.0
    spec:
      # Service Account
      serviceAccountName: django-api
      
      # Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      # Init Containers
      initContainers:
      - name: wait-for-db
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          until nc -z postgres-service 5432; do
            echo "Waiting for PostgreSQL...";
            sleep 2;
          done
      
      # Main Container
      containers:
      - name: django
        image: globalmarketeu.azurecr.io/django-api:1.0.0
        imagePullPolicy: Always
        
        # Resources
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        
        # Ports
        ports:
        - name: http
          containerPort: 8000
          protocol: TCP
        
        # Environment Variables
        env:
        - name: DJANGO_SETTINGS_MODULE
          value: "config.settings.production"
        - name: ENVIRONMENT
          value: "production"
        - name: TENANT_ID
          value: "eu-west-1"
        
        # Secrets from Azure Key Vault
        envFrom:
        - secretRef:
            name: django-secrets
        
        # Health Checks
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Startup Probe (for slow starts)
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8000
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 30
        
        # Volume Mounts
        volumeMounts:
        - name: static-files
          mountPath: /app/static
        - name: media-files
          mountPath: /app/media
      
      # Volumes
      volumes:
      - name: static-files
        persistentVolumeClaim:
          claimName: static-files-pvc
      - name: media-files
        azureFile:
          secretName: azure-storage-secret
          shareName: media-files
          readOnly: false
      
      # Image Pull Secrets
      imagePullSecrets:
      - name: acr-secret
      
      # Node Selector
      nodeSelector:
        node-type: application
      
      # Affinity (spread across zones)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - django-api
              topologyKey: topology.kubernetes.io/zone
---
apiVersion: v1
kind: Service
metadata:
  name: django-api-service
  namespace: globalmarket-prod
spec:
  type: ClusterIP
  selector:
    app: django-api
  ports:
  - name: http
    port: 80
    targetPort: 8000
    protocol: TCP
```

### 3.3 Horizontal Pod Autoscaler

```yaml
# k8s/hpa/django-api-hpa.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: django-api-hpa
  namespace: globalmarket-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: django-api
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric: Request rate
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
    
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 4
        periodSeconds: 30
      selectPolicy: Max
```

---

<a name="cicd"></a>
## 🔄 4. CI/CD Pipeline

### 4.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy-production.yml

name: Deploy to Production

on:
  push:
    branches:
      - main
    paths:
      - 'backend/**'
      - '.github/workflows/deploy-production.yml'
  
  workflow_dispatch:  # Manual trigger

env:
  AZURE_REGISTRY: globalmarketeu.azurecr.io
  IMAGE_NAME: django-api
  CLUSTER_NAME: globalmarket-eu-aks
  RESOURCE_GROUP: globalmarket-eu-prod
  NAMESPACE: globalmarket-prod

jobs:
  # Job 1: Build and Test
  build:
    name: Build & Test
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'
    
    - name: Install dependencies
      run: |
        cd backend
        pip install -r requirements/test.txt
    
    - name: Run linting
      run: |
        cd backend
        black --check .
        flake8 .
        mypy .
    
    - name: Run tests
      run: |
        cd backend
        pytest --cov=apps --cov-report=xml --cov-report=html
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./backend/coverage.xml
    
    - name: Security scan (Bandit)
      run: |
        cd backend
        bandit -r apps/ -f json -o bandit-report.json
    
    - name: Dependency check
      run: |
        cd backend
        pip-audit
  
  # Job 2: Build Docker Image
  docker:
    name: Build Docker Image
    needs: build
    runs-on: ubuntu-latest
    
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Azure Container Registry
      uses: azure/docker-login@v1
      with:
        login-server: ${{ env.AZURE_REGISTRY }}
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.AZURE_REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix={{date 'YYYYMMDD'}}-
          type=ref,event=branch
          type=semver,pattern={{version}}
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: ./backend
        file: ./backend/docker/Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=registry,ref=${{ env.AZURE_REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
        cache-to: type=registry,ref=${{ env.AZURE_REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
        build-args: |
          ENVIRONMENT=production
  
  # Job 3: Deploy to Kubernetes
  deploy:
    name: Deploy to AKS
    needs: docker
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Set AKS context
      uses: azure/aks-set-context@v3
      with:
        resource-group: ${{ env.RESOURCE_GROUP }}
        cluster-name: ${{ env.CLUSTER_NAME }}
    
    - name: Install kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Install Helm
      uses: azure/setup-helm@v3
      with:
        version: '3.13.0'
    
    - name: Deploy to Kubernetes
      run: |
        helm upgrade --install django-api ./k8s/helm/django-api \
          --namespace ${{ env.NAMESPACE }} \
          --set image.repository=${{ env.AZURE_REGISTRY }}/${{ env.IMAGE_NAME }} \
          --set image.tag=${{ needs.docker.outputs.image_tag }} \
          --set environment=production \
          --set replicas=3 \
          --wait \
          --timeout 5m
    
    - name: Verify deployment
      run: |
        kubectl rollout status deployment/django-api \
          -n ${{ env.NAMESPACE }} \
          --timeout=5m
    
    - name: Run smoke tests
      run: |
        kubectl run smoke-test \
          --image=curlimages/curl:latest \
          --restart=Never \
          --rm \
          -i \
          -n ${{ env.NAMESPACE }} \
          -- curl -f http://django-api-service/health
  
  # Job 4: Notify
  notify:
    name: Notify Team
    needs: [build, docker, deploy]
    runs-on: ubuntu-latest
    if: always()
    
    steps:
    - name: Slack Notification
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "Deployment to Production",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Deployment Status:* ${{ job.status }}\n*Environment:* Production\n*Commit:* ${{ github.sha }}"
                }
              }
            ]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

**🎯 Infrastructure & DevOps Guide كامل!**

الآن بكمل **Development Setup Guide** - آخر وثيقة! 🚀