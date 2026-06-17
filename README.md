# GitOps and ArgoCD — Explained

![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?style=flat-square&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.29-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![GCP](https://img.shields.io/badge/GCP-kubeadm-4285F4?style=flat-square&logo=google-cloud&logoColor=white)

> A deep dive into the GitOps model — what ArgoCD actually does, why we made specific design choices on this kubeadm cluster, and what to say about it in an interview.

---

## 🎯 What Problem Does ArgoCD Solve?

**Without GitOps:**
> **💡 Key Idea:** Git becomes the single source of truth. The cluster is expected to always match whatever is in Git — you never run `kubectl apply` for application changes again. You just push.

---

## 🌐 Why NodePort Instead of LoadBalancer

Most ArgoCD tutorials (including the course this repo is based on) use **EKS**, where setting a Service to `type: LoadBalancer` automatically provisions a real AWS load balancer with a public DNS name.

This cluster is built with **kubeadm on raw GCP VMs** — there is no cloud controller watching for LoadBalancer requests. Setting `type: LoadBalancer` here means the `EXTERNAL-IP` stays `<pending>` forever, because nothing in the cluster knows how to fulfil it.

**NodePort is the correct equivalent** for a self-managed cluster — it opens a fixed port (30000–32767 range) on every node, and you access the app via `<any-node-ip>:<nodeport>`.

| | LoadBalancer (EKS) | NodePort (kubeadm) |
|:---|:---:|:---:|
| **Who provisions it** | AWS Cloud Controller | Built into kube-proxy |
| **Access via** | Public DNS name | `<node-ip>:<port>` |
| **Cost** | AWS charges per LB | Free |
| **Works on bare metal** | ❌ No | ✅ Yes |
| **Setup complexity** | Zero — automatic | Zero — automatic |

---

## 🧱 Three Core ArgoCD Concepts

### 1️⃣ Application

An ArgoCD custom resource that tells ArgoCD three things:

| Field | Answers |
|:---|:---|
| `source` | **Where** — which Git repo, branch, and folder path to watch |
| `destination` | **What** — which cluster and namespace to deploy into |
| `syncPolicy` | **How** — automation behaviour |

You `kubectl apply` this file **once**. After that, ArgoCD manages everything inside the path it points to — forever.

### 2️⃣ Sync

The act of making the live cluster match what's in Git.

- **Manual sync** — you click "Sync" in the UI or run a CLI command
- **Automated sync** — ArgoCD does this automatically, within seconds of detecting a Git change

> This repo uses **automated sync** (`syncPolicy.automated`).

### 3️⃣ Self-Heal

If the live cluster drifts from Git — someone runs `kubectl delete deployment nginx`, or manually edits a replica count — ArgoCD detects the difference and **reverts the cluster back to match Git.**

> This is the "magic" most people notice first: delete something manually, watch ArgoCD bring it back within seconds.

---

## ⚙️ syncPolicy — Full Reference

```yaml
syncPolicy:
  automated:
    prune: true        # delete cluster resources if removed from Git
    selfHeal: true      # revert manual cluster changes back to Git state
    allowEmpty: false   # safety guard - blocks sync that would wipe everything
  syncOptions:
  - CreateNamespace=true            # auto-create destination namespace if missing
  - PrunePropagationPolicy=foreground
  - PruneLast=true                  # delete removed resources LAST during a sync
```

| Field | true | false (default) |
|:---|:---|:---|
| **prune** | Deletes resources removed from Git | Leaves orphaned resources in cluster |
| **selfHeal** | Auto-reverts manual cluster drift | Shows "OutOfSync", waits for manual click |
| **allowEmpty** | Allows sync that wipes everything | Blocks sync that would result in zero resources |

> **🎤 Interview line:** *"In production I'd lean toward manual sync with selfHeal off for prod — so a human reviews changes before they hit production — while using automated sync with selfHeal on for dev/staging where speed matters more than caution."*

---

## 🔗 How the Pieces Connect in This Repo
argo-apps/nginx-application.yaml
│
│  repoURL: this same repo
│  path: manifests/nginx
▼
manifests/nginx/deployment.yaml + service.yaml
│
│  ArgoCD reads these and applies them
▼
Cluster: namespace "webapp" → nginx Deployment + Service

argo-apps/grafana-application.yaml
│
│  path: manifests/grafana
▼
manifests/grafana/deployment.yaml + service.yaml
│
▼
Cluster: namespace "monitoring" → Grafana Deployment + Service

> You only ever run `kubectl apply -f argo-apps/` **once**. Everything inside `manifests/` after that is managed entirely by ArgoCD — pushing a change to those files and running `git push` **is** the new "deploy" command.

---

## 📄 Application YAML — Field by Field

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-nginx
  namespace: argocd              # Application objects always live here

spec:
  project: default               # ArgoCD's grouping concept

  source:
    repoURL: https://github.com/sourav-ndx/argocd_gitops.git
    targetRevision: main         # branch to track
    path: manifests/nginx        # folder inside repo to deploy

  destination:
    server: https://kubernetes.default.svc   # fixed value = "this same cluster"
    namespace: webapp            # where the actual pods land

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

---

## 📊 Why Grafana Runs as a Single Replica

Grafana stores dashboards, users, and data source config in **local SQLite** by default.

Deployment (replicas: 2)
│
├── Pod A → own isolated filesystem → grafana.db lives HERE only
└── Pod B → own isolated filesystem → grafana.db lives HERE only

**Kubernetes Deployments do not share disk across replicas by default.** A dashboard created while talking to Pod A simply doesn't exist when traffic happens to route to Pod B.

Even with a shared `ReadWriteMany` volume attached to both pods, SQLite isn't designed for concurrent writes from multiple processes — file locking conflicts would occur.

> **✅ The correct production fix:** set `GF_DATABASE_TYPE=postgres` to move Grafana's state to an external database before scaling horizontally.

This mirrors why MariaDB in production runs as a **StatefulSet** with one PVC *per pod*, not a shared volume across replicas — replication happens at the database protocol level, not the filesystem level.

---

## 🧪 Testing Self-Heal — What To Expect

```bash
# Delete the nginx deployment manually
kubectl delete deployment nginx -n webapp

# Within seconds, check again
kubectl get deployment nginx -n webapp
# It's back — ArgoCD recreated it because Git still says it should exist
```

> **This is the single most important thing to demonstrate** when explaining GitOps to an interviewer — it proves the cluster is no longer the source of truth. Git is.

---

## ⚖️ Comparison to the EKS-Based Tutorial

| Step | EKS Tutorial | This Repo (kubeadm/GCP) |
|:---|:---:|:---:|
| Cluster | EKS (managed) | kubeadm (self-managed) |
| Expose ArgoCD UI | LoadBalancer + public DNS | NodePort + node IP |
| Expose apps | LoadBalancer | NodePort |
| Install ArgoCD | Same manifest | Same manifest |
| Application YAML | Same structure | Same structure |
| Self-heal behaviour | Identical | Identical |

> **The GitOps concepts and ArgoCD mechanics are identical regardless of cluster provider.** Only the Service exposure method changes — proof that understanding fundamentals transfers across any infrastructure.

Note : This Repo is Just to Understand the Concepts of GITOPS & ARCOCD , this does not deploy an End to End two or three tier working application .
Its just deploy some deployments and demostrates the use cases of GitOps and ARGOCD in detail .

---

<div align="center">



*Built on [k8s-kubeadm-gcp](https://github.com/sourav-ndx/k8s-kubeadm-gcp) — the cluster behind this GitOps setup.*

</div>

