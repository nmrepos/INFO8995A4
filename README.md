# INFO8995A3 Assignment 3 - Orchestration with Gitea

## Overview

This project demonstrates deploying Gitea on Kubernetes using Ansible and Helm, with CI/CD and public exposure via ngrok. It supports both local (sqlite) and production (mysql) database backends.

---

## Repository Structure

- `gitea/up.yml`: Ansible playbook to deploy Gitea via Helm
- `gitea/values.yaml`: Helm values (default: MySQL production mode with persistence)
- `gitea/ci-cd-pipeline-example.yml`: Example Gitea Actions workflow
- `ngrok/up.yaml`: Kubernetes manifest to expose Gitea via ngrok
- `k8s/`: Additional k3d/k3s cluster setup scripts

---

## Prerequisites

- Python 3.x
- Ansible
- Kubernetes cluster (e.g., Minikube, k3d, or k3s)
- kubectl
- Helm
- ngrok account (for public exposure)

---

## ngrok Token Setup

**Important:** the ngrok authtoken is not included in this repository.

Before running the ngrok manifest, you must add your own ngrok authtoken:

1. Get your authtoken from https://dashboard.ngrok.com/get-started/your-authtoken
2. Base64-encode your token:
   ```bash
   echo -n 'YOUR_NGROK_AUTHTOKEN' | base64
   ```
3. Edit `ngrok/up.yaml` and replace `REPLACE_WITH_BASE64_NGROK_TOKEN` with your base64-encoded token in the Secret section.

---
## Quick Start (Production Mode - Default)

```bash
pip install ansible kubernetes
git submodule update --init --recursive
minikube start  # or your preferred k8s cluster
ansible-playbook gitea/up.yml
kubectl get pods
kubectl port-forward svc/gitea-http 3000:3000

ansible-playbook gitea/up.yml
kubectl get pods
kubectl port-forward svc/gitea-http 3000:3000
```

Visit [http://localhost:3000](http://localhost:3000) to access Gitea.

> **Note:** The default configuration now uses MySQL database and persistent storage for production readiness.

---

## Exposing Gitea Publicly with ngrok

1. **Apply the ngrok manifest:**
   ```bash
   kubectl apply -f ngrok/up.yaml
   ```

2. **Get the ngrok pod logs to find your public URL:**
   ```bash
   kubectl logs -n ngrok -l app=ngrok
   ```
   Look for a line starting with `https://...ngrok-free.app`.

   > If you see an error about too many agent sessions, end old sessions in your ngrok dashboard and delete error pods.

3. **Open the ngrok URL in your browser to access Gitea from anywhere.**

### Managing ngrok Free Tier

Due to free tier limitations, you may need to manage the tunnel periodically:

**Check tunnel status:**
```bash
kubectl get pods -n ngrok
kubectl logs -n ngrok -l app=ngrok --tail=10
```

**Restart tunnel (after 2-hour limit or to get new URL):**
```bash
kubectl delete pod -n ngrok -l app=ngrok
# Wait for pod to restart automatically
kubectl logs -n ngrok -l app=ngrok | grep "started tunnel"
```

**Clean up ngrok deployment:**
```bash
kubectl delete -f ngrok/up.yaml
```

---

## CI/CD Example

- See `gitea/ci-cd-pipeline-example.yml` for a sample Gitea Actions workflow.
- Place this file in your Gitea repo as `.gitea/workflows/ci.yml`.

---

## Troubleshooting

### ngrok Issues

**"Account limit: 1 tunnel session"**
- Only one tunnel can be active per free account
- Check your ngrok dashboard for other active tunnels
- Terminate other tunnels or delete the pod and restart

**"Session expired" or tunnel stops working**
- Free tier has 2-hour session limits
- Restart the ngrok pod to get a new session and URL

**Can't access Gitea through ngrok URL**
- Verify Gitea is running: `kubectl get pods -l app.kubernetes.io/name=gitea`
- Check ngrok logs: `kubectl logs -n ngrok -l app=ngrok`
- Ensure the URL is correct and starts with `https://`

### Database/Persistence Issues

**Gitea won't start with MySQL**
- Check if MySQL pod is ready: `kubectl get pods -l app.kubernetes.io/name=mysql`
- Verify database connection in Gitea logs: `kubectl logs -l app.kubernetes.io/name=gitea`

**Lost data after pod restart**
- Verify persistent volumes: `kubectl get pv`
- Check if persistence is enabled in values.yaml

---

## Production Mode

For production deployment with persistent storage and external MySQL database:

### Option 1: Quick Production Setup (Recommended)
The repository is already configured for production mode in `gitea/values.yaml`. Simply run:

```bash
ansible-playbook gitea/up.yml
```

This will deploy Gitea with:
- **MySQL database** for persistent data storage
- **Persistent volumes** for repository data (10Gi)
- **Redis-based** session and cache management

### Option 2: Development Mode (SQLite)
For development/testing with SQLite, create a separate values file:

```bash
cp gitea/values.yaml gitea/values-dev.yaml
```

Edit `gitea/values-dev.yaml` to use SQLite:
```yaml
mysql:
  enabled: false
persistence:
  enabled: false
gitea:
  config:
    database:
      DB_TYPE: sqlite3
    session:
      PROVIDER: memory
    cache:
      ADAPTER: memory
```

Then deploy with:
```bash
ansible-playbook gitea/up.yml -e values_file=values-dev.yaml
```

### Verifying Production Setup
After deployment, verify persistent storage and database:

```bash
# Check if MySQL pod is running
kubectl get pods -l app.kubernetes.io/name=mysql

# Check persistent volumes
kubectl get pv

# Verify Gitea database connection
kubectl logs -l app.kubernetes.io/name=gitea | grep -i database
```

---


## Step-by-Step Screenshots

Below are screenshots for each major step of the deployment process.

1. **Install dependencies**
   
   ![Install dependencies](screenshots/Screenshot%20(1059).png)

2. **Initialize git submodules & Starting Kubernetes cluster**
   
   ![Initialize submodules & Start cluster](screenshots/Screenshot%20(1060).png)

3. **Deploy Gitea using Ansible**
   
   ![Deploy Gitea](screenshots/Screenshot%20(1061).png)

4. **Check Gitea pods & Port-forward to access Gitea locally**
   
   ![Check pods](screenshots/image.png)

5. **Access Gitea Web UI Locally**
   
   ![Gitea Web UI Locally](screenshots/Screenshot%20(1071).png)

6. **Apply ngrok manifest**
   
   ![Apply ngrok](screenshots/Screenshot%20(1076).png)

7. **Check ngrok agent details**
   
   ![ngrok logs](screenshots/Screenshot%20(1073).png)

8. **Access the ngrok endpoints**
   
   ![ngrok URL](screenshots/Screenshot%20(1074).png)

9. **Access Gitea via ngrok public URL**
   
   ![Gitea UI](screenshots/Screenshot%20(1075).png)

---