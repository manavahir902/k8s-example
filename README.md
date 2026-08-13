# myapp — GitHub → Kubernetes CI/CD demo

Simple Node app that redeploys to your 3-node minikube cluster on every push to `main`.

## How it works

1. You push code to GitHub.
2. A **self-hosted GitHub Actions runner** (installed on your EC2 box, right next to minikube) picks up the job.
3. It builds the Docker image, loads it directly into minikube (no external registry needed), and updates the Kubernetes Deployment.
4. The Deployment has `replicas: 3` + pod anti-affinity, so Kubernetes spreads one pod per node — you'll see one on `minikube`, one on `minikube-m02`, one on `minikube-m03`.

## One-time setup on your EC2 box

### 1. Push this repo to GitHub
```bash
cd myapp
git init
git add .
git commit -m "initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USER/YOUR_REPO.git
git push -u origin main
```

### 2. Register a self-hosted runner
GitHub repo → **Settings → Actions → Runners → New self-hosted runner**, then on your EC2 box:
```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz
tar xzf actions-runner-linux-x64.tar.gz
./config.sh --url https://github.com/YOUR_USER/YOUR_REPO --token YOUR_TOKEN
sudo ./svc.sh install
sudo ./svc.sh start
```
This runs as a background service, listening for jobs from your repo.

**Important:** the runner service user needs permission to run `docker`, `minikube`, and `kubectl`. Easiest fix — install the runner as your `ubuntu` user (already in the `docker` group) rather than root:
```bash
# if svc.sh install prompts for a user, choose your regular non-root user
```

### 3. First deploy
Just push to `main` — the workflow will build the image, load it into minikube, and `kubectl apply` the deployment (creating it the first time, updating it every time after).

## Verifying it worked

```bash
kubectl get pods -o wide -l app=myapp
# you should see 3 pods, one per node

kubectl get deployment myapp
kubectl rollout history deployment/myapp
```

### Test the app
```bash
minikube service myapp-service --url
curl $(minikube service myapp-service --url)
# repeat a few times — you'll see different pod hostnames in the response,
# proving the traffic is load-balanced across all 3 pods
```

## Making a change

Edit `server.js`, commit, push to `main`. Watch the Actions tab on GitHub — the runner builds and rolls out automatically. Run `kubectl get pods -w` on the EC2 box to watch pods roll over live.
