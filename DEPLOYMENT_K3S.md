# OpenMRS Deployment Guide for k3s (Single Node)

This guide will help you deploy OpenMRS on a single-node k3s cluster, which is perfect for Proxmox VMs or local development.

## Quick Start for Proxmox VM

If you're deploying on a Proxmox VM and want to access OpenMRS at `http://<VM_IP>/openmrs` (e.g., `http://192.168.1.172/openmrs`):

1. **Configure ingress controller as NodePort** (see Prerequisites section below)
2. **Deploy OpenMRS** following the deployment steps
3. **Access at:** `http://<YOUR_VM_IP>/openmrs/spa/home`

**Example:** If your VM IP is `192.168.1.172`:
- Frontend: http://192.168.1.172/openmrs/spa/home
- Backend API: http://192.168.1.172/openmrs/

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
   
   # IMPORTANT for Proxmox VM access: Configure ingress controller as NodePort
   # This allows access via VM IP address (e.g., http://192.168.1.172/openmrs)
   kubectl patch svc ingress-nginx-controller -n ingress-nginx \
     -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":80,"protocol":"TCP","name":"http"},{"port":443,"targetPort":443,"protocol":"TCP","name":"https"}]}}'
   
   # Verify the service is configured as NodePort
   kubectl get svc -n ingress-nginx ingress-nginx-controller
   # You should see ports like 80:3XXXX/TCP and 443:3YYYY/TCP
   ```

## Deployment Steps (Individual Components)

### Step 1: Create Namespace

```bash
kubectl create namespace openmrs
```

### Step 2: Deploy MariaDB Database (Standalone)

First, deploy only the database to ensure it's running and accessible:
git clone 

git clone https://github.com/mdaminulislam42610-cyber/openmrs-contrib-cluster.git

```bash
cd openmrs-contrib-cluster/helm/openmrs-backend

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
# If you deployed MariaDB separately (openmrs-db), delete the conflicting ingress first
kubectl delete ingress -n openmrs openmrs-db-openmrs-backend 2>/dev/null || true

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

### Quick Check - Verify Everything is Running

First, verify all components are ready:

```bash
# Check all pods are running
kubectl get pods -n openmrs

# Check ingress is configured
kubectl get ingress -n openmrs

# Check ingress controller is running
kubectl get pods -n ingress-nginx
```

Expected output:
- All pods should show status "Running" or "Ready"
- Ingress should show "ADDRESS" (may be empty on single node, that's OK)

### Option 1: Access via Proxmox VM IP (Recommended for Proxmox)

**For Proxmox VM deployments, access OpenMRS directly via VM IP:**

```bash
# First, ensure ingress controller is configured as NodePort
kubectl get svc -n ingress-nginx ingress-nginx-controller

# If type is not NodePort, configure it:
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":80,"protocol":"TCP","name":"http"},{"port":443,"targetPort":443,"protocol":"TCP","name":"https"}]}}'

# Get your VM IP (replace with your actual IP)
# Example: 192.168.1.172
kubectl get nodes -o wide
```

**Access OpenMRS via your Proxmox VM IP:**

- **Frontend (Main Application):** 
  - http://192.168.1.172/openmrs/spa/home
  - Or: http://192.168.1.172/openmrs/spa

- **Backend API:**
  - http://192.168.1.172/openmrs/
  - Health check: http://192.168.1.172/openmrs/health/alive

**Default Login Credentials:**
- Username: `admin`
- Password: `Admin123` (or check OpenMRS documentation for default credentials)

**Troubleshooting Proxmox access:**

1. **If port 80 doesn't work, check NodePort:**
   ```bash
   kubectl get svc -n ingress-nginx ingress-nginx-controller
   # Look for port like 80:31234/TCP - use the 31234 number
   # Access via: http://192.168.1.172:31234/openmrs/spa/home
   ```

2. **Check firewall on Proxmox VM:**
   ```bash
   sudo ufw status
   # If needed, allow ports:
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

3. **Test from VM itself:**
   ```bash
   curl http://localhost/openmrs/health/alive
   curl http://192.168.1.172/openmrs/health/alive
   ```

### Option 2: Port Forwarding (For Local Testing)

If you want to test locally first:

```bash
# Forward ingress controller port (run this in a separate terminal or background)
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80

# Keep this terminal open, or run in background:
# kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 &
```

**Then open your browser and visit:**

- **OpenMRS Frontend (Main Application):** 
  - http://localhost:8080/openmrs/spa/home
  - Or just: http://localhost:8080/openmrs/spa

- **OpenMRS Backend API:**
  - http://localhost:8080/openmrs/
  - Health check: http://localhost:8080/openmrs/health/alive

### Option 3: Direct Service Port Forward (Alternative)

If ingress doesn't work, you can access services directly:

```bash
# Forward backend service directly
kubectl port-forward -n openmrs svc/openmrs-backend 8080:8080
# Access at: http://localhost:8080/openmrs/

# Forward frontend service directly  
kubectl port-forward -n openmrs svc/openmrs-frontend 3000:80
# Access at: http://localhost:3000
```

### Option 4: LoadBalancer (If MetalLB is installed)

If you have MetalLB installed on k3s:

```bash
# Check if LoadBalancer IP is assigned
kubectl get svc -n ingress-nginx ingress-nginx-controller

# If an EXTERNAL-IP is shown, access via:
# http://<EXTERNAL-IP>/openmrs/spa/home
```

### Troubleshooting Access Issues

**If you get a 404 error when accessing OpenMRS:**

**Step 1: Verify all components are running**
```bash
# Check all pods are running
kubectl get pods -n openmrs
# Expected: All pods should show "Running" status

# Check ingress controller
kubectl get pods -n ingress-nginx
# Should show ingress-nginx-controller running
```

**Step 2: Check ingress configuration**
```bash
# List all ingress resources
kubectl get ingress -n openmrs

# Describe ingress to see detailed configuration
kubectl describe ingress -n openmrs

# Check if ingress has the correct paths
kubectl get ingress -n openmrs -o yaml
# Look for paths: /openmrs/ and /openmrs/spa
```

**Step 3: Verify services exist and are correct**
```bash
# Check services
kubectl get svc -n openmrs

# Verify backend service
kubectl get svc -n openmrs openmrs-backend -o yaml

# Verify frontend service
kubectl get svc -n openmrs openmrs-frontend -o yaml
```

**Step 4: Test services directly (bypass ingress)**
```bash
# Test backend service directly
kubectl port-forward -n openmrs svc/openmrs-backend 8080:8080
# In another terminal or browser, test:
curl http://localhost:8080/openmrs/health/alive
# Should return JSON response

# Test frontend service directly
kubectl port-forward -n openmrs svc/openmrs-frontend 3000:80
# In browser: http://localhost:3000
```

**Step 5: Check ingress controller logs**
```bash
# Check ingress controller logs for routing errors
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=100

# Look for errors like:
# - "no upstream"
# - "404"
# - "service not found"
```

**Step 6: Verify ingress controller is receiving traffic**
```bash
# Check ingress controller service
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Test if ingress controller is responding
curl -H "Host: $(kubectl get ingress -n openmrs -o jsonpath='{.items[0].spec.rules[0].host}')" \
  http://localhost/openmrs/health/alive

# Or test directly
curl http://192.168.1.172/openmrs/health/alive
```

**Step 7: Common fixes for 404 errors**

**Fix 1: Recreate ingress if misconfigured**
```bash
# Delete and let Helm recreate
kubectl delete ingress -n openmrs --all
# Then upgrade the Helm release
helm upgrade openmrs-backend . -n openmrs --reuse-values
helm upgrade openmrs-frontend ../openmrs-frontend -n openmrs --reuse-values
```

**Fix 2: Check if backend is actually ready**
```bash
# Check backend logs
kubectl logs -n openmrs -l app.kubernetes.io/name=openmrs-backend --tail=50

# Check if backend health endpoint works
kubectl exec -n openmrs $(kubectl get pod -n openmrs -l app.kubernetes.io/name=openmrs-backend -o jsonpath='{.items[0].metadata.name}') -- \
  curl -s http://localhost:8080/openmrs/health/alive
```

**Fix 3: Verify ingress paths match**
```bash
# Backend should have path: /openmrs/
kubectl get ingress -n openmrs openmrs-backend -o jsonpath='{.spec.rules[0].http.paths[0].path}'
# Should output: /openmrs/

# Frontend should have path: /openmrs/spa
kubectl get ingress -n openmrs openmrs-frontend -o jsonpath='{.spec.rules[0].http.paths[0].path}'
# Should output: /openmrs/spa(/|$)(.*)
```

**Fix 4: Check ingress controller annotations**
```bash
# Verify ingress has correct class
kubectl get ingress -n openmrs -o jsonpath='{.items[*].spec.ingressClassName}'
# Should output: nginx

# If empty, add it:
kubectl patch ingress -n openmrs openmrs-backend -p '{"spec":{"ingressClassName":"nginx"}}'
kubectl patch ingress -n openmrs openmrs-frontend -p '{"spec":{"ingressClassName":"nginx"}}'
```

**Fix 5: Restart ingress controller (if needed)**
```bash
# Restart ingress controller pods
kubectl rollout restart deployment -n ingress-nginx ingress-nginx-controller

# Wait for it to be ready
kubectl rollout status deployment -n ingress-nginx ingress-nginx-controller
```

**Quick diagnostic script:**
```bash
#!/bin/bash
echo "=== Checking OpenMRS Deployment ==="
echo ""
echo "1. Pods:"
kubectl get pods -n openmrs
echo ""
echo "2. Services:"
kubectl get svc -n openmrs
echo ""
echo "3. Ingress:"
kubectl get ingress -n openmrs
echo ""
echo "4. Ingress Details:"
kubectl describe ingress -n openmrs
echo ""
echo "5. Testing backend health (from pod):"
BACKEND_POD=$(kubectl get pod -n openmrs -l app.kubernetes.io/name=openmrs-backend -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
if [ ! -z "$BACKEND_POD" ]; then
  kubectl exec -n openmrs $BACKEND_POD -- curl -s http://localhost:8080/openmrs/health/alive || echo "Backend not responding"
else
  echo "Backend pod not found"
fi
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
