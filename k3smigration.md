# K3s Migration Plan

Migration plan for moving the homelab DNS server stack from Docker Compose to k3s with Cilium CNI.

## Overview

**Goal**: Migrate Pi-hole, Traefik, and test services to k3s cluster while maintaining wildcard DNS functionality and adding easy service deployment capabilities.

**Key Components**:
- k3s (lightweight Kubernetes)
- Cilium (CNI for networking)
- Pi-hole (DNS + ad blocking)
- Traefik (Ingress controller)
- MetalLB (LoadBalancer for bare metal)

---

## Phase 1: Repository Changes

### 1.1 Create New Ansible Role: `k3s-cluster`

**Location**: `roles/k3s-cluster/`

**Tasks**:
- Install k3s with `--flannel-backend=none` and `--disable-network-policy` (for Cilium)
- Install Cilium CNI via Helm
- Configure kubeconfig access
- Set up kubectl and helm on the server

**Files to create**:
```
roles/k3s-cluster/
├── defaults/main.yml
├── tasks/main.yml
├── handlers/main.yml
└── templates/
    └── k3s-config.yaml.j2
```

### 1.2 Create New Ansible Role: `k3s-ingress`

**Location**: `roles/k3s-ingress/`

**Tasks**:
- Deploy MetalLB for LoadBalancer IPs
- Deploy Traefik as Kubernetes Ingress Controller
- Configure Traefik dashboard access
- Set up IngressRoute CRDs

### 1.3 Create New Ansible Role: `k3s-dns`

**Location**: `roles/k3s-dns/`

**Tasks**:
- Deploy Pi-hole as Kubernetes Deployment
- Create PersistentVolume for Pi-hole config
- Expose Pi-hole on host network (port 53)
- Configure wildcard DNS entries
- Set up Pi-hole admin interface via Ingress

### 1.4 Create Kubernetes Manifests Directory

**Location**: `k8s-manifests/`

**Structure**:
```
k8s-manifests/
├── namespaces/
│   └── homelab.yaml
├── pihole/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── pvc.yaml
│   └── ingress.yaml
├── traefik/
│   ├── helmvalues.yaml
│   └── dashboard-ingress.yaml
├── metallb/
│   ├── config.yaml
│   └── ippool.yaml
└── examples/
    ├── nodejs-app.yaml
    └── media-player.yaml
```

### 1.5 Update Main Playbook

**File**: `site.yml`

Add new roles:
```yaml
- hosts: homelab
  roles:
    - system-setup
    - docker-engine  # Keep for local dev if needed
    - k3s-cluster
    - k3s-ingress
    - k3s-dns
    - monitoring
```

### 1.6 Update Host Variables

**File**: `host_vars/archserver.yml`

Add k3s-specific variables:
```yaml
# Existing vars...
root_domain: "homelab"
server_ip: "192.168.10.217"

# New k3s vars
k3s_version: "v1.28.5+k3s1"
cilium_version: "1.14.5"
metallb_ip_range: "192.168.10.220-192.168.10.230"
k3s_cluster_cidr: "10.42.0.0/16"
k3s_service_cidr: "10.43.0.0/16"
```

### 1.7 Create Service Template

**File**: `templates/service-template.yaml`

Template for easy service deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ service_name }}
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ service_name }}
  template:
    metadata:
      labels:
        app: {{ service_name }}
    spec:
      containers:
      - name: {{ service_name }}
        image: {{ image }}
        ports:
        - containerPort: {{ port }}
---
apiVersion: v1
kind: Service
metadata:
  name: {{ service_name }}
  namespace: homelab
spec:
  selector:
    app: {{ service_name }}
  ports:
  - port: 80
    targetPort: {{ port }}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ service_name }}
  namespace: homelab
spec:
  rules:
  - host: {{ service_name }}.{{ root_domain }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ service_name }}
            port:
              number: 80
```

---

## Phase 2: Arch Linux Server Preparation

### 2.1 Pre-Migration Checks

**System Requirements**:
```bash
# Check kernel version (need 4.19+)
uname -r

# Check available memory (recommend 2GB+)
free -h

# Check disk space (need 10GB+ free)
df -h /opt

# Verify cgroups v2
mount | grep cgroup

# Check if ports are available
ss -tulpn | grep -E ':(6443|10250|53)'
```

### 2.2 Install Dependencies

```bash
# Update system
sudo pacman -Syu

# Install required packages
sudo pacman -S curl wget git

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 2.3 Kernel Module Configuration

**Check/Enable modules**:
```bash
# Required for Cilium
sudo modprobe br_netfilter
sudo modprobe overlay
sudo modprobe xt_socket
sudo modprobe xt_mark

# Make persistent
cat <<EOF | sudo tee /etc/modules-load.d/k3s.conf
br_netfilter
overlay
xt_socket
xt_mark
EOF
```

### 2.4 Sysctl Configuration

```bash
# Configure networking
cat <<EOF | sudo tee /etc/sysctl.d/k3s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

sudo sysctl --system
```

### 2.5 Firewall Configuration

```bash
# If using firewalld or iptables, allow k3s ports
sudo firewall-cmd --permanent --add-port=6443/tcp  # k3s API
sudo firewall-cmd --permanent --add-port=10250/tcp # kubelet
sudo firewall-cmd --permanent --add-port=53/udp    # DNS
sudo firewall-cmd --permanent --add-port=53/tcp    # DNS
sudo firewall-cmd --permanent --add-port=80/tcp    # HTTP
sudo firewall-cmd --permanent --add-port=443/tcp   # HTTPS
sudo firewall-cmd --reload
```

### 2.6 Backup Current Configuration

```bash
# Backup Docker Compose configs
sudo cp -r /opt/homelab /opt/homelab.backup.$(date +%Y%m%d)

# Export Pi-hole configuration
docker exec pihole pihole -a -t /etc/pihole/backup.tar.gz

# Copy backup to safe location
sudo cp /opt/homelab/pihole/config/backup.tar.gz ~/pihole-backup.tar.gz
```

### 2.7 Stop Current Services

```bash
# Stop Docker Compose stack
cd /opt/homelab
sudo docker-compose down

# Verify all containers stopped
sudo docker ps
```

---

## Phase 3: Migration Execution

### 3.1 Run Ansible Playbook

```bash
# From repository root
ansible-playbook -i inventory/hosts site.yml --tags k3s

# Or run full playbook
ansible-playbook -i inventory/hosts site.yml
```

### 3.2 Verify k3s Installation

```bash
# On the server
sudo k3s kubectl get nodes
sudo k3s kubectl get pods -A

# Check Cilium status
cilium status

# Verify MetalLB
kubectl get ipaddresspool -n metallb-system
```

### 3.3 Deploy Core Services

```bash
# Apply manifests
kubectl apply -f /opt/homelab/k8s-manifests/namespaces/
kubectl apply -f /opt/homelab/k8s-manifests/pihole/
kubectl apply -f /opt/homelab/k8s-manifests/traefik/

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=pihole -n homelab --timeout=300s
```

### 3.4 Restore Pi-hole Configuration

```bash
# Copy backup into Pi-hole pod
kubectl cp ~/pihole-backup.tar.gz homelab/pihole-pod:/tmp/backup.tar.gz

# Restore inside pod
kubectl exec -n homelab pihole-pod -- pihole -a -r /tmp/backup.tar.gz
```

### 3.5 Configure DNS Entries

```bash
# Add wildcard DNS via Pi-hole admin or kubectl
kubectl exec -n homelab pihole-pod -- bash -c "echo 'address=/homelab/192.168.10.217' >> /etc/dnsmasq.d/02-custom.conf"
kubectl exec -n homelab pihole-pod -- pihole restartdns
```

### 3.6 Test Services

```bash
# Test DNS resolution
dig @192.168.10.217 pihole.homelab
dig @192.168.10.217 traefik.homelab

# Test HTTP access
curl http://pihole.homelab
curl http://traefik.homelab

# Check ingress
kubectl get ingress -n homelab
```

---

## Phase 4: Adding New Services

### 4.1 Example: Node.js Media Player

**File**: `media-player-deployment.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: media
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: media-player
  namespace: media
spec:
  replicas: 1
  selector:
    matchLabels:
      app: media-player
  template:
    metadata:
      labels:
        app: media-player
    spec:
      containers:
      - name: media-player
        image: node:20-alpine
        workingDir: /app
        command: ["node", "server.js"]
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: app-code
          mountPath: /app
        - name: media-storage
          mountPath: /media
      volumes:
      - name: app-code
        hostPath:
          path: /opt/homelab/apps/media-player
      - name: media-storage
        hostPath:
          path: /mnt/media
---
apiVersion: v1
kind: Service
metadata:
  name: media-player
  namespace: media
spec:
  selector:
    app: media-player
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: media-player
  namespace: media
spec:
  rules:
  - host: media.homelab
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: media-player
            port:
              number: 80
```

**Deploy**:
```bash
kubectl apply -f media-player-deployment.yaml
```

### 4.2 Quick Service Deployment Script

**File**: `scripts/deploy-service.sh`

```bash
#!/bin/bash
# Usage: ./deploy-service.sh <name> <image> <port>

NAME=$1
IMAGE=$2
PORT=$3
DOMAIN=${4:-homelab}

kubectl create deployment $NAME --image=$IMAGE -n homelab
kubectl expose deployment $NAME --port=80 --target-port=$PORT -n homelab
kubectl create ingress $NAME --rule="${NAME}.${DOMAIN}/*=${NAME}:80" -n homelab

echo "Service deployed: http://${NAME}.${DOMAIN}"
```

---

## Phase 5: Post-Migration

### 5.1 Monitoring Setup

```bash
# Check cluster health
kubectl get nodes
kubectl get pods -A
kubectl top nodes
kubectl top pods -A

# View logs
kubectl logs -n homelab -l app=pihole
kubectl logs -n homelab -l app=traefik
```

### 5.2 Update Documentation

- Update README.md with k3s instructions
- Document kubectl commands for common tasks
- Add troubleshooting section for k3s

### 5.3 Cleanup Old Docker Resources

```bash
# After confirming k3s works
sudo docker system prune -a
sudo rm -rf /opt/homelab.backup.*  # After 30 days
```

---

## Rollback Plan

If migration fails:

```bash
# Stop k3s
sudo systemctl stop k3s

# Restore Docker Compose
cd /opt/homelab.backup.*
sudo docker-compose up -d

# Restore Pi-hole config
docker exec pihole pihole -a -r /backup/backup.tar.gz
```

---

## Benefits of k3s Migration

1. **Declarative Configuration**: All services defined in YAML
2. **Self-Healing**: Automatic pod restarts on failure
3. **Easy Scaling**: Scale services with `kubectl scale`
4. **Resource Management**: CPU/memory limits and requests
5. **Rolling Updates**: Zero-downtime deployments
6. **Service Discovery**: Built-in DNS for pod-to-pod communication
7. **Secrets Management**: Kubernetes secrets for sensitive data
8. **Future-Proof**: Industry-standard orchestration platform

---

## Next Steps

1. Review and adjust IP ranges in host_vars
2. Test Ansible roles in development environment
3. Schedule maintenance window for migration
4. Execute Phase 2 (server preparation)
5. Run Ansible playbook (Phase 3)
6. Validate all services working
7. Deploy first custom service (media player)
