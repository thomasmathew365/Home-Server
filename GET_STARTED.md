# Get Started - Your Next Steps

Welcome! This guide will help you get your home server up and running.

## Overview

You now have a complete home server setup with:
- RustDesk server configuration
- Kubernetes (k3s) deployment manifests
- ArgoCD GitOps automation
- Complete documentation

## What You Need to Do (In Order)

### Step 1: Push to GitHub (Do This First!)

1. Create a GitHub repository at [github.com](https://github.com)
   - Name it `home-server`
   - Don't initialize with README

2. Run these commands in this directory:

```bash
cd /home/tom/Projects/home-server

git init
git add .
git commit -m "Initial commit: RustDesk server setup"
git remote add origin https://github.com/YOUR_USERNAME/home-server.git
git branch -M main
git push -u origin main
```

3. Update [argocd/rustdesk-app.yaml](argocd/rustdesk-app.yaml):
   - Change `repoURL: https://github.com/yourusername/home-server.git`
   - To your actual GitHub URL

4. Commit and push the change:
```bash
git add argocd/rustdesk-app.yaml
git commit -m "Update ArgoCD repo URL"
git push
```

### Step 2: Install k3s

**On your laptop (master node):**
```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

Save these values:
```bash
# Your laptop IP
ip -4 addr show | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | grep -v '127.0.0.1' | head -1

# Node token for desktop
sudo cat /var/lib/rancher/k3s/server/node-token
```

**On your desktop (worker node):**
```bash
# Replace <LAPTOP_IP> and <NODE_TOKEN> with values from above
curl -sfL https://get.k3s.io | K3S_URL=https://<LAPTOP_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```

**Back on laptop, setup kubectl:**
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config

# Add KUBECONFIG to your shell configuration
echo 'export KUBECONFIG=~/.kube/config' >> ~/.zshrc  # or ~/.bashrc for bash
source ~/.zshrc  # or source ~/.bashrc

kubectl get nodes  # Should show both nodes
```

### Step 3: Label the Laptop Node

```bash
# Get node name
kubectl get nodes

# Label it (replace <laptop-node-name>)
kubectl label nodes <laptop-node-name> workload=rustdesk
```

### Step 4: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for it to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### Step 5: Access ArgoCD UI

```bash
# Port forward in background
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Get the password

```

Open browser: `https://localhost:8080`
- Username: `admin`
- Password: (from command above)

### Step 6: Deploy RustDesk

```bash
kubectl apply -f argocd/rustdesk-app.yaml
```

Watch it deploy in ArgoCD UI or:
```bash
kubectl get pods -n rustdesk -w
```

### Step 7: Get RustDesk Public Key

Get the RustDesk server's public key (needed for client configuration).

**Method 1: From kubectl logs**
```bash
sudo kubectl -n rustdesk logs deployment/rustdesk-hbbs | grep "Public Key"
```

**Method 2: From the filesystem** (if kubectl doesn't work)
```bash
sudo cat /var/lib/rustdesk/hbbs/id_ed25519.pub
```

This will output something like:
```
Public Key: SsF2YzCVaNjYN5z3qEdiCKmxBh87krAcslsJEmmsguw=
```

Save this key - you'll need it for client configuration.

### Step 8: Connect RustDesk Client

1. Get laptop IP:
```bash
ip -4 addr show | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | grep -v '127.0.0.1' | head -1
```

2. Download RustDesk client from [rustdesk.com](https://rustdesk.com)

3. In client settings, set:
   - ID Server: `<LAPTOP_IP>:31116`
   - Relay Server: `<LAPTOP_IP>:31117`
   - Key: `<PUBLIC_KEY from Step 7>`

4. Done! You can now connect to devices using your self-hosted RustDesk server.

### Step 9: Setup Unattended Access (Optional)

To enable unattended access on remote machines:

1. **Install RustDesk client on the remote machine**

2. **Configure it to run as a service:**
```bash
# Create systemd user service
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/rustdesk.service << 'EOF'
[Unit]
Description=RustDesk Remote Desktop
After=graphical-session.target

[Service]
Type=simple
Environment=DISPLAY=:0
Environment=XAUTHORITY=$HOME/.Xauthority
ExecStart=/usr/bin/rustdesk
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
EOF

# Enable and start the service
systemctl --user daemon-reload
systemctl --user enable rustdesk
systemctl --user start rustdesk
```

3. **Configure RustDesk on the remote machine:**
   - Click the RustDesk icon in system tray
   - Go to Settings → Configure ID/Relay Server
   - Set ID Server: `<LAPTOP_IP>:31116`
   - Set Relay Server: `<LAPTOP_IP>:31117`
   - Set Key: `<PUBLIC_KEY from Step 7>`
   - Go to Security settings
   - Set a permanent password
   - Enable "Direct IP Access" with port **31116**

4. Now you can connect without needing to accept each connection!

## Important Files to Know

- [README.md](README.md) - Complete documentation with detailed explanations
- [QUICK_REFERENCE.md](QUICK_REFERENCE.md) - Quick command reference
- [argocd/rustdesk-app.yaml](argocd/rustdesk-app.yaml) - ArgoCD application (update the repoURL!)
- [manifests/rustdesk/](manifests/rustdesk/) - Kubernetes manifests for RustDesk

## How GitOps Works

After setup, any time you want to make changes:

1. Edit files in `manifests/rustdesk/`
2. `git add .`
3. `git commit -m "description"`
4. `git push`
5. ArgoCD automatically deploys changes (within 3 minutes)

That's it! Git is your source of truth.

## Need Help?

- **Detailed instructions**: See [README.md](README.md)
- **Quick commands**: See [QUICK_REFERENCE.md](QUICK_REFERENCE.md)
- **Troubleshooting**: See README.md > Troubleshooting section

## Estimated Time

- GitHub setup: 5 minutes
- k3s installation: 10 minutes
- ArgoCD installation: 5 minutes
- RustDesk deployment: 5 minutes
- Testing connection: 5 minutes

**Total: ~30 minutes**

Good luck! You've got this!
