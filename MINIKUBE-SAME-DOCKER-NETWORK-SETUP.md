# Minikube Same-Docker-Network: Full Guide

**One setup for transfer-pvc and ArgoCD multi-cluster**

This guide walks through a single Minikube topology: two clusters on a **shared Docker network** with **static IPs**. You can use it for:

- **transfer-pvc** — migrate PVCs between clusters without tunnel or socat
- **ArgoCD** — deploy from one cluster (src) to another (tgt)

Both rely on the same foundation: one Docker network and two clusters with fixed IPs so they can reach each other by IP.

---

## Table of contents

1. [Overview](#overview)
2. [Part I: Foundation — Network and clusters](#part-i-foundation--network-and-clusters)
3. [Part II: transfer-pvc setup](#part-ii-transfer-pvc-setup)
4. [Part III: ArgoCD multi-cluster (optional)](#part-iii-argocd-multi-cluster-optional)
5. [Summary and reference](#summary-and-reference)
6. [Troubleshooting](#troubleshooting)

---

## Overview

| Item | Value |
|------|--------|
| **Docker network** | `minikube-mc` |
| **Source cluster (src)** | Profile `src`, IP `172.18.0.2` |
| **Target cluster (tgt)** | Profile `tgt`, IP `172.18.0.3` |
| **Kubeconfig** | Merged config with contexts `src` and `tgt` |

**Why this works**

- Pods in src can reach tgt’s node at `172.18.0.3`.
- For **transfer-pvc**: target ingress listens on 443 on that IP (via hostPort), so no tunnel or socat.
- For **ArgoCD**: src’s ArgoCD talks to tgt’s API at `https://172.18.0.3:8443`.

---

## Part I: Foundation — Network and clusters

Do this once. Everything else (transfer-pvc and ArgoCD) builds on it.

### 1. Create the shared Docker network

```bash
docker network create minikube-mc
```

Verify:

```bash
docker network ls
```

#### Inspect the network to see the IP range

Before assigning static IPs to Minikube, check which subnet Docker uses for this network:

```bash
docker network inspect minikube-mc
```

In the output, look for the **IPAM** section and its **Subnets** entry. For example:

```json
"IPAM": {
    "Subnets": {
        "172.18.0.0/16": {
            "IPsInUse": 5,
            "DynamicIPsAvailable": 65531
        }
    }
}
```

The subnet (e.g. `172.18.0.0/16`) is the range Docker can assign. We use **static IPs within that range** for the two clusters — in this guide, `172.18.0.2` (src) and `172.18.0.3` (tgt). If your network uses a different subnet (e.g. `10.0.0.0/24`), pick two free addresses in that range and use them in the `--static-ip` flags in the next steps.

---

### 2. Start the first cluster (src — source / ArgoCD)

```bash
minikube start -p src --driver=docker --network=minikube-mc --static-ip=172.18.0.2
```

This cluster will be the **source** for transfer-pvc and can run **ArgoCD** for Part III.

---

### 3. Start the second cluster (tgt — target)

```bash
minikube start -p tgt --driver=docker --network=minikube-mc --static-ip=172.18.0.3
```

Verify both profiles:

```bash
minikube profile list
```

---

### 4. Merge kubeconfig (contexts src and tgt)

Use one kubeconfig with contexts **src** and **tgt**. Minikube usually adds these when you use `-p src` and `-p tgt`:

```bash
cp ~/.kube/config /tmp/merged-kubeconfig
# If you use a custom KUBECONFIG, copy that instead.
```

Check contexts:

```bash
kubectl config get-contexts --kubeconfig=/tmp/merged-kubeconfig
```

You should see **src** and **tgt**. If your profile names differ, rename them:

```bash
kubectl config rename-context <profile1> src --kubeconfig=/tmp/merged-kubeconfig
kubectl config rename-context <profile2> tgt --kubeconfig=/tmp/merged-kubeconfig
```

From here on, use:

```bash
export KUBECONFIG=/tmp/merged-kubeconfig
```

---

### 5. Verify cluster-to-cluster connectivity

From a pod in **src**, the **tgt** API server should be reachable at `172.18.0.3:8443`:

```bash
kubectl config use-context src
kubectl run test --rm -it --image=busybox -- sh
```

Inside the container:

```bash
wget -qO- https://172.18.0.3:8443/version --no-check-certificate
```

You should see Kubernetes version JSON. That confirms: **src pod → tgt API server**.

---

## Part II: transfer-pvc setup

Use this when you want to run **crane transfer-pvc** between src (source) and tgt (target). No minikube tunnel or socat on the host.

**Idea:** transfer-pvc creates an Ingress on the target. The **source** rsync client pod connects to the **target node IP** on **port 443** (TLS). So the target must listen on 443 on `172.18.0.3` via the ingress controller **hostPort 443**.

### 1. Enable ingress and SSL passthrough on target (tgt)

Enable the ingress addon and wait for the controller:

```bash
export KUBECONFIG=/tmp/merged-kubeconfig

minikube addons enable ingress -p tgt
kubectl wait -n ingress-nginx --for=condition=available deployment/ingress-nginx-controller \
  --timeout=300s --context=tgt
```

Add SSL passthrough (required for transfer-pvc):

```bash
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --context=tgt --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--enable-ssl-passthrough"}]'
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx --context=tgt --timeout=300s
```

> **Note:** After restarting the target cluster, re-apply the SSL passthrough patch once ingress is up.

---

### 2. Expose port 443 on the target node (hostPort)

So that **172.18.0.3:443** serves the ingress, add **hostPort: 443** to the controller’s HTTPS port (usually the second port, index 1):

```bash
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --context=tgt --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/ports/1/hostPort", "value": 443}]'
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx --context=tgt --timeout=300s
```

If your controller’s HTTPS port is not at index 1, use the correct path (e.g. `ports/0`). After rollout, **172.18.0.3:443** serves the ingress.

---

### 3. Verify connectivity from source to target

From a pod in **src**, check that the target ingress is reachable:

```bash
kubectl run curl-verify --rm -it --restart=Never --image=curlimages/curl -n default --context=src -- \
  curl -k -s -o /dev/null -w "%{http_code}\n" --connect-timeout 10 https://172.18.0.3
```

A **404** (or any HTTP response) means the connection works. **000** or timeout means 443 is not reachable — check hostPort and that tgt’s node is really `172.18.0.3` (`minikube ip -p tgt`).

---

### 4. Run transfer-pvc

Use the **target node IP** in the subdomain. No tunnel or socat on the host.

**Subdomain format:** `<pvc-name>.<namespace>.172.18.0.3.nip.io` (nip.io resolves the hostname to 172.18.0.3).

**Example: PVC `data-pvc` in namespace `sajid-test`**

```bash
export KUBECONFIG=/tmp/merged-kubeconfig
./crane transfer-pvc \
  --source-context=src \
  --destination-context=tgt \
  --pvc-name=data-pvc \
  --pvc-namespace=sajid-test:sajid-test \
  --endpoint=nginx-ingress \
  --ingress-class=nginx \
  --subdomain=data-pvc.sajid-test.172.18.0.3.nip.io
```

**Example: PVC in namespace `kafka`**

```bash
./crane transfer-pvc \
  --source-context=src \
  --destination-context=tgt \
  --pvc-name=data-my-cluster-my-cluster-pool-0 \
  --pvc-namespace=kafka:kafka \
  --endpoint=nginx-ingress \
  --ingress-class=nginx \
  --subdomain=data-my-cluster-my-cluster-pool-0.kafka.172.18.0.3.nip.io
```

Wait until the command reports completion.

**Compared to tunnel + socat flow**

- No **minikube tunnel** or **socat** on the host.
- Subdomain uses the **target node IP** (172.18.0.3), not the host’s LAN IP.
- **hostPort 443** on the ingress controller is required.

---

## Part III: ArgoCD multi-cluster (optional)

Use this when you want **ArgoCD** in src to deploy applications to tgt. Complete **Part I** first (network, both clusters, merged kubeconfig with contexts **src** and **tgt**, and connectivity from src to tgt’s API at 172.18.0.3:8443).

### 1. Install ArgoCD in src

```bash
export KUBECONFIG=/tmp/merged-kubeconfig
kubectl config use-context src

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for ArgoCD to be ready (so step 2 can connect):

```bash
kubectl wait -n argocd --for=condition=available deployment/argocd-server --timeout=300s --context=src
```

---

### 2. Access ArgoCD UI

Port-forward the ArgoCD server (run in the foreground or background; keep it running for step 3):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open **http://localhost:8080**. Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

(Or: `argocd admin initial-password -n argocd` if using the ArgoCD CLI.)

Keep this port-forward running for step 3 (ArgoCD CLI login and cluster add).

---

### 3. Login to ArgoCD CLI and create the token secret on tgt

**Do this before** getting the token or registering the cluster. The `argocd cluster add` command **only** creates the `argocd-manager-long-lived-token` secret on tgt; step 4 uses that token to register the cluster with ArgoCD.

Ensure your kubeconfig has context **tgt** (from Part I). Install the [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) if needed. Keep the port-forward from step 2 running, then log in with the admin password from step 2:

```bash
argocd login localhost:8080 --username admin --insecure
```

When prompted, enter the password you obtained in step 2.

Then run `argocd cluster add tgt`. This **only** creates the `argocd-manager-long-lived-token` secret in the tgt cluster's `kube-system` namespace (so the next step can retrieve the token):

```bash
argocd cluster add tgt
```

When prompted, confirm. After this, the secret exists on tgt; you still need to register the cluster with ArgoCD using that token (step 4).

---

### 4. Get the token and register tgt in ArgoCD

Retrieve the token that step 3 created. This command only works after `argocd cluster add tgt` has run:

```bash
kubectl config use-context tgt
kubectl -n kube-system get secret argocd-manager-long-lived-token -o jsonpath='{.data.token}' | base64 -d
```

Save the output (the token). Then create the cluster registration Secret in ArgoCD. Create a file `cluster-tgt.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-tgt
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  name: tgt
  server: https://172.18.0.3:8443
  config: |
    {
      "bearerToken": "PASTE_TOKEN_HERE",
      "tlsClientConfig": {
        "insecure": true
      }
    }
```

Replace `PASTE_TOKEN_HERE` with the token you retrieved above. Apply the Secret to the **src** cluster (where ArgoCD runs) so ArgoCD can use it to talk to tgt:

```bash
kubectl config use-context src
kubectl apply -f cluster-tgt.yaml
```

---

### 5. Verify ArgoCD sees tgt

```bash
argocd cluster list
```

Example output:

```
SERVER                     NAME
https://kubernetes.default in-cluster
https://172.18.0.3:8443    tgt
```

---

### 6. Confirm tgt is reachable (optional)

Check that the tgt context works and that ArgoCD can target tgt for deployments:

```bash
kubectl --context tgt get pods
```

You can now deploy applications to tgt via ArgoCD using `--dest-server https://172.18.0.3:8443`.

---

## Summary and reference

### Architecture

```
Docker network: minikube-mc
        │
        ├── Minikube src (172.18.0.2)  ← source / ArgoCD
        │         │
        │         ├── transfer-pvc: source cluster
        │         └── ArgoCD (deploys to tgt)
        │
        └── Minikube tgt (172.18.0.3)  ← target
                  │
                  ├── transfer-pvc: ingress on :443 (hostPort)
                  └── ArgoCD: workloads deployed here
```

### Quick reference tables

**Part I — Foundation**

| Step | What |
|------|------|
| 1 | Create Docker network `minikube-mc` |
| 2 | Start src with `--network=minikube-mc --static-ip=172.18.0.2` |
| 3 | Start tgt with `--network=minikube-mc --static-ip=172.18.0.3` |
| 4 | Merge kubeconfig → contexts src, tgt |
| 5 | Verify src pod can reach https://172.18.0.3:8443/version |

**Part II — transfer-pvc**

| Step | What |
|------|------|
| 1 | Enable ingress on tgt, add **--enable-ssl-passthrough** |
| 2 | Add **hostPort: 443** to ingress controller HTTPS port |
| 3 | Verify from src: `curl -k https://172.18.0.3` → 404 or similar |
| 4 | Run **transfer-pvc** with **--subdomain=...172.18.0.3.nip.io** |

**Part III — ArgoCD**

| Step | What |
|------|------|
| 1 | Install ArgoCD in src, wait for argocd-server to be ready |
| 2 | Port-forward argocd-server (keep running), get admin password |
| 3 | `argocd login localhost:8080`, then `argocd cluster add tgt` (only creates secret on tgt) |
| 4 | Get token from tgt, create cluster Secret YAML with server 172.18.0.3:8443, apply to src |
| 5 | `argocd cluster list` to verify tgt appears |
| 6 | (Optional) Confirm tgt reachable; deploy apps via ArgoCD with --dest-server https://172.18.0.3:8443 |

---

## Troubleshooting

### transfer-pvc

- **Connection timeout or 000 from source pod to 172.18.0.3:443**  
  Confirm tgt’s IP: `minikube ip -p tgt`. Ensure the hostPort 443 patch was applied and the ingress controller pod is running. Check both clusters are on the same Docker network and that a src pod can reach 172.18.0.3.

- **“Endpoint is unhealthy”**  
  Ingress on the target is not ready or not reachable. Re-check Part II steps 1 and 2 and the connectivity check in step 3.

- **SSL or certificate errors**  
  Re-apply the SSL passthrough patch (Part II step 1) and wait for rollout.

- **Different IPs**  
  If your target node is not 172.18.0.3, use your target IP everywhere: hostPort is still 443 on that node, and use that IP in the subdomain (e.g. `...172.18.0.5.nip.io`).

### ArgoCD

- **Cannot reach tgt from ArgoCD**  
  From a pod in src, test `wget -qO- https://172.18.0.3:8443/version --no-check-certificate`. If that fails, fix networking (Part I step 5) first.

- **Cluster not appearing in `argocd cluster list`**  
  Check the cluster secret in `argocd` namespace, correct server URL (`https://172.18.0.3:8443`), and valid bearer token.
