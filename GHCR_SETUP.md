# GHCR + GitHub Actions + k3s Setup Guide

This guide walks you through setting up automated deployment of your full-stack app from GitHub to k3s using GitHub Container Registry.

## Prerequisites
- ✅ k3s cluster running on 3 Ubuntu VMs (1 control plane, 2 workers)
- ✅ `kubectl` configured on control plane
- ✅ GitHub repository with your code
- ✅ GitHub Account

## Step 1: Get Your GitHub Repository Info

```bash
# Determine your GitHub repository owner and name
# Format: yourusername/fullstack-k8s-webapp
# Or: yourorgname/fullstack-k8s-webapp

# Example: if your repo URL is https://github.com/johndoe/fullstack-k8s-webapp
# Then: GITHUB_OWNER=johndoe, REPO_NAME=fullstack-k8s-webapp
```

## Step 2: Update Kubernetes Manifests

Replace `yourusername` in the deployment files with your actual GitHub username:

```bash
cd deployment/

# Backend deployment
sed -i 's/yourusername/<YOUR_GITHUB_USERNAME>/g' backend-deployment.yaml

# Frontend deployment
sed -i 's/yourusername/<YOUR_GITHUB_USERNAME>/g' frontend-deployment.yaml

# Verify changes
cat backend-deployment.yaml | grep "image:"
cat frontend-deployment.yaml | grep "image:"
```

**Expected output:**
```
image: ghcr.io/yourusername/fullstack-k8s-webapp/backend:latest
image: ghcr.io/yourusername/fullstack-k8s-webapp/frontend:latest
```

## Step 3: Setup GitHub Actions Self-Hosted Runner

Choose ONE location for your runner:

### Option A: Install on k3s Control Plane Node (Recommended)

Connect to your control plane VM and run:

```bash
# Create directory for runner
mkdir -p /home/ubuntu/actions-runner
cd /home/ubuntu/actions-runner

# Download latest runner
curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/download/v2.114.1/actions-runner-linux-x64.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

# Generate registration token from GitHub
# Go to: Your Repo > Settings > Actions > Runners > New self-hosted runner
# Copy the registration token

# Run the configuration script
./config.sh --url https://github.com/<YOUR_GITHUB_OWNER>/<YOUR_REPO_NAME> --token <REGISTRATION_TOKEN>

# When asked for runner name, enter: k3s-runner
# Leave other options as default (press Enter)

# Install runner as service
sudo ./svc.sh install
sudo ./svc.sh start

# Verify runner is running
sudo ./svc.sh status
```

### Option B: Install on External Machine (If not enough resources on VMs)

```bash
# Follow the same steps above but on a machine that can reach your k3s cluster via network
```

### Verify Runner is Connected

In GitHub:
- Go to your repo > Settings > Actions > Runners
- You should see your runner with a green dot (online status)

## Step 4: Create GHCR Authentication on k3s Cluster

### 4a. Create GitHub Personal Access Token (PAT)

1. Go to GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic)
2. Click "Generate new token (classic)"
3. Give it a name: `k3s-deployment`
4. Select scopes:
   - ✅ `read:packages` - read container images
   - ✅ `write:packages` - write container images
5. Click "Generate token"
6. **Copy the token** (you won't see it again)

### 4b. Create imagePullSecret on k3s

Run this on your control plane:

```bash
# Set variables
GITHUB_USERNAME="<YOUR_GITHUB_USERNAME>"
GITHUB_TOKEN="<YOUR_GITHUB_PAT>"
REGISTRY="ghcr.io"

# Create namespace if not exists
kubectl create namespace fullstack-app --dry-run=client -o yaml | kubectl apply -f -

# Create secret
kubectl create secret docker-registry ghcr-secret \
  --docker-server=$REGISTRY \
  --docker-username=$GITHUB_USERNAME \
  --docker-password=$GITHUB_TOKEN \
  --namespace=fullstack-app \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify (don't show the secret content)
kubectl get secrets -n fullstack-app ghcr-secret -o jsonpath='{.data}' | wc -c
# Should output: non-zero number indicating secret exists
```

## Step 5: Prepare Storage Directory on Worker Nodes

Run this on EACH worker node:

```bash
# SSH to each worker node
sudo mkdir -p /mnt/data
sudo chmod 777 /mnt/data

# Verify
ls -la /mnt/data
```

## Step 6: Initial Deployment Setup

Run this once on your control plane:

```bash
# Create the PersistentVolume (needs /mnt/data on nodes)
kubectl apply -f deployment/pv.yaml

# Create the PersistentVolumeClaim
kubectl apply -f deployment/pvc.yaml

# Apply ConfigMap (environment variables)
kubectl apply -f deployment/configmap.yaml

# Apply Secrets (MongoDB URI)
kubectl apply -f deployment/secrets.yaml

# Verify
kubectl get pv
kubectl get pvc -n fullstack-app
kubectl get configmap -n fullstack-app
kubectl get secrets -n fullstack-app
```

## Step 7: Push Code to Enable GitHub Actions

The GitHub Actions workflow triggers on push to main/master branch:

```bash
git add .
git commit -m "Setup GHCR deployment pipeline"
git push origin main
```

**Watch the workflow:**
- Go to your repo > Actions
- You'll see the workflow running: `Build and Deploy to k3s`
- Workflow stages:
  1. `build-and-push` (5-10 min) - Builds images and pushes to GHCR
  2. `deploy-to-k3s` (1-2 min) - Deploys to your k3s cluster

## Step 8: Verify Deployment

```bash
# Check pod status
kubectl get pods -n fullstack-app

# Check services
kubectl get svc -n fullstack-app

# Check logs
kubectl logs -n fullstack-app -l app=backend
kubectl logs -n fullstack-app -l app=frontend
kubectl logs -n fullstack-app -l app=mongodb

# Get frontend access
kubectl get svc frontend-service -n fullstack-app
# Look for NodePort (usually 30000)
# Access at: http://<CONTROL_PLANE_IP>:30000
```

---

## Complete Automation Workflow

```
Developer pushes code to GitHub main branch
        ↓
GitHub Actions workflow triggers
        ↓
[Build Stage]
  - Backend built and pushed to ghcr.io/username/.../backend:latest
  - Frontend built and pushed to ghcr.io/username/.../frontend:latest
        ↓
[Deploy Stage - self-hosted runner on k3s]
  - kubectl pulls new images from GHCR using ghcr-secret
  - Updates deployments with new image tags
  - Database, backend, and frontend pods restart with new images
  - Workflow verifies rollout status
        ↓
New version live on k3s cluster
```

---

## Troubleshooting

### ❌ "ImagePullBackOff" Error

```bash
# Check if secret exists
kubectl get secrets -n fullstack-app ghcr-secret

# Check pod events
kubectl describe pod <pod-name> -n fullstack-app

# Solution: Recreate the secret
kubectl delete secret ghcr-secret -n fullstack-app
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=$GITHUB_USERNAME \
  --docker-password=$GITHUB_TOKEN \
  --namespace=fullstack-app
```

### ❌ "CrashLoopBackOff" on Backend

```bash
# Check logs
kubectl logs <backend-pod> -n fullstack-app

# Common issue: MongoDB not running
kubectl get pods -n fullstack-app -l app=mongodb

# If MongoDB CrashLoopBackOff:
# Check /mnt/data exists on worker node
# Check permissions: sudo ls -la /mnt/data
```

### ❌ GitHub Actions Runner Not Connecting

```bash
# On the runner machine
cd /home/ubuntu/actions-runner
./run.sh

# This shows verbose logs of what's happening
# Look for error messages and fix accordingly
```

### ❌ GHCR Image Not Found

```bash
# Verify image pushed to GHCR
# Go to your GitHub repo > Packages

# Or check via CLI
docker login ghcr.io -u <USERNAME> -p <PAT>
docker pull ghcr.io/username/fullstack-k8s-webapp/backend:latest
```

---

## Performance Tips

- **Image layer caching**: GitHub Actions caches layers to speed up builds (workflow uses `buildcache`)
- **Multi-stage builds**: Already configured in Dockerfiles to minimize final image size
- **Resource limits**: Already set in manifests for proper scheduling

## Security Best Practices ✅

- ✅ Using GHCR instead of public Docker Hub
- ✅ PAT has limited scope (`read:packages`, `write:packages`)
- ✅ imagePullSecret prevents public image pulls
- ✅ Self-hosted runner keeps your k3s credentials local (not in GitHub)

---

## Next Steps

1. ✅ Update manifests with your GitHub username
2. ✅ Set up self-hosted runner
3. ✅ Create GitHub PAT and k3s secret
4. ✅ Prepare storage directories
5. ✅ Push code to trigger first deployment
6. ✅ Monitor GitHub Actions and verify pods running

Questions? Check the troubleshooting section or review workflow logs in GitHub Actions UI.
