# INFO8995A4 Assignment 4 - CI/CD Pipeline with Jenkins and Gitea

## Overview

This project demonstrates deploying both Gitea and Jenkins on Kubernetes with a complete CI/CD pipeline integration. Jenkins automatically builds and deploys applications when code is pushed to Gitea repositories, with public exposure via ngrok.

---

## Repository Structure

- `gitea/up.yml`: Ansible playbook to deploy Gitea via Helm
- `gitea/values.yaml`: Helm values (default: MySQL production mode with persistence)
- `jenkins/`: Jenkins deployment manifests and configuration
- `ngrok/up.yaml`: Kubernetes manifest to expose Jenkins via ngrok for webhook access
- `k8s/`: Additional k3d/k3s cluster setup scripts
- `Jenkinsfile`: Example CI/CD pipeline configuration

---

## Prerequisites

- Python 3.x
- Ansible
- Kubernetes cluster (e.g., Minikube, k3d, or k3s)
- kubectl
- Helm
- ngrok account (for public webhook access)

---

## ngrok Token Setup

**Important:** the ngrok authtoken is not included in this repository.

Before running the ngrok manifest, you must add your own ngrok authtoken:

1. Get your authtoken from https://dashboard.ngrok.com/get-started/your-authtoken
2. Base64-encode your token:
   ```bash
   echo -n 'YOUR_NGROK_AUTHTOKEN' | base64
   ```
3. Edit `ngrok/up.yaml` and replace `YOUR_NGROK_AUTHTOKEN` with your base64-encoded token in the Secret section.

---

## Quick Start (Complete CI/CD Setup)

### 1. Deploy Gitea
```bash
pip install ansible kubernetes
git submodule update --init --recursive
minikube start  # or your preferred k8s cluster
ansible-playbook gitea/up.yml
kubectl get pods
```

### 2. Deploy Jenkins
```bash
# Deploy Jenkins to the cluster
kubectl apply -f jenkins/
kubectl get pods -n jenkins
```

### 3. Setup ngrok for Jenkins webhook access
```bash
# Configure ngrok to expose Jenkins
kubectl apply -f ngrok/up.yaml
kubectl logs -n ngrok deployment/ngrok -f
```

### 4. Access Services
- **Gitea**: `kubectl port-forward svc/gitea-http 3000:3000` → [http://localhost:3000](http://localhost:3000)
- **Jenkins**: Use ngrok URL from logs or `kubectl port-forward -n jenkins svc/jenkins 8080:8080` → [http://localhost:8080](http://localhost:8080)

---

## CI/CD Pipeline Setup

### 1. Configure Jenkins Job

1. Access Jenkins via ngrok URL (e.g., `https://abc123.ngrok-free.app`)
2. Install required plugins:
   - Git plugin
   - Kubernetes plugin
   - Pipeline plugin
3. Create new Pipeline job:
   - **Pipeline from SCM**: Git
   - **Repository URL**: `http://gitea-http.default.svc.cluster.local:3000/your-username/your-repo`
   - **Script Path**: `Jenkinsfile`

### 2. Configure Gitea Webhook

1. In your Gitea repository, go to **Settings** → **Webhooks**
2. Add webhook with:
   ```
   Payload URL: https://your-ngrok-url.ngrok.io/git/notifyCommit?url=http://gitea-http.default.svc.cluster.local:3000/your-username/your-repo
   Content Type: application/json
   Events: Push events
   ```

### 3. Test the Pipeline

```bash
# Make changes to your repository
git add .
git commit -m "Trigger CI/CD pipeline"
git push
```

Jenkins will automatically:
- Detect the push via webhook
- Pull the latest code from Gitea
- Execute the Jenkinsfile pipeline
- Deploy using Kubernetes agents

---

## Architecture

```
┌─────────────┐    webhook    ┌─────────────┐    git clone    ┌─────────────┐
│    Gitea    │──────────────▶│   Jenkins   │────────────────▶│   Gitea     │
│ (Internal)  │               │ (via ngrok) │                 │ (Internal)  │
└─────────────┘               └─────────────┘                 └─────────────┘
                                      │
                                      ▼
                              ┌─────────────┐
                              │ Kubernetes  │
                              │   Agents    │
                              └─────────────┘
```

- **Gitea**: Internal service accessible at `gitea-http.default.svc.cluster.local:3000`
- **Jenkins**: Internal service accessible at `jenkins.jenkins.svc.cluster.local:8080`
- **ngrok**: Exposes Jenkins for external webhook access
- **Webhook flow**: Gitea → ngrok → Jenkins → clone from Gitea → deploy to K8s

---

## Managing ngrok

**Check tunnel status:**
```bash
kubectl get pods -n ngrok
kubectl logs -n ngrok deployment/ngrok --tail=10
```

**Restart tunnel (after 2-hour limit or to get new URL):**
```bash
kubectl delete pod -n ngrok -l app=ngrok
# Wait for pod to restart automatically
kubectl logs -n ngrok deployment/ngrok | grep "started tunnel"
```

**Update webhook URL in Gitea after restart:**
- Get new ngrok URL from logs
- Update webhook configuration in Gitea repository settings

---

### Jenkins Persistence

```yaml
# Add to Jenkins deployment
volumeMounts:
- name: jenkins-home
  mountPath: /var/jenkins_home
volumes:
- name: jenkins-home
  persistentVolumeClaim:
    claimName: jenkins-pvc
```

---

## Example Pipeline Output

```
Generic Cause
Obtained Jenkinsfile from git http://gitea-http.default.svc.cluster.local:3000/nidhun/JenPyCICD.git
[Pipeline] Start of Pipeline
[Pipeline] podTemplate
[Pipeline] {
[Pipeline] node
Created Pod: kubernetes jenkins/cicd-4-b3p0k-slz3n-kdcd4
[PodInfo] jenkins/cicd-4-b3p0k-slz3n-kdcd4
    Container [jnlp] waiting [ContainerCreating] No message
    Container [python] waiting [ContainerCreating] No message
    Pod [Pending][ContainersNotReady] containers with unready status: [python jnlp]
Agent cicd-4-b3p0k-slz3n-kdcd4 is provisioned from template cicd_4-b3p0k-slz3n
```

This shows successful integration where Jenkins:
- Receives webhook from Gitea
- Clones code from internal Gitea service
- Provisions Kubernetes pod agents
- Executes pipeline steps

---

## Screeenshots

[📄 View the Screenshots ](./Group12.pdf)
