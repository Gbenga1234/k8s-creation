# Production-Grade K3s Single-Node Cluster Setup Guide
## Complete Step-by-Step Commands That Worked

---

## Prerequisites

### Hardware Requirements
- **CPU**: 4 cores (minimum 2)
- **RAM**: 8 GB (minimum 2 GB)
- **Disk**: 100 GB SSD
- **Network**: Static IP address

### Software
- Ubuntu 22.04 LTS (or similar)
- Proxmox VE 7.x/8.x (for VM hosting)

---

## Part 1: VM Preparation

### 1.1 Create VM in Proxmox VE
```
1. Click "Create VM"
2. General: VM ID (e.g., 100), Name: k8s-master
3. OS: Select Ubuntu 22.04 ISO
4. System: Default
5. Disks: 100 GB, SSD emulation enabled
6. CPU: 4 cores (type: host)
7. Memory: 8192 MB
8. Network: Bridge (vmbr0)
9. Start VM and install Ubuntu
```

### 1.2 Initial System Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y curl wget nano
```

### 1.3 Configure Static IP

```bash
# Edit netplan configuration
sudo nano /etc/netplan/00-installer-config.yaml
```

**Add this configuration (adjust for your network):**
```yaml
network:
  version: 2
  ethernets:
    ens18:  # Your interface name
      addresses:
        - 192.168.100.X/24  # Your static IP
      gateway4: 192.168.100.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

**Apply the configuration:**
```bash
sudo netplan apply
```

### 1.4 Set Hostname

```bash
# Set hostname
sudo hostnamectl set-hostname k8s-master

# Update hosts file
sudo nano /etc/hosts
```

**Add:**
```
192.168.100.X k8s-master
```

### 1.5 Disable Swap

```bash
# Disable swap permanently
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is off
free -h  # Should show 0 for swap
```

---

## Part 2: Clean Up Old Kubernetes Installation (If Exists)

**Only run if you had kubeadm installed previously:**

```bash
# Reset kubeadm
sudo kubeadm reset -f

# Clean up all Kubernetes files
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/etcd/
sudo rm -rf /var/lib/kubelet/
sudo rm -rf /var/lib/cni/
sudo rm -rf /etc/cni/net.d/
sudo rm -rf $HOME/.kube/

# Clean up iptables rules
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# Remove packages
sudo apt remove -y kubelet kubeadm kubectl containerd
sudo apt autoremove -y

# Reboot
sudo reboot
```

---

## Part 3: Install K3s

### 3.1 Install K3s Server

```bash
# Install K3s with recommended settings
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --disable traefik \
  --tls-san $(hostname -I | awk '{print $1}')
```

**Flags explained:**
- `--write-kubeconfig-mode 644`: Makes kubectl config readable
- `--disable traefik`: We'll use nginx-ingress instead
- `--tls-san`: Adds your IP to API server certificate

### 3.2 Wait for K3s to Start

```bash
# Check K3s status
sudo systemctl status k3s

# Wait for K3s to be ready
sleep 20
```

### 3.3 Configure kubectl for Root User

```bash
# Switch to root
sudo -i

# Configure kubectl
mkdir -p $HOME/.kube
cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Verify it works
kubectl get nodes
```

### 3.4 Configure kubectl for Regular User

```bash
# Exit root and return to your user
exit

# Configure kubectl for your user
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify it works
kubectl get nodes
kubectl get pods -A
```

### 3.5 Verify Cluster is Running

```bash
# Check node status (should show Ready)
kubectl get nodes

# Check all system pods are running
kubectl get pods -A

# Check cluster info
kubectl cluster-info
```

**Expected output:**
```
NAME         STATUS   ROLES                  AGE   VERSION
k8s-master   Ready    control-plane,master   2m    v1.28.x+k3s1
```

---

## Part 4: Install MetalLB Load Balancer

### 4.1 Install MetalLB

```bash
# Install MetalLB v0.13.12
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

### 4.2 Wait for MetalLB Pods

```bash
# Watch pods until they're running (Ctrl+C to stop)
kubectl get pods -n metallb-system -w
```

**Wait until you see:**
```
NAME                          READY   STATUS    RESTARTS   AGE
controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
speaker-xxxxx                 1/1     Running   0          2m
```

Press **Ctrl+C** to stop watching.

### 4.3 Configure MetalLB IP Address Pool

```bash
# Create IP pool configuration (ADJUST IP RANGE FOR YOUR NETWORK!)
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.100.200-192.168.100.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
```

**⚠️ IMPORTANT:** 
- Replace `192.168.100.200-192.168.100.220` with IPs from YOUR network
- Ensure these IPs are NOT used by DHCP
- IPs must be in the same subnet as your cluster node

### 4.4 Verify MetalLB Configuration

```bash
# Check IP pool was created
kubectl get ipaddresspool -n metallb-system

# Check L2 advertisement
kubectl get l2advertisement -n metallb-system

# Check MetalLB pods
kubectl get pods -n metallb-system
```

### 4.5 Test MetalLB with Nginx

```bash
# Create test deployment
kubectl create deployment nginx --image=nginx

# Expose with LoadBalancer
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Wait a few seconds
sleep 10

# Check service (should have EXTERNAL-IP from MetalLB pool)
kubectl get svc nginx
```

**Expected output:**
```
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
nginx   LoadBalancer   10.43.x.x       192.168.100.200   80:xxxxx/TCP   10s
```

### 4.6 Test Access

```bash
# Test nginx is accessible
curl http://192.168.100.200

# Or visit in browser: http://192.168.100.200
```

You should see the nginx welcome page!

### 4.7 Clean Up Test Deployment

```bash
# Remove test nginx
kubectl delete deployment nginx
kubectl delete service nginx
```

---

## Part 5: Install ArgoCD

### 5.1 Create ArgoCD Namespace

```bash
# Create namespace
kubectl create namespace argocd
```

### 5.2 Install ArgoCD

```bash
# Install ArgoCD stable version
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 5.3 Wait for ArgoCD Pods

```bash
# Watch pods until all are running (takes 2-3 minutes)
kubectl get pods -n argocd -w
```

**Wait until all pods show Running:**
```
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-application-controller-0       1/1     Running   0          2m
argocd-applicationset-controller-...  1/1     Running   0          2m
argocd-dex-server-...                 1/1     Running   0          2m
argocd-notifications-controller-...   1/1     Running   0          2m
argocd-redis-...                      1/1     Running   0          2m
argocd-repo-server-...                1/1     Running   0          2m
argocd-server-...                     1/1     Running   0          2m
```

Press **Ctrl+C** once all are running.

### 5.4 Expose ArgoCD Server with LoadBalancer

```bash
# Change service type to LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Wait for MetalLB to assign IP
sleep 10

# Check service (should have EXTERNAL-IP)
kubectl get svc argocd-server -n argocd
```

**Expected output:**
```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)
argocd-server   LoadBalancer   10.43.x.x       192.168.100.201   80:xxx/TCP,443:xxx/TCP
```

### 5.5 Get ArgoCD Access Credentials

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Get LoadBalancer IP
kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

**Or get everything at once:**
```bash
echo "================================"
echo "ArgoCD Access Information"
echo "================================"
echo "URL: https://$(kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
echo "Username: admin"
echo "Password: $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)"
echo "================================"
```

### 5.6 Access ArgoCD Web UI

1. Open browser to: `https://<EXTERNAL-IP>` (from command above)
2. Accept self-signed certificate warning
3. Login with:
   - **Username:** `admin`
   - **Password:** (from command above)

---

## Part 6: Install ArgoCD CLI (Optional but Recommended)

### 6.1 Download and Install ArgoCD CLI

```bash
# Download ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Install it
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Remove downloaded file
rm argocd-linux-amd64

# Verify installation
argocd version --client
```

### 6.2 Login via CLI

```bash
# Get ArgoCD server IP
ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Get initial password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

# Login (use --insecure for self-signed cert)
argocd login $ARGOCD_SERVER --username admin --password $ARGOCD_PASSWORD --insecure
```

### 6.3 Change Admin Password (Recommended)

```bash
# Change password interactively
argocd account update-password
```

---

## Part 7: Verification and Status Checks

### 7.1 Complete Cluster Status

```bash
# Check node status
kubectl get nodes -o wide

# Check all pods
kubectl get pods -A

# Check all services with LoadBalancer
kubectl get svc -A -o wide | grep LoadBalancer

# Check MetalLB IP pool
kubectl get ipaddresspool -n metallb-system

# Check ArgoCD applications
argocd app list
```

### 7.2 Create Verification Script

```bash
# Create status check script
cat > cluster-status.sh << 'SCRIPT'
#!/bin/bash

echo "======================================"
echo "K3s Cluster Status Check"
echo "======================================"

echo -e "\n=== Cluster Nodes ==="
kubectl get nodes -o wide

echo -e "\n=== All Pods ==="
kubectl get pods -A

echo -e "\n=== LoadBalancer Services ==="
kubectl get svc -A -o wide | grep LoadBalancer

echo -e "\n=== MetalLB Configuration ==="
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system

echo -e "\n=== ArgoCD Access ==="
echo "URL: https://$(kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
echo "Username: admin"
echo "Password: $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" 2>/dev/null | base64 -d || echo 'Password already changed')"

echo -e "\n=== ArgoCD Applications ==="
argocd app list 2>/dev/null || echo "ArgoCD CLI not configured"

echo -e "\n======================================"
SCRIPT

# Make it executable
chmod +x cluster-status.sh

# Run it
./cluster-status.sh
```

---

## Part 8: Deploy First Application via ArgoCD

### 8.1 Deploy Sample Guestbook App via CLI

```bash
# Create ArgoCD application
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Sync the application
argocd app sync guestbook

# Watch the deployment
kubectl get pods -w
```

Press **Ctrl+C** once pods are running.

### 8.2 Deploy via ArgoCD Web UI

1. Open ArgoCD Web UI
2. Click **+ NEW APP**
3. Fill in:
   - **Application Name:** `guestbook`
   - **Project:** `default`
   - **Sync Policy:** `Manual` or `Automatic`
   - **Repository URL:** `https://github.com/argoproj/argocd-example-apps.git`
   - **Path:** `guestbook`
   - **Cluster URL:** `https://kubernetes.default.svc`
   - **Namespace:** `default`
4. Click **CREATE**
5. Click **SYNC** → **SYNCHRONIZE**

### 8.3 Access the Application

```bash
# Expose guestbook UI with LoadBalancer
kubectl patch svc guestbook-ui -p '{"spec": {"type": "LoadBalancer"}}'

# Get the external IP
kubectl get svc guestbook-ui

# Access in browser: http://<EXTERNAL-IP>
```

---

## Part 9: Additional Production Features

### 9.1 Install Nginx Ingress Controller (Optional)

```bash
# Install nginx-ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/cloud/deploy.yaml

# Patch to use LoadBalancer
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'

# Check service
kubectl get svc -n ingress-nginx
```

### 9.2 Setup K3s Automated Backups

```bash
# K3s automatically backs up etcd snapshots
# View existing snapshots
sudo ls -lah /var/lib/rancher/k3s/server/db/snapshots/

# Create manual snapshot
sudo k3s etcd-snapshot save --name manual-backup-$(date +%Y%m%d)

# Schedule automated backups (edit K3s service)
sudo systemctl edit k3s
```

**Add this to the override file:**
```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/k3s server --etcd-snapshot-schedule-cron "0 */6 * * *" --etcd-snapshot-retention 14
```

**Then reload:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

### 9.3 Install Storage Provisioner (local-path)

K3s includes local-path-provisioner by default.

```bash
# Verify it's installed
kubectl get storageclass

# Set as default if not already
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 9.4 Enable Metrics Server

K3s includes metrics-server by default.

```bash
# Check if it's running
kubectl get pods -n kube-system | grep metrics-server

# Test it
kubectl top nodes
kubectl top pods -A
```

---

## Part 10: Useful Commands Reference

### Cluster Management

```bash
# Check K3s status
sudo systemctl status k3s

# Restart K3s
sudo systemctl restart k3s

# View K3s logs
sudo journalctl -u k3s -f

# Get cluster info
kubectl cluster-info

# Get all resources
kubectl get all -A
```

### ArgoCD Management

```bash
# List applications
argocd app list

# Get app details
argocd app get <app-name>

# Sync application
argocd app sync <app-name>

# Delete application
argocd app delete <app-name>

# Access ArgoCD server
argocd login <server-ip> --insecure
```

### Debugging

```bash
# Get pod logs
kubectl logs <pod-name> -n <namespace>

# Describe resource
kubectl describe pod <pod-name> -n <namespace>

# Get events
kubectl get events -A --sort-by='.lastTimestamp'

# Check node resources
kubectl top nodes
kubectl describe node k8s-master

# Test service connectivity
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://<service-name>
```

### Backup and Restore

```bash
# Create etcd snapshot
sudo k3s etcd-snapshot save --name backup-$(date +%Y%m%d-%H%M%S)

# List snapshots
sudo k3s etcd-snapshot ls

# Restore from snapshot (CAUTION: This will reset your cluster!)
sudo k3s server --cluster-reset --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<snapshot-name>
```

---

## Part 11: Troubleshooting

### Issue: kubectl not working

```bash
# Reconfigure kubectl
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Issue: Pods stuck in Pending

```bash
# Check why
kubectl describe pod <pod-name> -n <namespace>

# Check node resources
kubectl top nodes
kubectl describe node k8s-master
```

### Issue: LoadBalancer service stuck in Pending

```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check IP pool configuration
kubectl get ipaddresspool -n metallb-system
kubectl describe ipaddresspool default-pool -n metallb-system

# Check MetalLB logs
kubectl logs -n metallb-system -l app=metallb
```

### Issue: ArgoCD not accessible

```bash
# Check ArgoCD pods
kubectl get pods -n argocd

# Check ArgoCD service
kubectl get svc argocd-server -n argocd

# Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
```

### Issue: Need to completely reset cluster

```bash
# Stop K3s
sudo systemctl stop k3s

# Uninstall K3s
sudo /usr/local/bin/k3s-uninstall.sh

# Clean up
sudo rm -rf /var/lib/rancher/k3s
sudo rm -rf /etc/rancher/k3s
sudo rm -rf $HOME/.kube

# Reinstall (go back to Part 3)
```

---

## Summary

You now have:
- ✅ K3s single-node cluster running
- ✅ MetalLB load balancer configured (192.168.100.200-220)
- ✅ ArgoCD installed and accessible via LoadBalancer
- ✅ GitOps ready for production deployments
- ✅ Automated etcd backups
- ✅ Monitoring with metrics-server

**Your cluster is production-ready!**

---

## Quick Reference Card

```bash
# Cluster Status
./cluster-status.sh

# Deploy App via ArgoCD
argocd app create <name> --repo <git-url> --path <path> --dest-server https://kubernetes.default.svc --dest-namespace default
argocd app sync <name>

# Create LoadBalancer Service
kubectl expose deployment <name> --port=<port> --type=LoadBalancer

# View Logs
kubectl logs -f <pod-name> -n <namespace>

# Backup Cluster
sudo k3s etcd-snapshot save --name backup-$(date +%Y%m%d)

# Get ArgoCD Password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## IP Address Reference (Adjust for Your Network)

| Component | IP Address | Port |
|-----------|------------|------|
| K3s Node | 192.168.100.X | 6443 |
| MetalLB Pool | 192.168.100.200-220 | - |
| ArgoCD Server | 192.168.100.201 | 80, 443 |
| Test Nginx | 192.168.100.200 | 80 |
| Ingress Controller | 192.168.100.202 | 80, 443 |

**Remember:** Always adjust IP addresses to match your network configuration!

---

## End of Setup Guide

Save this document for future cluster recreations. All commands have been tested and verified to work.