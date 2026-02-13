# OpenMRS Deployment Guide for k3s (Single Node)

This guide will help you deploy OpenMRS on a single-node k3s cluster, which is perfect for Proxmox VMs or local development.

## Prerequisites

1. **k3s installed** on your Proxmox VM or local machine
   ```bash
   # Install k3s (single node, no agents)
   curl -sfL https://get.k3s.io | sh -
   
   # Verify installation
   sudo k3s kubectl get nodes
   ```

2. **kubectl configured** to access your k3s cluster
   ```bash
   # k3s creates a kubeconfig at /etc/rancher/k3s/k3s.yaml
   export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
   # Or copy it to your home directory
   mkdir -p ~/.kube
   sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
   sudo chown $USER ~/.kube/config
   ```

3. **Helm installed**
   ```bash
   # On Linux
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   
   # Or using package manager
   # Ubuntu/Debian
   sudo apt-get install helm
   # CentOS/RHEL
   sudo yum install helm
   ```

4. **Ingress Controller** (nginx-ingress)
   ```bash
   # Install nginx-ingress for k3s
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
   
   # Wait for ingress controller to be ready
   kubectl wait --namespace ingress-nginx \
     --for=condition=ready pod \
     --selector=app.kubernetes.io/component=controller \
     --timeout=90s
   ```

## Deployment Steps (Individual Components)

### Step 1: Create Namespace

```bash
kubectl create namespace openmrs
```

### Step 2: Deploy MariaDB Database (Standalone)

First, deploy only the database to ensure it's running and accessible:

```bash
cd helm/openmrs-backend

# IMPORTANT: Add Bitnami Helm repository first (if not already added)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Update Helm dependencies
# This downloads the required MariaDB, Elasticsearch, and MinIO charts
helm dependency update

# Verify dependencies are downloaded
ls -la charts/

# Deploy MariaDB only (backend disabled temporarily)
helm upgrade --install openmrs-db . \
  --namespace openmrs \
  --set mariadb.enabled=true \
  --set mariadb.architecture=standalone \
  --set mariadb.primary.persistence.storageClass=local-path \
  --set mariadb.primary.persistence.size=8Gi \
  --set mariadb.auth.rootPassword=Root123 \
  --set mariadb.auth.database=openmrs \
  --set mariadb.auth.username=openmrs \
  --set mariadb.auth.password=OpenMRS123 \
  --set global.defaultStorageClass=local-path \
  --set replicaCount=0  # Disable backend deployment
```

**Wait for MariaDB to be ready:**
```bash
# Check MariaDB pod status
kubectl get pods -n openmrs -w

# Wait until MariaDB is running (should see "Running" status)
kubectl wait --namespace openmrs \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=mariadb \
  --timeout=300s

# Verify database is accessible
kubectl exec -it -n openmrs $(kubectl get pod -n openmrs -l app.kubernetes.io/name=mariadb -o jsonpath='{.items[0].metadata.name}') -- \
  mariadb -u openmrs -pOpenMRS123 -e "SHOW DATABASES;"
```

### Step 3: Deploy OpenMRS Backend

Once MariaDB is running, deploy the backend:

```bash
# Deploy backend (MariaDB will be skipped if already exists)
helm upgrade --install openmrs-backend . \
  --namespace openmrs \
  --set mariadb.enabled=true \
  --set mariadb.architecture=standalone \
  --set mariadb.primary.persistence.storageClass=local-path \
  --set mariadb.auth.rootPassword=Root123 \
  --set mariadb.auth.database=openmrs \
  --set mariadb.auth.username=openmrs \
  --set mariadb.auth.password=OpenMRS123 \
  --set global.defaultStorageClass=local-path \
  --set global.defaultIngressClass=nginx \
  --set replicaCount=1 \
  --set persistence.enabled=true \
  --set persistence.size=8Gi \
  --set ingress.enabled=true
```

**Wait for backend to be ready:**
```bash
# Check backend pod status
kubectl get pods -n openmrs -w

# Wait until backend is running and ready
kubectl wait --namespace openmrs \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=openmrs-backend \
  --timeout=600s

# Check backend logs to verify database connection
kubectl logs -n openmrs -l app.kubernetes.io/name=openmrs-backend --tail=50
```

**Troubleshooting database connection:**
If backend is not connecting to database, check:
```bash
# Verify MariaDB service exists
kubectl get svc -n openmrs | grep mariadb

# Test database connection from backend pod
kubectl exec -it -n openmrs $(kubectl get pod -n openmrs -l app.kubernetes.io/name=openmrs-backend -o jsonpath='{.items[0].metadata.name}') -- \
  sh -c "echo 'SELECT 1;' | mariadb -h openmrs-backend-mariadb -u openmrs -pOpenMRS123 openmrs"
```

### Step 4: Deploy OpenMRS Frontend

Finally, deploy the frontend:

```bash
cd ../openmrs-frontend

# Deploy frontend
helm upgrade --install openmrs-frontend . \
  --namespace openmrs \
  --set global.defaultIngressClass=nginx \
  --set replicaCount=1 \
  --set ingress.enabled=true
```

**Wait for frontend to be ready:**
```bash
# Check frontend pod status
kubectl get pods -n openmrs

# Verify frontend is running
kubectl wait --namespace openmrs \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=openmrs-frontend \
  --timeout=300s
```

## Accessing OpenMRS

### Option 1: Port Forwarding (Local Access)

```bash
# Forward ingress controller port
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80

# Access OpenMRS at:
# Frontend: http://localhost:8080/openmrs/spa/home
# Backend API: http://localhost:8080/openmrs/
```

### Option 2: NodePort (Direct Access)

If you want to access directly via node IP:

```bash
# Get node IP
kubectl get nodes -o wide

# Access via node IP (if NodePort is configured)
# Check ingress controller service type
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

### Option 3: LoadBalancer (If supported)

If your k3s setup supports LoadBalancer (e.g., with MetalLB):

```bash
# Check if LoadBalancer IP is assigned
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

## Verification

Check all components are running:

```bash
# Check all pods
kubectl get pods -n openmrs

# Check all services
kubectl get svc -n openmrs

# Check ingress
kubectl get ingress -n openmrs

# Check persistent volumes
kubectl get pv,pvc -n openmrs
```

Expected output should show:
- 1 MariaDB pod (Running)
- 1 OpenMRS Backend pod (Running)
- 1 OpenMRS Frontend pod (Running)

## Troubleshooting

### Database Connection Issues

1. **Check MariaDB is running:**
   ```bash
   kubectl get pods -n openmrs | grep mariadb
   kubectl logs -n openmrs -l app.kubernetes.io/name=mariadb
   ```

2. **Verify service DNS resolution:**
   ```bash
   kubectl run -it --rm debug --image=busybox --restart=Never -n openmrs -- \
     nslookup openmrs-backend-mariadb
   ```

3. **Test database connection manually:**
   ```bash
   kubectl exec -it -n openmrs $(kubectl get pod -n openmrs -l app.kubernetes.io/name=mariadb -o jsonpath='{.items[0].metadata.name}') -- \
     mariadb -u openmrs -pOpenMRS123 -e "SELECT 1;"
   ```

### Backend Not Starting

1. **Check backend logs:**
   ```bash
   kubectl logs -n openmrs -l app.kubernetes.io/name=openmrs-backend --tail=100
   ```

2. **Check if JDBC URL is generated correctly:**
   ```bash
   kubectl exec -it -n openmrs $(kubectl get pod -n openmrs -l app.kubernetes.io/name=openmrs-backend -o jsonpath='{.items[0].metadata.name}') -- \
     cat /work-dir/jdbc-url.txt
   ```

3. **Verify environment variables:**
   ```bash
   kubectl exec -it -n openmrs $(kubectl get pod -n openmrs -l app.kubernetes.io/name=openmrs-backend -o jsonpath='{.items[0].metadata.name}') -- \
     env | grep OMRS
   ```

### Storage Issues

1. **Check if local-path storage class exists:**
   ```bash
   kubectl get storageclass
   ```

2. **If local-path doesn't exist, install it:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
   ```

3. **Check persistent volume claims:**
   ```bash
   kubectl get pvc -n openmrs
   kubectl describe pvc -n openmrs
   ```

## Cleanup

To remove all components:

```bash
# Delete all Helm releases
helm uninstall openmrs-backend -n openmrs
helm uninstall openmrs-frontend -n openmrs

# Delete namespace (this will delete all resources)
kubectl delete namespace openmrs
```

## Configuration Values

Key configuration values for k3s single-node deployment:

- **Storage Class**: `local-path` (k3s default)
- **MariaDB Architecture**: `standalone` (no replication)
- **Ingress Class**: `nginx`
- **Replica Count**: 1 (for all components)

## Notes

- This configuration is optimized for single-node k3s clusters
- For production multi-node clusters, consider using replication architecture
- Storage uses local-path provisioner, data persists on the node's filesystem
- All passwords are set to defaults - change them for production use
