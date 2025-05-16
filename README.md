# Local Kubernetes & Argo CD Setup Guide

## Prerequisites Check

### Verify kubectl Installation
```bash
kubectl version --client
```
Install kubectl (macOS):
```bash
brew install kubectl
```

### Minikube Setup
Install Minikube (macOS):
```bash
brew install minikube
```
Verify Minikube Installation:
```bash
minikube version
```
Start Minikube with Docker Driver:
```bash
minikube start --driver=docker
```
Enable Local Registry Addon:
```bash
minikube addons enable registry
```
After running `minikube start`, it configures your kubeconfig file at `~/.kube/config`, and `kubectl` points to the Minikube cluster.

### Verifying Minikube Cluster
Check Current Context:
```bash
kubectl config current-context
```
Should return: `minikube`

List All Contexts:
```bash
kubectl config get-contexts
```

## Install Argo CD on Minikube

### Create Namespace for Argo CD
```bash
kubectl create namespace argocd
```

### Install Argo CD
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Install Argo CD CLI
```bash
brew install argocd
```

### Check Argo CD Pod Status
```bash
kubectl get pods -n argocd
```
Wait until all pods are in the `Running` state.

### Access Argo CD UI
Patch Argo CD Server to LoadBalancer:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
Optional: Port Forwarding:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Access via: `https://localhost:8080`

### Argo CD Login
Retrieve Initial Admin Password:
```bash
argocd admin initial-password -n argocd
```
Login to Argo CD CLI:
```bash
argocd login localhost:8080
```
- Username: `admin`
- Password: (from previous command)

## Local Docker Registry with Minikube

### Use Minikube Docker Daemon
```bash
eval $(minikube docker-env)
```

### Check Docker Info
```bash
docker info
```

## Deploy PostgreSQL Operator

### Apply Postgres Operator
```bash
kubectl apply -k root-app.yaml
```

## Build and Push Docker Image Locally 
```bash
docker build -t localhost:5000/my-image:tag .
docker push localhost:5000/my-image:tag
```
Minikube's local registry runs on `localhost:5000`.


## Deploy Jobs via Argo CD

### `all_jobs.yaml` Example
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
    name: plant-opti
    namespace: argocd
spec:
    project: default
    source:
        repoURL: https://github.com/rohitkhazanchi121/argocd.git
        path: jobs
        targetRevision: HEAD
    destination:
        server: https://kubernetes.default.svc
        namespace: default
    syncPolicy:
        automated:
            selfHeal: true
            prune: true
```

### Apply Argo CD Application
```bash
kubectl apply -f all_jobs.yaml
```

### Job Manifest Example: `datastore-cron.yaml`
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
    name: datastore-cronjob
    namespace: default
spec:
    schedule: "*/30 * * * *"
    concurrencyPolicy: Forbid
    suspend: true
    jobTemplate:
        spec:
            template:
                spec:
                    containers:
                        - name: syntheticdata
                            image: localhost:5000/datastore:tag
                            imagePullPolicy: Always
                            env:
                                - name: PG_DATABASE
                                    value: postgres
                                - name: PG_HOST
                                    value: acid-minimal-cluster.default.svc.cluster.local
                                - name: PG_PORT
                                    value: "5432"
                                - name: PG_USERNAME
                                    valueFrom:
                                        secretKeyRef:
                                            name: postgres.acid-minimal-cluster.credentials.postgresql.acid.zalan.do
                                            key: username
                                - name: PG_PASSWORD
                                    valueFrom:
                                        secretKeyRef:
                                            name: postgres.acid-minimal-cluster.credentials.postgresql.acid.zalan.do
                                            key: password
                    restartPolicy: OnFailure
```
