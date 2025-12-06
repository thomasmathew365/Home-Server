# Home Server k3s Cluster

A 2-node k3s cluster setup for home server applications, managed with ArgoCD.

## Cluster Setup

### Hardware
- **Laptop**: Primary node running RustDesk server
- **Desktop**: Secondary node

### Components
- **k3s**: Lightweight Kubernetes distribution
- **ArgoCD**: GitOps continuous delivery tool
- **RustDesk Server**: Self-hosted remote desktop solution

## Applications

### RustDesk Server
Self-hosted remote desktop server running on the laptop node, allowing secure remote access from anywhere.

**Features:**
- Self-hosted alternative to TeamViewer/AnyDesk
- End-to-end encryption
- Low latency remote desktop access
- Cross-platform client support

## Prerequisites

- Git installed on your local machine
- GitHub account
- Both laptop and desktop running Linux
- Basic terminal/command line knowledge

## Complete Setup Guide (Beginner-Friendly)

This guide will walk you through every step from scratch. Follow the sections in order.

---

## Step 1: Push Project to GitHub

### 1.1 Create a GitHub Repository

1. Go to [https://github.com](https://github.com) and sign in
2. Click the **+** icon in the top right, select **New repository**
3. Repository name: `home-server`
4. Description: `k3s home server cluster with ArgoCD`
5. Select **Public** or **Private** (your choice)
6. **Do NOT** check "Initialize with README" (we already have one)
7. Click **Create repository**

### 1.2 Initialize Git and Push to GitHub

In your terminal, navigate to this project directory and run:

```bash
cd /home/tom/Projects/home-server

# Initialize git repository
git init

# Add all files
git add .

# Create your first commit
git commit -m "Initial commit: RustDesk server setup"

# Add GitHub as remote (replace YOUR_USERNAME with your GitHub username)
git remote add origin https://github.com/YOUR_USERNAME/home-server.git

# Push to GitHub
git branch -M main
git push -u origin main
```

### 1.3 Update ArgoCD Configuration

After pushing to GitHub, edit the file `argocd/rustdesk-app.yaml` and replace the `repoURL` with your actual repository URL:

```yaml
repoURL: https://github.com/YOUR_USERNAME/home-server.git
```

Then commit and push this change:

```bash
git add argocd/rustdesk-app.yaml
git commit -m "Update ArgoCD repo URL"
git push
```

---

## Step 2: Install k3s on Both Machines

### 2.1 Install k3s on Laptop (Master Node)

On your **laptop**, open a terminal and run:

```bash
# Install k3s as master node
curl -sfL https://get.k3s.io | sh -

# Wait for k3s to be ready (takes 30-60 seconds)
sudo systemctl status k3s

# Verify k3s is running
sudo k3s kubectl get nodes
```

You should see your laptop node listed with status "Ready".

### 2.2 Get Laptop IP Address and Node Token

Still on the **laptop**, get the information needed for the desktop:

```bash
# Get laptop IP address (look for your local network IP, usually 192.168.x.x)
ip addr show

# Get the node token (save this for the next step)
sudo cat /var/lib/rancher/k3s/server/node-token
```

**Save both values:**
- Laptop IP: `192.168.x.x` (example)
- Node token: Long string like `K10abc123...`

### 2.3 Install k3s on Desktop (Worker Node)

On your **desktop**, open a terminal and run:

```bash
# Replace <LAPTOP_IP> with your laptop's IP address
# Replace <NODE_TOKEN> with the token from the previous step
curl -sfL https://get.k3s.io | K3S_URL=https://<LAPTOP_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -

# Example:
# curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.100:6443 K3S_TOKEN=K10abc123... sh -
```

### 2.4 Verify the Cluster

Back on the **laptop**, verify both nodes are connected:

```bash
sudo k3s kubectl get nodes
```

You should see both laptop and desktop nodes listed with status "Ready".

### 2.5 Setup kubectl Access

For easier access without sudo, on the **laptop**:

```bash
# Create kubeconfig directory
mkdir -p ~/.kube

# Copy k3s config
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# Fix permissions
sudo chown $USER:$USER ~/.kube/config

# Test kubectl (no sudo needed now)
kubectl get nodes
```

---

## Step 3: Label the Laptop Node

We need to label the laptop node so RustDesk runs on it specifically:

```bash
# First, get the exact name of your laptop node
kubectl get nodes

# Label the laptop node (replace <laptop-node-name> with actual name from above)
kubectl label nodes <laptop-node-name> workload=rustdesk

# Verify the label
kubectl get nodes --show-labels | grep workload=rustdesk
```

---

## Step 4: Install ArgoCD

### 4.1 Install ArgoCD in the Cluster

On the **laptop**, run:

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready (takes 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 4.2 Access ArgoCD UI

```bash
# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

Now open your browser and go to: `https://localhost:8080`

(Accept the self-signed certificate warning)

### 4.3 Login to ArgoCD

**Username:** `admin`

**Password:** Get it by running:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Copy the password and login to ArgoCD UI.

### 4.4 (Optional) Install ArgoCD CLI

For easier command-line management:

```bash
# Download ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Install it
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Remove download
rm argocd-linux-amd64

# Login via CLI
argocd login localhost:8080 --username admin --insecure
```

---

## Step 5: Deploy RustDesk Server with ArgoCD

### 5.1 Create the ArgoCD Application

```bash
# Apply the ArgoCD application manifest
kubectl apply -f argocd/rustdesk-app.yaml
```

### 5.2 Watch ArgoCD Deploy RustDesk

In the ArgoCD UI (`https://localhost:8080`), you should see:
- A new application called **"rustdesk"**
- It will automatically sync and deploy all resources
- Watch it create the namespace, deployments, and services

Or via CLI:
```bash
# Watch the sync status
kubectl get applications -n argocd

# Get detailed status
argocd app get rustdesk
```

### 5.3 Verify RustDesk is Running

```bash
# Check if RustDesk pods are running
kubectl get pods -n rustdesk

# You should see:
# rustdesk-hbbs-xxxxx  1/1  Running
# rustdesk-hbbr-xxxxx  1/1  Running

# Check services
kubectl get svc -n rustdesk
```

---

## Step 6: How ArgoCD Auto-Sync Works (GitOps Workflow)

### Understanding ArgoCD GitOps

ArgoCD continuously monitors your Git repository and automatically applies changes to your cluster. Here's how it works:

**The GitOps Loop:**
1. You make changes to YAML files locally
2. You commit and push to GitHub
3. ArgoCD detects the changes (polls every 3 minutes by default)
4. ArgoCD automatically syncs the changes to your cluster
5. Your cluster now matches what's in Git

### Making Changes to RustDesk

Let's say you want to change the number of replicas:

```bash
# 1. Edit the deployment file
nano manifests/rustdesk/deployment.yaml

# Change replicas: 1 to replicas: 2 for hbbs deployment

# 2. Commit the change
git add manifests/rustdesk/deployment.yaml
git commit -m "Scale hbbs to 2 replicas"

# 3. Push to GitHub
git push

# 4. Wait 3 minutes OR manually trigger sync
argocd app sync rustdesk

# 5. Verify the change
kubectl get deployments -n rustdesk
```

### ArgoCD Auto-Sync Configuration

In our `argocd/rustdesk-app.yaml`, we configured:

```yaml
syncPolicy:
  automated:
    prune: true        # Delete resources removed from git
    selfHeal: true     # Undo manual changes to keep in sync with git
    allowEmpty: false  # Don't delete everything if git is empty
```

This means:
- **Automatic sync**: Changes in Git are applied automatically
- **Self-heal**: If someone manually edits resources in cluster, ArgoCD reverts them back to Git state
- **Prune**: If you delete a YAML file from Git, ArgoCD deletes the resource from cluster

### Checking Sync Status

```bash
# Via kubectl
kubectl get applications -n argocd

# Via ArgoCD CLI
argocd app list
argocd app get rustdesk

# In the UI
# Go to https://localhost:8080 and click on the rustdesk application
```

---

## Step 7: Connect to RustDesk Server

### 7.1 Get Your Laptop's IP Address

On the **laptop**, get the IP address that clients will use to connect:

```bash
# Get laptop IP address (look for your local network IP, usually 192.168.x.x)
hostname -I | awk '{print $1}'
```

### 7.2 Download RustDesk Client

On the **device you want to control your laptop from**:

1. Go to [https://rustdesk.com/](https://rustdesk.com/)
2. Download the RustDesk client for your operating system (Windows/Mac/Linux/Mobile)
3. Install and open RustDesk

### 7.3 Configure RustDesk Client

In the RustDesk client:

1. Click the **menu icon** (three dots) or **Settings**
2. Go to **Network** settings
3. Enable **"Unlock security settings"** or **"Custom server"**
4. Configure:
   - **ID Server**: `<LAPTOP_IP>:31116`
   - **Relay Server**: `<LAPTOP_IP>:31117`
   - **API Server**: Leave blank or use `<LAPTOP_IP>:31116`
   - **Key**: Leave blank (optional encryption key)

Example if your laptop IP is `192.168.1.100`:
- ID Server: `192.168.1.100:31116`
- Relay Server: `192.168.1.100:31117`

5. Click **OK** or **Apply**

### 7.4 Get Your Public Key (Optional)

For added security, you can get your RustDesk server's public key:

```bash
# On the laptop, read the public key from the hbbs container
kubectl exec -n rustdesk deploy/rustdesk-hbbs -- cat /data/id_ed25519.pub
```

Enter this key in the RustDesk client settings under **Key** field.

### 7.5 Connect!

Now you can use RustDesk to connect to any device that has RustDesk client installed and is connected to your custom server!

**Note:** For connections from outside your home network, you'll need to:
- Forward ports 31116 and 31117 on your router to your laptop
- Use your public IP address instead of local IP

---

## Accessing Services

### RustDesk Server
The RustDesk server is accessible via NodePort on the laptop:
- **ID Server (hbbs)**: `<LAPTOP_IP>:31116` (TCP/UDP)
- **Relay Server (hbbr)**: `<LAPTOP_IP>:31117` (TCP)
- **NAT traversal**: Port 31115 (TCP)
- **Web console**: `<LAPTOP_IP>:31118` (TCP)

### ArgoCD UI
Access the ArgoCD dashboard:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Then open: `https://localhost:8080`

**Default username:** `admin`

**Get password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## Repository Structure

```
.
├── README.md
├── argocd/
│   └── rustdesk-app.yaml          # ArgoCD Application for RustDesk
└── manifests/
    └── rustdesk/
        ├── namespace.yaml          # RustDesk namespace
        ├── deployment.yaml         # RustDesk server deployments
        └── service.yaml            # RustDesk services
```

---

## Troubleshooting

### RustDesk pods not starting

```bash
# Check pod status and events
kubectl get pods -n rustdesk
kubectl describe pod -n rustdesk <pod-name>

# Check if the laptop node has the correct label
kubectl get nodes --show-labels | grep workload=rustdesk
```

### Can't connect to RustDesk server

1. Verify pods are running:
   ```bash
   kubectl get pods -n rustdesk
   ```

2. Check services are exposed:
   ```bash
   kubectl get svc -n rustdesk
   ```

3. Test connectivity from laptop:
   ```bash
   # Test if ports are listening
   sudo netstat -tulpn | grep -E '31116|31117'
   ```

4. Check firewall (if enabled):
   ```bash
   # Allow RustDesk ports through firewall
   sudo ufw allow 31115:31119/tcp
   sudo ufw allow 31116:31117/udp
   ```

### ArgoCD not syncing changes

1. Check ArgoCD application status:
   ```bash
   kubectl get applications -n argocd
   argocd app get rustdesk
   ```

2. Manually trigger sync:
   ```bash
   argocd app sync rustdesk
   ```

3. Check ArgoCD logs:
   ```bash
   kubectl logs -n argocd deployment/argocd-application-controller
   ```

### Need to reset everything

```bash
# Delete RustDesk application from ArgoCD
kubectl delete -f argocd/rustdesk-app.yaml

# Delete RustDesk namespace
kubectl delete namespace rustdesk

# Reapply
kubectl apply -f argocd/rustdesk-app.yaml
```

---

## Next Steps

Now that you have RustDesk running, you can:

1. Add more applications to your home server (Jellyfin, Home Assistant, etc.)
2. Set up ingress controller for easier access to services
3. Configure SSL certificates with cert-manager
4. Set up persistent storage with Longhorn or NFS
5. Add monitoring with Prometheus and Grafana

Check out the [Awesome Home Kubernetes](https://github.com/k8s-at-home/awesome-home-kubernetes) repository for more ideas!
