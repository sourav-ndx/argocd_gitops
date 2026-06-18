# GCP Setup Guide — Prerequisites for Kubernetes on GCP VMs

> Everything you need to configure at the GCP infrastructure level BEFORE installing Kubernetes. These are GCP-specific requirements not covered in generic kubeadm documentation — skipping any of these will cause silent, hard-to-diagnose failures later.

---

## Required Firewall Rules

GCP blocks all non-standard traffic by default. Run these commands in Google Cloud Shell before installing anything.

### 1. Allow Calico IP-in-IP tunnel traffic between nodes
```bash
gcloud compute firewall-rules create allow-calico-ipip \
    --network=default \
    --allow=ipip \
    --direction=INGRESS \
    --source-ranges=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16 \
    --description="Allow Calico IP-in-IP traffic between K8s nodes"
```
**Why:** Calico uses IP-in-IP encapsulation to route pod traffic between nodes. GCP treats this as a separate protocol (not TCP/UDP) and blocks it by default. Without this rule, cross-node pod networking is completely broken — pods on different nodes cannot communicate at all, with no obvious error message.

### 2. Allow internal cluster traffic
```bash
gcloud compute firewall-rules create allow-k8s-internal \
    --network=default \
    --allow=tcp,udp,icmp \
    --direction=INGRESS \
    --source-ranges=10.128.0.0/9,192.168.0.0/16 \
    --description="Allow internal K8s cluster traffic"
```

### 3. Allow ArgoCD UI access
```bash
gcloud compute firewall-rules create allow-argocd-nodeport \
    --network=default \
    --allow=tcp:30843 \
    --direction=INGRESS \
    --source-ranges=0.0.0.0/0 \
    --description="Allow ArgoCD UI NodePort access"
```

### 4. Allow application NodePorts
```bash
gcloud compute firewall-rules create allow-webapp-nodeports \
    --network=default \
    --allow=tcp:30080,tcp:30030,tcp:30088 \
    --direction=INGRESS \
    --source-ranges=0.0.0.0/0 \
    --description="Allow nginx, grafana, guestbook NodePort access"
```

---

## Verify All Firewall Rules

```bash
gcloud compute firewall-rules list --format="table(name,allowed,sourceRanges)"
```

---

## Important GCP Gotchas

### Ephemeral vs Static IPs
GCP assigns ephemeral external IPs to VMs by default. These IPs CHANGE every time you stop and restart a VM.

```bash
# Always check current external IP after a restart
gcloud compute instances describe k8s-control \
    --zone=us-central1-a \
    --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
```

In production, reserve a static IP to avoid this:
```bash
gcloud compute addresses create k8s-control-ip --region=us-central1
```

### IP Forwarding
Check if IP forwarding is enabled on your VMs:
```bash
gcloud compute instances describe k8s-control \
    --zone=us-central1-a \
    --format="get(canIpForward)"
```

If it returns `False` — enable it via GCP Console:
1. Stop the VM
2. Edit the VM
3. Enable "IP forwarding" checkbox
4. Start the VM

---

## Summary — What Each Rule Does

| Rule | Protocol/Port | Why Needed |
|:---|:---|:---|
| allow-calico-ipip | ipip protocol | Cross-node pod networking |
| allow-k8s-internal | tcp,udp,icmp | Internal cluster communication |
| allow-argocd-nodeport | tcp:30843 | ArgoCD UI from browser |
| allow-webapp-nodeports | tcp:30080,30030,30088 | App access from browser |
