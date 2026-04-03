# GitHub Actions + k3s Deployment Troubleshooting

## Common Issues and Solutions

### 1. GitHub Actions Runner Issues

#### Issue: "No runners available matching"
```
No runners available matching the required labels: 'self-hosted'
```

**Solution:**
```bash
# 1. Check runner is installed on the machine
cd /home/ubuntu/actions-runner
ls -la config.sh

# 2. Start the runner service
sudo ./svc.sh start

# 3. Check status
sudo ./svc.sh status

# 4. Check in GitHub repo > Settings > Actions > Runners
# The runner should show as green (online)

# 5. If runner keeps going offline, check logs
tail -f /home/ubuntu/actions-runner/_diag/Runner_*.log
```

---

### 2. Image Build Failures

#### Issue: "Docker build failed" in GitHub Actions
```
ERROR: failed to solve with frontend dockerfile.v0
```

**Solution:**
```bash
# 1. Check Dockerfile syntax locally
docker build -t test-build ./backend
docker build -t test-build ./frontend

# 2. Check for missing files
ls -la backend/Dockerfile
ls -la frontend/Dockerfile
cat backend/Dockerfile

# 3. Check dependencies
cat backend/package.json | grep -A 5 dependencies
cat frontend/package.json | grep -A 5 dependencies

# 4. Push fix and retry
git add Dockerfile
git commit -m "Fix Dockerfile"
git push
```

---

### 3. ImagePullBackOff on k3s

#### Issue: Pods stuck in "ImagePullBackOff"
```
kubectl get pods -n fullstack-app
# backend-deployment-xxxxx   0/1   ImagePullBackOff
```

**Commands to debug:**
```bash
# Check pod details
kubectl describe pod <pod-name> -n fullstack-app

# Look for message like:
# Failed to pull image "ghcr.io/...backend:latest": ... 401 Unauthorized

# This means credentials are wrong or secret missing

# Solution 1: Verify secret exists
kubectl get secrets -n fullstack-app ghcr-secret

# Solution 2: Check secret is referenced in deployment
kubectl get deployment backend-deployment -n fullstack-app -o yaml | grep imagePullSecrets

# Solution 3: Recreate secret with correct credentials
kubectl delete secret ghcr-secret -n fullstack-app

# Verify your GitHub PAT has correct scopes:
# - read:packages
# - write:packages

kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USERNAME> \
  --docker-password=<GITHUB_PAT> \
  --namespace=fullstack-app

# Solution 4: Force pod recreation
kubectl rollout restart deployment/backend-deployment -n fullstack-app
```

---

### 4. Authentication Failures

#### Issue: "401 Unauthorized" when pushing to GHCR

**Check GitHub Actions workflow has correct setup:**
```yaml
# In .github/workflows/deploy.yml - verify this exists:
- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}          # Automatically your GitHub username
    password: ${{ secrets.GITHUB_TOKEN }}  # Built-in GitHub token
```

**Verify GitHub token has permissions:**
- Go to repo > Settings > Actions > General > Workflow permissions
- Select: "Read and write permissions"

---

### 5. Database Connection Issues

#### Issue: Backend can't connect to MongoDB

**Check MongoDB is running:**
```bash
kubectl get pods -n fullstack-app -l app=mongodb

# If MongoDB is pending/failed
kubectl describe pod <mongo-pod> -n fullstack-app

# Common issue: /mnt/data doesn't exist on worker node
# SSH to the worker node and create it:
sudo mkdir -p /mnt/data
sudo chmod 777 /mnt/data
```

**Check backend has correct MongoDB URI:**
```bash
# Get the secret
kubectl get secret newsletter-secret -n fullstack-app -o yaml

# Decode the URI
kubectl get secret newsletter-secret -n fullstack-app -o jsonpath='{.data.MONGODB_URI}' | base64 -d
# Should output: mongodb://mongodb-service:27017/newsletter

# Verify DNS resolution
kubectl exec -it <backend-pod> -n fullstack-app -- nslookup mongodb-service
```

---

### 6. Network/Connectivity Issues

#### Issue: k3s cluster can't reach GHCR (no internet)

**Solution: Use local registry**
```bash
# If k3s nodes don't have internet access, you need local registry
# Option 1: Setup Harbor on a node with internet
# Option 2: Pre-load images manually

# Pre-load images manually:
# On machine with internet:
docker pull ghcr.io/yourusername/fullstack-k8s-webapp/backend:latest
docker pull ghcr.io/yourusername/fullstack-k8s-webapp/frontend:latest

# Export to file
docker save -o backend.tar ghcr.io/yourusername/fullstack-k8s-webapp/backend:latest
docker save -o frontend.tar ghcr.io/yourusername/fullstack-k8s-webapp/frontend:latest

# Copy to k3s node and load
scp backend.tar ubuntu@<k3s-node-ip>:~
ssh ubuntu@<k3s-node-ip>
k3s ctr images import backend.tar
k3s ctr images import frontend.tar
```

---

### 7. Deployment Status Checks

#### Check rollout status
```bash
# Wait for deployment to be ready
kubectl rollout status deployment/backend-deployment -n fullstack-app --timeout=5m

# If timeout, check why:
kubectl get pods -n fullstack-app
kubectl describe pod <pending-pod> -n fullstack-app
kubectl logs <pod> -n fullstack-app
```

---

### 8. View Detailed Logs

```bash
# GitHub Actions logs
# Go to: repo > Actions > workflow run > click job > see live logs

# Kubernetes logs
kubectl logs <pod-name> -n fullstack-app          # Current logs
kubectl logs <pod-name> -n fullstack-app --tail=50  # Last 50 lines
kubectl logs <pod-name> -n fullstack-app -f       # Follow logs (like tail -f)

# Previous logs (if pod crashed and restarted)
kubectl logs <pod-name> -n fullstack-app --previous

# Logs from all pods in namespace
kubectl logs -n fullstack-app --all-containers=true -l app=backend
```

---

### 9. Reset and Clean Deploy

```bash
# If something is very broken, start fresh:

# Delete everything in the namespace
kubectl delete namespace fullstack-app

# Create fresh namespace
kubectl create namespace fullstack-app

# Recreate secrets
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<USERNAME> \
  --docker-password=<PAT> \
  --namespace=fullstack-app

# Redeploy
bash deploy-k3s
```

---

### 10. Verify GHCR Images Exist

```bash
# Check GitHub Packages
# Go to: Your GitHub Profile > Packages > Select a container

# Or list via CLI
docker login ghcr.io -u <USERNAME> -p <PAT>
docker images | grep ghcr.io

# Or use curl
curl -H "Authorization: Bearer <PAT>" \
  https://api.github.com/user/packages?package_type=container
```

---

## Quick Diagnostic Commands

```bash
# 1. Overall cluster health
kubectl get nodes
kubectl get namespaces
kubectl get all -n fullstack-app

# 2. Deployment status
kubectl get deployments -n fullstack-app
kubectl get pods -n fullstack-app -o wide

# 3. Service connectivity
kubectl get svc -n fullstack-app
kubectl get endpoints -n fullstack-app

# 4. Storage
kubectl get pv
kubectl get pvc -n fullstack-app

# 5. Secrets and ConfigMaps
kubectl get secrets -n fullstack-app
kubectl get configmap -n fullstack-app

# 6. Events (what happened recently)
kubectl get events -n fullstack-app --sort-by='.lastTimestamp'
```

---

## How to Report Issues

If deployment still fails, collect this info:

```bash
# 1. Workflow output
# Screenshot from GitHub Actions > your workflow run

# 2. Pod description
kubectl describe pod -n fullstack-app --all=true > pods-describe.txt

# 3. Events
kubectl get events -n fullstack-app --sort-by='.lastTimestamp' > events.txt

# 4. Logs
kubectl logs -n fullstack-app --all-containers=true > all-logs.txt
```

Share these files when asking for help!
