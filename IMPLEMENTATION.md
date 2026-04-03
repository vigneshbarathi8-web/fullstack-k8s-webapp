# Implementation Summary: GHCR + GitHub Actions + k3s

## ✅ What Has Been Created

### 1. **GitHub Actions Workflow** (`.github/workflows/deploy.yml`)
- ✅ Automatically builds Docker images when you push to main/master
- ✅ Pushes images to GHCR with git SHA tags
- ✅ Self-hosted runner on k3s deploys to your cluster
- ✅ Includes health checks and rollout verification

**Triggers:** Push to main/master branch

**What it does:**
1. Builds backend image → pushes to `ghcr.io/username/backend:latest` and `:sha`
2. Builds frontend image → pushes to `ghcr.io/username/frontend:latest` and `:sha`
3. Deploys to k3s with automatic kubectl rollout verification

---

### 2. **Updated Kubernetes Manifests**
- ✅ `deployment/backend-deployment.yaml` - Updated to use GHCR + imagePullSecrets
- ✅ `deployment/frontend-deployment.yaml` - Updated to use GHCR + imagePullSecrets

**Changes made:**
- Image references changed from Docker Hub → GHCR
- Added `imagePullSecrets: ghcr-secret` for authentication
- Ready for automated tag updates from GitHub Actions

---

### 3. **Setup & Reference Documentation**

| File | Purpose |
|------|---------|
| `GHCR_SETUP.md` | 📖 Step-by-step detailed setup guide (7 steps) |
| `QUICKSTART.md` | ⚡ Quick reference and cheat sheet |
| `TROUBLESHOOTING.md` | 🔧 Common problems and solutions |
| `deploy-k3s` | 🚀 Bash script for manual deployment |

---

## 🎯 Next Steps (In Order)

### **Phase 1: Local Preparation** (~15 minutes)

1. **Update image references in manifests** ← **START HERE**
   ```bash
   cd deployment/
   sed -i 's/yourusername/<YOUR_GITHUB_USERNAME>/g' backend-deployment.yaml frontend-deployment.yaml
   ```

2. **Commit and push updated files**
   ```bash
   git add .github/workflows/deploy.yml GHCR_SETUP.md QUICKSTART.md TROUBLESHOOTING.md deploy-k3s deployment/
   git commit -m "Add GitHub Actions + GHCR deployment pipeline"
   git push origin main
   ```

---

### **Phase 2: GitHub Setup** (~5 minutes)

3. **Create GitHub Personal Access Token (PAT)**
   - Go to: GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Click "Generate new token (classic)"
   - Scopes: ✅ `read:packages`, ✅ `write:packages`
   - Copy the token (save it securely)

4. **Setup workflow permissions**
   - Go to: Your Repo → Settings → Actions → General → Workflow permissions
   - Select: "Read and write permissions"
   - Click "Save"

---

### **Phase 3: k3s Infrastructure Setup** (~20 minutes)

5. **Setup storage on worker nodes**
   ```bash
   # SSH to each worker node and run:
   sudo mkdir -p /mnt/data
   sudo chmod 777 /mnt/data
   
   # Verify
   ls -la /mnt/data
   ```

6. **Create GHCR secret on k3s cluster** (on control plane)
   ```bash
   kubectl create namespace fullstack-app --dry-run=client -o yaml | kubectl apply -f -
   
   kubectl create secret docker-registry ghcr-secret \
     --docker-server=ghcr.io \
     --docker-username=<YOUR_GITHUB_USERNAME> \
     --docker-password=<YOUR_GITHUB_PAT> \
     --namespace=fullstack-app
   ```

7. **Initial deployment setup** (on control plane)
   ```bash
   kubectl apply -f deployment/pv.yaml
   kubectl apply -f deployment/pvc.yaml
   kubectl apply -f deployment/configmap.yaml
   kubectl apply -f deployment/secrets.yaml
   ```

---

### **Phase 4: GitHub Actions Runner Setup** (~15 minutes)

8. **Install self-hosted runner on k3s control plane**
   ```bash
   mkdir -p /home/ubuntu/actions-runner && cd /home/ubuntu/actions-runner
   curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/download/v2.114.1/actions-runner-linux-x64.tar.gz
   tar xzf actions-runner-linux-x64.tar.gz
   
   # Get token from: Your Repo → Settings → Actions → Runners → New self-hosted runner
   ./config.sh --url https://github.com/<OWNER>/<REPO> --token <TOKEN>
   # When prompted for name, enter: k3s-runner
   # Press Enter for other options
   
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

9. **Verify runner is online**
   - Go to: Your Repo → Settings → Actions → Runners
   - You should see a green dot next to your runner

---

### **Phase 5: Trigger First Deployment** (~5 minutes)

10. **Push a small change to trigger the workflow**
    ```bash
    # Make a small change
    echo "# Automated deployment ready" >> README.md
    
    git add README.md
    git commit -m "Trigger first automated deployment"
    git push origin main
    ```

11. **Monitor the deployment**
    - Go to: Your Repo → Actions
    - Click the latest workflow run
    - Watch "Build and Deploy to k3s" execute
    - Stages:
      - `build-and-push` (~5-10 min): Builds images
      - `deploy-to-k3s` (~2 min): Deploys to cluster

12. **Verify pods are running**
    ```bash
    kubectl get pods -n fullstack-app
    kubectl get svc -n fullstack-app
    ```

---

## 📊 Architecture Diagram

```
Your Laptop (Hyper-V)
├── Control Plane (Ubuntu VM)
│   ├── k3s control plane
│   ├── GitHub Actions self-hosted runner
│   └── /mnt/data (storage)
│
├── Worker 1 (Ubuntu VM)
│   ├── k3s worker
│   └── /mnt/data (storage)
│
└── Worker 2 (Ubuntu VM)
    ├── k3s worker
    └── /mnt/data (storage)

GitHub Cloud
├── Your Repository (main branch)
├── GitHub Container Registry (GHCR)
│   ├── ghcr.io/.../backend:latest
│   ├── ghcr.io/.../frontend:latest
│   └── (tags with git SHA)
└── Actions Workflow
    ├── Build stage
    └── Deploy stage (triggers self-hosted runner)
```

**Data Flow:**
```
Code push → GitHub Actions builds images → GHCR
                                             ↓
                            Self-hosted runner (on k3s control plane)
                                             ↓
                             kubectl applies new deployments
                                             ↓
                             k3s pulls images from GHCR using ghcr-secret
                                             ↓
                            Pods restart with new code ✅
```

---

## 🔑 Key Files Reference

```
.github/
├── workflows/
│   └── deploy.yml                    # GitHub Actions automation ⭐

deployment/
├── backend-deployment.yaml           # Updated with GHCR ⭐
├── frontend-deployment.yaml          # Updated with GHCR ⭐
├── db-deployment.yaml
├── configmap.yaml
├── secrets.yaml
├── pv.yaml
├── pvc.yaml

Documentation/
├── GHCR_SETUP.md                     # Detailed 7-step setup guide
├── QUICKSTART.md                     # Quick reference
├── TROUBLESHOOTING.md                # Common issues & fixes

Scripts/
├── deploy-k3s                        # Manual deployment script
```

---

## ✅ Pre-Deployment Checklist

Before pushing code, make sure:

- [ ] You've updated `<YOUR_GITHUB_USERNAME>` in backend-deployment.yaml and frontend-deployment.yaml
- [ ] Workflow file exists: `.github/workflows/deploy.yml`
- [ ] GitHub PAT created with `read:packages` and `write:packages` scopes
- [ ] k3s cluster is running (3 nodes)
- [ ] `/mnt/data` created on all worker nodes with proper permissions
- [ ] `kubectl` configured on control plane
- [ ] GitHub Actions self-hosted runner installed and running (green dot in Settings)
- [ ] GHCR secret created on k3s cluster
- [ ] PV, PVC, ConfigMap, and Secrets applied to cluster

---

## 🚀 What Happens After You Push

1. **GitHub detects push to main**
2. **Workflow starts automatically**
3. **Build stage (5-10 min):**
   - Builds your backend and frontend Docker images
   - Pushes to GHCR with `:latest` and `:git-sha` tags
4. **Deploy stage (2 min):**
   - Self-hosted runner pulls new images
   - Updates k8s deployments with new image SHA
   - kubectl applies changes
   - Pods restart with new code
5. **Result:**
   - Your app is live on k3s ✅
   - Accessible at `http://<control-plane-ip>:30000`

---

## 📞 If Something Goes Wrong

1. Check GitHub Actions logs: Your Repo → Actions → click workflow run
2. Check k3s pods: `kubectl get pods -n fullstack-app`
3. Check pod details: `kubectl describe pod <name> -n fullstack-app`
4. See `TROUBLESHOOTING.md` for detailed debugging steps
5. See `QUICKSTART.md` for quick fixes table

---

## 🎓 Learning Resources

The workflow demonstrates:
- ✅ Docker multi-stage builds (already in your Dockerfiles)
- ✅ GitHub Actions CI/CD pipeline
- ✅ Container registry integration (GHCR)
- ✅ Kubernetes automated deployments
- ✅ Self-hosted runner setup
- ✅ Image pull secret management
- ✅ GitOps principles

---

**You're ready to automate! 🎉**

Next step: Follow Phase 1 (Local Preparation) above.
