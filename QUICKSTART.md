# GitHub Actions + GHCR + k3s: Quick Start Reference

## 🎯 Quick Setup (5-10 minutes)

### Step 1: Update your repo username
```bash
cd deployment/
sed -i 's/yourusername/<YOUR_GITHUB_USERNAME>/g' backend-deployment.yaml frontend-deployment.yaml
git add .
git commit -m "Update image references"
git push
```

### Step 2: Setup GitHub Actions Runner (on k3s control plane)
```bash
mkdir -p /home/ubuntu/actions-runner && cd /home/ubuntu/actions-runner
curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/download/v2.114.1/actions-runner-linux-x64.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

# Get token from: GitHub > Your Repo > Settings > Actions > Runners > New self-hosted runner
./config.sh --url https://github.com/<OWNER>/<REPO> --token <TOKEN>
# Enter name: k3s-runner
# Press Enter for other options

sudo ./svc.sh install
sudo ./svc.sh start
```

### Step 3: Create GitHub PAT
1. GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic)
2. Generate new token with: `read:packages`, `write:packages`
3. Copy the token

### Step 4: Create k3s GHCR Secret
```bash
kubectl create namespace fullstack-app --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<GITHUB_USERNAME> \
  --docker-password=<GITHUB_PAT> \
  --namespace=fullstack-app
```

### Step 5: Prepare storage on worker nodes
```bash
# SSH to each worker node and run:
sudo mkdir -p /mnt/data
sudo chmod 777 /mnt/data
```

### Step 6: Initial deployment setup (on control plane)
```bash
kubectl apply -f deployment/pv.yaml
kubectl apply -f deployment/pvc.yaml
kubectl apply -f deployment/configmap.yaml
kubectl apply -f deployment/secrets.yaml
```

### Step 7: Trigger deployment
```bash
git add .
git commit -m "Ready for automated deployment"
git push origin main
```

**Watch it go:**
- GitHub repo > Actions
- See "Build and Deploy to k3s" workflow running
- Check deployment status:
  ```bash
  kubectl get pods -n fullstack-app
  kubectl get svc -n fullstack-app
  ```

---

## 📋 File Reference

| File | Purpose | Update? |
|------|---------|---------|
| `.github/workflows/deploy.yml` | GitHub Actions automation | ✅ Create (done) |
| `deployment/backend-deployment.yaml` | Backend deployment | ✅ Update image references |
| `deployment/frontend-deployment.yaml` | Frontend deployment | ✅ Update image references |
| `deployment/db-deployment.yaml` | MongoDB | ✅ No changes needed |
| `deployment/configmap.yaml` | Environment variables | ✅ No changes needed |
| `deployment/secrets.yaml` | MongoDB URI | ✅ No changes needed |
| `deployment/pv.yaml` | Persistent storage | ✅ No changes needed |
| `deployment/pvc.yaml` | Storage claim | ✅ No changes needed |
| `GHCR_SETUP.md` | Detailed setup guide | 📖 Reference |
| `TROUBLESHOOTING.md` | Problem solving | 🔧 Reference |

---

## 🔄 How It Works (Automated Flow)

```
1. Developer: git push to main
        ↓
2. GitHub Actions: Triggers workflow
        ↓
3. Build Stage: Builds images, pushes to GHCR
   - ghcr.io/username/repo/backend:sha123
   - ghcr.io/username/repo/frontend:sha123
        ↓
4. Deploy Stage: Self-hosted runner pulls new images
   - Updates deployment with new image tags
   - k3s pulls images using ghcr-secret
   - Pods restart with new code
        ↓
5. Result: New version live on k3s
```

---

## ✅ Verification Checklist

```bash
# 1. Runner is online
# GitHub: repo > Settings > Actions > Runners (should show green dot)

# 2. GHCR secret exists on k3s
kubectl get secrets -n fullstack-app ghcr-secret

# 3. Images pushed to GHCR
# GitHub: repo > Packages (should show backend and frontend)

# 4. Deployment is running
kubectl get pods -n fullstack-app
# Should see pods in Running state (not Pending/CrashLoopBackOff)

# 5. Services are ready
kubectl get svc -n fullstack-app
# Should have backend-service (ClusterIP) and frontend-service (NodePort:30000)
```

---

## 🚀 Deployment Workflow Commands

```bash
# Manual deployment (no GitHub Actions)
bash deploy-k3s

# Check everything is running
kubectl get all -n fullstack-app

# View logs
kubectl logs -n fullstack-app -l app=backend -f
kubectl logs -n fullstack-app -l app=frontend -f
kubectl logs -n fullstack-app -l app=mongodb -f

# Restart a deployment
kubectl rollout restart deployment/backend-deployment -n fullstack-app

# Access frontend
kubectl get svc frontend-service -n fullstack-app
# NodePort will be shown (default 30000)
# Access at: http://<your-control-plane-ip>:30000

# Delete everything
kubectl delete namespace fullstack-app
```

---

## 🔐 Security Notes

- ✅ GHCR is private (only you and GitHub see images)
- ✅ GitHub PAT scoped to only `read:packages` and `write:packages`
- ✅ Runner credentials stay on your network (not exposed to GitHub)
- ✅ All pulls require `ghcr-secret` authentication
- ⚠️ Don't commit secrets to git
- ⚠️ Don't share GitHub PAT or runner registration tokens

---

## 🆘 Quick Fixes

| Problem | Fix |
|---------|-----|
| Pods in ImagePullBackOff | Check ghcr-secret exists: `kubectl get secrets -n fullstack-app` |
| Pods in Pending | Check /mnt/data exists on worker nodes |
| Backend can't reach MongoDB | Check MongoDB pod is running: `kubectl get pods -n fullstack-app -l app=mongodb` |
| New code not deployed | Check GitHub Actions workflow passed: `repo > Actions > workflow run` |
| Runner offline | SSH to runner machine, check: `sudo ./svc.sh status` |

---

## 📞 Get Help

1. Check `TROUBLESHOOTING.md` for detailed debugging
2. Check GitHub Actions logs for build errors
3. Use `kubectl logs` and `kubectl describe pod` for k3s issues
4. Collect diagnostics:
   ```bash
   kubectl describe pod -n fullstack-app --all=true > pods.txt
   kubectl get events -n fullstack-app > events.txt
   kubectl logs -n fullstack-app --all-containers=true > logs.txt
   ```

---

## ⚡ Performance Tips

- First push will take ~5-10 minutes (full build)
- Subsequent pushes: ~2-3 minutes (layer caching)
- k3s is very efficient (uses ~1-2GB RAM per machine)
- Images are cached in GHCR for faster pulls

---

**You're all set! 🎉 Push code, and your app automatically deploys to k3s!**
