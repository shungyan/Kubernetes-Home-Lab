# K3s Cluster Setup 

This repo provides step-by-step instructions to set up a lightweight Kubernetes cluster using K3s, deploy ArgoCD for GitOps, configure an ArgoCD application to sync with a GitHub repository different applications.

## Prerequisites
- Two nodes (e.g., VMs or servers): one for the K3s master and one for the worker.
- Ubuntu 22.04 or later (or a compatible Linux distribution) on both nodes.
- SSH access to both nodes.
- `curl`, `git`, and `helm` installed on your local machine or nodes.

## Step 1: Set Up K3s Cluster
### 1.1 Install K3s on the Master Node
1. SSH into the master node.
2. Install K3s with the following command:
   ```bash
   curl -sfL https://get.k3s.io | sh -
   ```
3. Verify the installation:
   ```bash
   kubectl get nodes
   ```
4. Copy the K3s token for the worker node:
   ```bash
   sudo cat /var/lib/rancher/k3s/server/node-token
   ```

### 1.2 Install K3s on the Worker Node
1. SSH into the worker node.
2. Install K3s as an agent, using the master node's IP and token:
   ```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
   ```
3. On the master node, verify the worker node has joined:
   ```bash
   kubectl get nodes
   ```

## Step 2: Install ArgoCD
1. On the master node, create a namespace for ArgoCD:
   ```bash
   kubectl create namespace argocd
   ```
2. Apply the ArgoCD installation manifest:
   ```bash
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
3. Verify ArgoCD pods are running:
   ```bash
   kubectl get pods -n argocd
   ```
4. Access the ArgoCD UI by port-forwarding:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
5. Retrieve the initial admin password:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
   ```
6. Open a browser and navigate to `http://localhost:8080`. Log in with username `admin` and the retrieved password.

## Step 3: Create ArgoCD Application to Sync with GitHub Repo
1. Ensure your GitHub repository contains the Nginx manifests in the `nginx-app/` directory.
2. Log in to the ArgoCD UI or use the ArgoCD CLI.
3. Create an ArgoCD application manifest (`nginx-app.yaml`) or apply it directly:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: nginx-app
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: https://github.com/<YOUR_USERNAME>/<YOUR_REPO>.git
       targetRevision: HEAD
       path: nginx-app
     destination:
       server: https://kubernetes.default.svc
       namespace: nginx
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
   ```
4. Apply the application manifest:
   ```bash
   kubectl apply -f nginx-app.yaml
   ```
5. Verify the application in the ArgoCD UI or with:
   ```bash
   argocd app get nginx-app
   ```

## Step 4: Install Longhorn Using Helm
1. Add the Longhorn Helm repository:
   ```bash
   helm repo add longhorn https://charts.longhorn.io
   helm repo update
   ```
2. Create a namespace for Longhorn:
   ```bash
   kubectl create namespace longhorn-system
   ```
3. Install Longhorn:
   ```bash
   helm install longhorn longhorn/longhorn --namespace longhorn-system
   ```
4. Verify Longhorn is running:
   ```bash
   kubectl get pods -n longhorn-system
   ```
5. Access the Longhorn UI by port-forwarding:
   ```bash
   kubectl port-forward svc/longhorn-frontend -n longhorn-system 8081:80
   ```
6. Open a browser and navigate to `http://localhost:8081`.

## Troubleshooting
- Ensure nodes have sufficient resources (CPU, memory, disk).
- Check pod logs for errors: `kubectl logs <pod-name> -n <namespace>`.
- Verify ArgoCD sync status in the UI or with `argocd app list`.
- For Longhorn, ensure storage requirements are met (e.g., available disk space).
