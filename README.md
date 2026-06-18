<div align="center">

# 🚀 ArgoCD GitOps on kubeadm Cluster
### Deploying nginx, Grafana and Guestbook via GitOps — on a self-built Kubernetes cluster on GCP

![ArgoCD](https://img.shields.io/badge/ArgoCD-v3.4-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.29-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![GCP](https://img.shields.io/badge/GCP-Compute_Engine-4285F4?style=for-the-badge&logo=google-cloud&logoColor=white)
![Calico](https://img.shields.io/badge/CNI-Calico_v3.26-F8861A?style=for-the-badge)
![Status](https://img.shields.io/badge/Apps-Synced_&_Healthy-brightgreen?style=for-the-badge)

<br/>

> This repo is about understanding GitOps properly — not just following a tutorial on a managed cluster.
> Built on raw GCP VMs using kubeadm, which means everything that could go wrong, did.
> The learnings from that are honestly more valuable than the setup itself.

</div>

---

> **📌 Note:** This repo demonstrates GitOps and ArgoCD concepts — deploying real workloads, testing self-healing, managing apps through Git. It is not a production-grade application. The point is understanding the pattern, not the apps themselves.

Built on top of [k8s-kubeadm-gcp](https://github.com/sourav-ndx/k8s-kubeadm-gcp) — the cluster behind all of this.

---

## 💡 What I Learned

The concept clicked fast — Git as the source of truth, cluster always matching what is in the repo, delete something manually and it comes back automatically. That self-healing moment when I deleted a deployment and watched ArgoCD recreate it within seconds — that is when it actually made sense, not when I read about it.

What took longer was everything around it. Full story in [docs/troubleshooting-learnings.md](docs/troubleshooting-learnings.md).

---

## 📸 Screenshots

### ArgoCD Dashboard — All Apps Synced and Healthy
![ArgoCD Dashboard](docs/images/argocd-dashboard.png)

### Guestbook App Resource Graph in ArgoCD UI
![Guestbook Graph](docs/images/argocd-guestbook-graph.png)

### Apps Running in Browser
![Guestbook](docs/images/guestbook-app.png)
![Grafana](docs/images/grafana-login.png)
![Nginx](docs/images/nginx-welcome.png)

---

## 📁 Repository Structure

```
argocd_gitops/
├── argo-apps/
│   ├── nginx-application.yaml          # ArgoCD Application — watches manifests/nginx
│   ├── grafana-application.yaml        # ArgoCD Application — watches manifests/grafana
│   ├── guestbook-application.yaml      # ArgoCD Application — watches manifests/guestbook
│   └── argocd-server-nodeport.yaml     # Custom NodePort Service for ArgoCD UI
├── manifests/
│   ├── nginx/
│   │   ├── deployment.yaml
│   │   └── service.yaml                # NodePort 30080
│   ├── grafana/
│   │   ├── deployment.yaml
│   │   └── service.yaml                # NodePort 30030
│   └── guestbook/
│       ├── deployment.yaml
│       └── service.yaml                # NodePort 30088
├── docs/
│   ├── gitops-explained.md             # Concepts deep dive — read this first
│   ├── troubleshooting-learnings.md    # What broke and what fixed it
│   └── gcp-setup-guide.md             # GCP-specific prerequisites and firewall rules
└── README.md
```

---

## 🌐 Why NodePort Instead of LoadBalancer

Tutorials based on EKS use `type: LoadBalancer` — AWS auto-provisions a real load balancer with a public DNS name. This cluster runs on raw kubeadm GCP VMs — there is no cloud controller to fulfil that request, so the EXTERNAL-IP stays `<pending>` forever.

**NodePort is the correct equivalent** for a self-managed cluster — access via `<node-ip>:<nodeport>` instead of a DNS name.

| | LoadBalancer (EKS) | NodePort (kubeadm) |
|:---|:---:|:---:|
| **Who provisions it** | AWS Cloud Controller | Built into kube-proxy |
| **Access via** | Public DNS name | `<node-ip>:<port>` |
| **Cost** | AWS charges per LB | Free |
| **Works on bare metal** | ❌ No | ✅ Yes |

Full comparison in [docs/gitops-explained.md](docs/gitops-explained.md).

---

## ⚠️ GCP Prerequisites — Do This First

> Skipping these will cause silent, hard-to-diagnose failures — especially the IP-in-IP rule which took hours to find.

### Allow Calico IP-in-IP tunnel traffic (Critical)
```bash
gcloud compute firewall-rules create allow-calico-ipip \
    --network=default \
    --allow=ipip \
    --direction=INGRESS \
    --source-ranges=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16 \
    --description="Allow Calico IP-in-IP traffic between K8s nodes"
```

### Allow internal cluster traffic
```bash
gcloud compute firewall-rules create allow-k8s-internal \
    --network=default \
    --allow=tcp,udp,icmp \
    --direction=INGRESS \
    --source-ranges=10.128.0.0/9,192.168.0.0/16
```

### Allow ArgoCD UI and app NodePorts
```bash
gcloud compute firewall-rules create allow-argocd-nodeport \
    --network=default \
    --allow=tcp:30843 \
    --direction=INGRESS \
    --source-ranges=0.0.0.0/0

gcloud compute firewall-rules create allow-webapp-nodeports \
    --network=default \
    --allow=tcp:30080,tcp:30030,tcp:30088 \
    --direction=INGRESS \
    --source-ranges=0.0.0.0/0
```

Full GCP setup guide: [docs/gcp-setup-guide.md](docs/gcp-setup-guide.md)

---

## 🔧 Setup — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

### Expose ArgoCD UI via NodePort

```bash
# Delete the default ClusterIP Service that comes with install.yaml
kubectl delete svc argocd-server -n argocd

# Apply our own NodePort Service
kubectl apply -f argo-apps/argocd-server-nodeport.yaml
kubectl get svc argocd-server -n argocd
```

### Get admin password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Username: `admin`

### Get current external IP

> GCP ephemeral IPs change every time you stop a VM — always check after restart

```bash
gcloud compute instances describe k8s-control \
    --zone=us-central1-a \
    --format="get(networkInterfaces[0].accessConfigs[0].natIP)"
```

Access UI: `https://<external-ip>:30843`

---

## 🚀 Deploy the Applications

```bash
git clone https://github.com/sourav-ndx/argocd_gitops.git
cd argocd_gitops

kubectl apply -f argo-apps/nginx-application.yaml
kubectl apply -f argo-apps/grafana-application.yaml
kubectl apply -f argo-apps/guestbook-application.yaml

kubectl get applications -n argocd -w
```

> ArgoCD reads those Application objects, pulls the manifests from this GitHub repo, and deploys everything automatically. You never touch the manifests folder directly — that is the whole point.

---

## 🌍 Access the Apps

| App | URL | Login |
|:---|:---|:---|
| **ArgoCD UI** | `https://node-ip:30843` | admin / (from secret above) |
| **nginx** | `http://node-ip:30080` | — |
| **Grafana** | `http://node-ip:30030` | admin / admin123 |
| **Guestbook** | `http://node-ip:30088` | — |

---

## 🧪 Test Self-Healing

```bash
# Delete nginx deployment manually
kubectl delete deployment nginx -n webapp

# Within seconds — check again
kubectl get deployment nginx -n webapp
# It is back. ArgoCD detected the drift and recreated it from Git.
```

> This is the single most important thing to demonstrate when explaining GitOps — it proves the cluster is no longer the source of truth. Git is.

---

## 🔄 Force a Manual Sync

```bash
kubectl patch application webapp-nginx -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

---

## 🧹 Cleanup

```bash
kubectl delete -f argo-apps/
kubectl delete namespace webapp monitoring guestbook argocd
```

---

## 📖 Further Reading

- [GitOps and ArgoCD Explained](docs/gitops-explained.md) — concepts, syncPolicy deep dive, field-by-field Application YAML breakdown
- [Troubleshooting and Learnings](docs/troubleshooting-learnings.md) — what broke, what fixed it, full debugging approach
- [GCP Setup Guide](docs/gcp-setup-guide.md) — all GCP-specific prerequisites and firewall rules with exact CLI commands

---

## 👤 Author

**Sourav Nandy** — Platform and DevOps Engineer | CKA Certified
Ericsson / Verizon | OpenShift Production SME | 8+ Years

[![LinkedIn](https://img.shields.io/badge/LinkedIn-sourav--nandy--0115-0A66C2?style=flat&logo=linkedin)](https://linkedin.com/in/sourav-nandy-0115)
[![GitHub](https://img.shields.io/badge/GitHub-sourav--ndx-181717?style=flat&logo=github)](https://github.com/sourav-ndx)

---

<div align="center">

*Built on [k8s-kubeadm-gcp](https://github.com/sourav-ndx/k8s-kubeadm-gcp) — the cluster behind this GitOps setup.*

⭐ Star this repo if it helped you understand GitOps and ArgoCD

</div>
