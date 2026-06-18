# Troubleshooting & Learnings — What Actually Happened

This was my first time running ArgoCD on a self-built cluster. Not on EKS, not on GKE — raw kubeadm on two GCP VMs. Everything that could go wrong, did. Here is what broke and what actually fixed it.

## 1. CoreDNS looked fine but was not working

ArgoCD server pod kept failing to connect to Redis. The error said DNS lookup timeout. I checked CoreDNS pods — all showing Running. Spent time assuming it was something else before realising Running does not mean working. A simple rollout restart fixed it immediately.

    kubectl rollout restart deployment coredns -n kube-system

Lesson: always verify the function, not just the status. A pod can be Running and completely broken at the same time.

## 2. Re-applying a manifest wiped my custom Service

I had created my own NodePort Service for ArgoCD. Later re-applied the full install.yaml to fix a missing CRD. It silently overwrote my custom Service back to ClusterIP — no warning, no error. Took me a while to even realise what happened.

Never re-apply a full manifest when you have manual customisations on top of it. Apply targeted fixes only.

## 3. The ApplicationSet CRD was too large for kubectl apply

One of ArgoCD CRDs kept failing with a 262KB annotation size limit error. kubectl apply stores the entire object definition in an annotation for future diffing — this CRD was just too large for that. Fixed by using kubectl create instead of kubectl apply for that specific CRD.

## 4. Cross-node pod networking was completely dead and silent

This one took the longest. Pods on the worker node were completely unreachable from the control plane. No connection refused, no error — just silence. Checked Calico routes, BGP state, block affinities, tunnel interfaces — everything looked correct on paper.

Turns out GCP silently drops IP-in-IP protocol traffic between VMs by default. Calico uses this protocol to route pod traffic between nodes. One firewall rule fixed what hours of debugging could not find.

    gcloud compute firewall-rules create allow-calico-ipip \
        --network=default \
        --allow=ipip \
        --direction=INGRESS \
        --source-ranges=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16

The frustrating part — there was no log anywhere telling me this was happening. Packets were disappearing silently at the GCP infrastructure level.

## 5. VM stopped, IP changed, nothing told me

After stopping k8s-control, it came back with a completely different external IP. GCP ephemeral IPs do not survive a stop. Always check the IP after a restart.

    gcloud compute instances describe k8s-control \
        --zone=us-central1-a \
        --format="get(networkInterfaces[0].accessConfigs[0].natIP)"

## 6. Namespace not created automatically from the UI

When creating an app through the ArgoCD UI, the CreateNamespace option is not enabled by default. Sync kept failing with namespace not found. Fixed by enabling Auto-Create Namespace in the app settings, or creating the namespace manually first.

## Debugging Flow — Cross-Node Networking

Working layer by layer from outside in:

1. kubectl get pods — Running, looked healthy
2. kubectl get endpoints — Service had real pod IPs
3. curl ClusterIP — hung
4. curl localhost NodePort — hung
5. curl pod IP directly — hung (ruled out Service and iptables)
6. curl node SSH port — connected instantly (ruled out VPC networking)
7. kubectl logs — saw DNS timeout (CoreDNS fix first)
8. ip route — blackhole route (red herring, turned out normal)
9. Calico reinstall — did not fix it
10. GCP firewall check — IP-in-IP protocol blocked
11. Added firewall rule — everything worked immediately

Key insight: when curl hangs silently and only cross-node traffic fails while node-to-node works — look at the underlying network infrastructure, not Kubernetes.
