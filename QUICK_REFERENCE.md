# Quick Reference Guide

This is a cheat sheet for common commands you'll use with your home server.

## Git Commands

```bash
# Check status
git status

# Add files
git add .

# Commit changes
git commit -m "Description of changes"

# Push to GitHub
git push

# Pull latest changes
git pull
```

## kubectl Commands

```bash
# View all nodes
kubectl get nodes

# View all pods in all namespaces
kubectl get pods -A

# View pods in specific namespace
kubectl get pods -n rustdesk

# View services
kubectl get svc -n rustdesk

# View deployments
kubectl get deployments -n rustdesk

# Get detailed info about a pod
kubectl describe pod -n rustdesk <pod-name>

# View pod logs
kubectl logs -n rustdesk <pod-name>

# Follow logs in real-time
kubectl logs -n rustdesk <pod-name> -f

# Execute command in a pod
kubectl exec -n rustdesk <pod-name> -- <command>

# Delete a pod (it will restart)
kubectl delete pod -n rustdesk <pod-name>

# Delete namespace
kubectl delete namespace rustdesk
```

## ArgoCD Commands

```bash
# List all applications
argocd app list

# Get application details
argocd app get rustdesk

# Manually sync an application
argocd app sync rustdesk

# View application logs
argocd app logs rustdesk

# Delete an application
argocd app delete rustdesk

# Refresh application (check git for changes)
argocd app refresh rustdesk
```

## Accessing Services

```bash
# Port forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Get laptop IP
hostname -I | awk '{print $1}'

# Check open ports
sudo netstat -tulpn | grep -E '31116|31117'
```

## RustDesk Specific

```bash
# Get RustDesk pods status
kubectl get pods -n rustdesk

# Get RustDesk public key
kubectl exec -n rustdesk deploy/rustdesk-hbbs -- cat /data/id_ed25519.pub

# Restart RustDesk
kubectl rollout restart deployment -n rustdesk

# View RustDesk logs
kubectl logs -n rustdesk -l app=rustdesk-hbbs
kubectl logs -n rustdesk -l app=rustdesk-hbbr
```

## Common Workflows

### Making a Change to RustDesk Configuration

```bash
# 1. Edit the file
nano manifests/rustdesk/deployment.yaml

# 2. Check what changed
git diff

# 3. Commit and push
git add .
git commit -m "Update RustDesk configuration"
git push

# 4. ArgoCD will auto-sync in ~3 minutes, or manually sync:
argocd app sync rustdesk

# 5. Verify the change
kubectl get deployments -n rustdesk
```

### Troubleshooting a Pod

```bash
# 1. Check pod status
kubectl get pods -n rustdesk

# 2. Get detailed information
kubectl describe pod -n rustdesk <pod-name>

# 3. Check logs
kubectl logs -n rustdesk <pod-name>

# 4. If stuck, delete it (will restart)
kubectl delete pod -n rustdesk <pod-name>
```

### Checking ArgoCD Sync Status

```bash
# Check if application is in sync
kubectl get applications -n argocd

# Get detailed sync status
argocd app get rustdesk

# Force a sync
argocd app sync rustdesk --force
```

## Important Ports

| Service | Port | Protocol | Description |
|---------|------|----------|-------------|
| RustDesk ID Server | 31116 | TCP/UDP | Device registration |
| RustDesk Relay | 31117 | TCP | Connection relay |
| RustDesk NAT | 31115 | TCP | NAT traversal |
| RustDesk Web | 31118 | TCP | Web console |
| ArgoCD UI | 8080 | TCP | Port forwarded |
| k3s API | 6443 | TCP | Kubernetes API |

## Useful Aliases (Add to ~/.bashrc)

```bash
# kubectl shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgn='kubectl get nodes'
alias kga='kubectl get all -A'

# ArgoCD shortcuts
alias argolist='argocd app list'
alias argosync='argocd app sync'

# Apply this with: source ~/.bashrc
```
