# Kafka GitOps — Argo CD Migration Guide

A step-by-step guide for migrating a Strimzi Kafka cluster from one Kubernetes cluster to another using Argo CD and [Crane](https://github.com/konveyor/crane) for PVC transfer.

---

## Overview

| Role   | Cluster | Description                    |
|--------|---------|--------------------------------|
| **Source** | `test1` | Existing Kafka cluster with data |
| **Target** | `test2` | Destination cluster for migration |

**Assumptions:** Both clusters are on the same network. The source cluster runs a single-broker node pool (replicas = 1) managed by Strimzi and Argo CD.

**Repository:** The Kustomize manifests for Argo CD deployment are in: https://github.com/msajidmansoori12/kafka-gitops.git

---

## Prerequisites

- `kubectl` with contexts `test1` (source) and `test2` (target)
- [Crane](https://github.com/konveyor/crane) CLI for PVC transfer
- Argo CD installed on the source cluster (or wherever you manage applications)
- Nginx Ingress (or compatible) for Crane transfer endpoint
- Strimzi Kafka operator and Kafka resources defined in this repo (base + overlays)

---

## Migration Phases

### Phase 1 — Source: Deploy Kafka and Verify Data

#### Step 1: Deploy Kafka on the source cluster

On the source overlay, ensure:
- **Node pool replicas** are set to `1` (e.g. in `overlays/src/nodepool-replicas.yaml`)
- **Pause reconciliation** is `false` in `overlays/src/pause-reconciliation.yaml`

Create the Argo CD application and wait until the app is **Synced** and **Healthy**:

```bash
kubectl create -f argocd/src-argocd-application.yaml
# Wait for the Kafka app to be ready in Argo CD
```

#### Step 2: Produce test messages

Run a one-off producer pod to write messages to a test topic:

```bash
kubectl --context test1 -n kafka run kafka-producer -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-producer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic
```

Type a few messages (e.g. "sample message 1", "Sample message 2") and exit when done.

#### Step 3: Verify messages on the source cluster

Confirm that messages were written:

```bash
kubectl --context test1 -n kafka run kafka-consumer -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-consumer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic \
     --from-beginning
```

You should see the messages you produced. Exit with Ctrl+C when done.

#### Step 4: Note the Kafka broker PVC

List PVCs in the `kafka` namespace and note the **broker data PVC** name (you will use it for Crane):

```bash
kubectl get pvc --context test1 -n kafka
```

Example output:

```
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-my-cluster-my-cluster-pool-0   Bound    pvc-fb8b00da-934c-43db-9549-741405b812ea   1Gi        RWO            standard      3m21s
```

The PVC we migrate is **`data-my-cluster-my-cluster-pool-0`**.

---

### Phase 2 — Source: Quiesce and Prepare for PVC Transfer

We must ensure no process is writing to the PVC before migrating it with Crane.

#### Step 5: Pause reconciliation and remove the broker workload

1. **Pause reconciliation** on the source so Argo CD / Strimzi do not recreate the broker:
   - Set `pause-reconciliation` to **`true`** in `overlays/src/pause-reconciliation.yaml`
   - Sync the application (or apply the overlay) so the change takes effect.

2. **Delete the StrimziPodSet** that owns the Kafka broker pod. This stops the broker without deleting the PVC:

```bash
kubectl delete strimzipodset my-cluster-my-cluster-pool -n kafka --context test1
```

Expected:

```
strimzipodset.core.strimzi.io "my-cluster-my-cluster-pool" deleted from kafka namespace
```

#### Step 6: Confirm the broker pod is gone

Only the entity operator and Strimzi cluster operator should remain:

```bash
kubectl get po -n kafka --context test1
```

Example:

```
NAME                                                READY   STATUS    RESTARTS   AGE
my-cluster-entity-operator-7dd4d8564-d5nj7          2/2     Running   0          7m42s
strimzi-cluster-operator-v0.51.0-585788bf7b-x962t   1/1     Running   0          8m41s
```

There should be **no** `my-cluster-my-cluster-pool-0` pod.

---

### Phase 3 — Transfer PVC to the target cluster

#### Step 7: Create namespace and run Crane transfer

1. Create the `kafka` namespace on the target cluster:

```bash
kubectl create ns kafka --context test2
```

2. Run Crane to copy the PVC from source to target. Adjust `--subdomain` to match your environment (e.g. nip.io or your ingress domain):

```bash
./crane transfer-pvc \
  --source-context=test1 \
  --destination-context=test2 \
  --pvc-name=data-my-cluster-my-cluster-pool-0 \
  --pvc-namespace=kafka:kafka \
  --endpoint=nginx-ingress \
  --ingress-class=nginx \
  --subdomain=data-my-cluster-my-cluster-pool-0.kafka.172.18.0.3.nip.io
```

When the transfer finishes, you should see something like:

```
Status: Finishing up
Progress:
  Percentage:   100%
  Transferred:  83.01 K
  Rate:         1.52 MB/s
Elapsed:        0s
```

#### Step 8: Verify PVC on the target cluster

```bash
kubectl get pvc -n kafka --context test2
```

The PVC `data-my-cluster-my-cluster-pool-0` should be **Bound**.

---

### Phase 4 — Target: Deploy Kafka with same identity and fix permissions

Kafka stores a **cluster ID** in metadata. The target cluster must use the **same** cluster ID as the source so it recognizes the existing data. Also, after Crane copies the PVC, ownership may show as `nobody:nobody`; we fix that with a one-off job.

#### Step 9: Capture the cluster ID from the source

Run this **before** tearing down or reusing the source Kafka resource (or use a saved value):

```bash
kubectl get kafka my-cluster -n kafka --context test1 -o jsonpath='{.status.clusterId}' > kafka-cluster-id
```

The file `kafka-cluster-id` will contain a string like `IbnrtWdHRMyabShR1HgMKw`. Keep this for the next step.

#### Step 10: Start Kafka on target with reconciliation paused

1. In **`overlays/tgt/pause-reconciliation.yaml`**, set pause reconciliation to **`true`**.
2. Create the **target** Argo CD application (from the source or your Argo CD cluster):

```bash
kubectl create -f argocd/tgt-argocd-application.yaml --context test1
```

3. Check pods on the target. Only the Strimzi cluster operator should be running (no Kafka broker yet):

```bash
kubectl get po -n kafka --context test2
```

Example:

```
NAME                                                READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-v0.51.0-585788bf7b-tfknh   1/1     Running   0          47s
```

#### Step 11: Patch the Kafka resource with the source cluster ID

Replace `IbnrtWdHRMyabShR1HgMKw` with the value from `kafka-cluster-id` (or use `$(cat kafka-cluster-id)`):

```bash
kubectl patch kafka my-cluster \
  -n kafka \
  --subresource=status \
  --type merge \
  -p '{"status":{"clusterId":"IbnrtWdHRMyabShR1HgMKw"}}' \
  --context test2
```

This makes the target Kafka cluster use the same identity as the source so it can bind to the migrated PVC correctly.

#### Step 12: PVC permissions (1001:0)

After migration, the PVC on the target may have ownership `nobody:nobody` instead of `1001:0` that Kafka expects. This repo includes a job that fixes permissions:

- **`overlays/tgt/pvc-permission-job.yaml`** — runs when the app is synced and sets ownership on the PVC mount to `1001:0`.

Ensure this overlay is part of the target kustomization so the job runs (e.g. once per sync). After it completes, the broker pod can use the volume.

---

### Phase 5 — Target: Resume reconciliation and verify

#### Step 13: Allow reconciliation and sync

1. In **`overlays/tgt/pause-reconciliation.yaml`**, set pause reconciliation to **`false`**.
2. Sync the Kafka application on the target (Argo CD UI or CLI).

Wait for the Strimzi operator to reconcile. You should see the broker and entity operator pods come up:

```bash
kubectl get po -n kafka --context test2
```

Example:

```
NAME                                                READY   STATUS    RESTARTS   AGE
my-cluster-entity-operator-8cf776495-n2bs9          2/2     Running   0          31s
my-cluster-my-cluster-pool-0                        1/1     Running   0          58s
strimzi-cluster-operator-v0.51.0-585788bf7b-tfknh   1/1     Running   0          8m44s
```

#### Step 14: Confirm data on the target

Consume from the same topic on the target cluster:

```bash
kubectl --context=test2 -n kafka run kafka-consumer -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-consumer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic \
     --from-beginning
```

You should see the same messages you produced on the source (e.g. "sample message 1", "Sample message 2", etc.). This confirms the migration was successful.

---

## Summary checklist

| #  | Phase   | Action |
|----|--------|--------|
| 1  | Source | Deploy Kafka (Argo CD), produce and verify data, note PVC name |
| 2  | Source | Pause reconciliation, delete StrimziPodSet, confirm broker pod gone |
| 3  | Transfer | Create `kafka` ns on target, run `crane transfer-pvc`, verify PVC on target |
| 4  | Target | Save cluster ID from source; deploy target app with pause=true; patch cluster ID; ensure PVC permission job is applied |
| 5  | Target | Set pause=false, sync app, verify all pods running and consumer shows data |

---

## Key files

| File | Purpose |
|------|--------|
| `argocd/src-argocd-application.yaml` | Argo CD app for Kafka on the **source** cluster |
| `argocd/tgt-argocd-application.yaml` | Argo CD app for Kafka on the **target** cluster |
| `overlays/src/pause-reconciliation.yaml` | Pause reconciliation on source (true = quiesce for migration) |
| `overlays/tgt/pause-reconciliation.yaml` | Pause reconciliation on target (true = deploy without starting broker) |
| `overlays/tgt/pvc-permission-job.yaml` | Job to set PVC ownership to 1001:0 on target |
| `kafka-cluster-id` | Stored cluster ID from source (created in Step 9) |

---

## Notes

- **Cluster ID:** Never let the target Kafka resource get a new cluster ID before you patch it; otherwise it will not treat the migrated PVC as its own.
- **Permissions:** If the broker fails to start with permission errors on the volume, ensure the PVC permission job has run and the overlay is included in the target kustomization.
- **Crane subdomain:** The `--subdomain` value must resolve (e.g. via nip.io or your DNS) to the cluster/ingress you use for Crane.

If you hit issues, double-check: PVC name and namespace, cluster ID patch, and that reconciliation was paused/unpaused at the right times.

---

## Issues after PVC migration

Two issues must be addressed when using a PVC that was migrated from the source cluster to the target.

### 1. Cluster ID must match the source

The Kafka **cluster ID** from the source (test1) is stored in the PVC metadata. Kafka uses this identity to recognize its own data. If the target (test2) Kafka resource gets a new cluster ID at creation time, it will not treat the migrated PVC as belonging to the same cluster and will not attach to it correctly.

**What to do:** Before allowing the Strimzi operator to reconcile the broker on the target, patch the `Kafka` resource’s status with the **same** cluster ID as the source (e.g. by running `kubectl get kafka my-cluster -n kafka -o jsonpath='{.status.clusterId}'` on the source and then patching the target `Kafka` with that value). This ensures the target cluster is seen as the same logical cluster and binds to the migrated PVC.

### 2. PVC permissions (nobody:nobody → Kafka user)

On the source cluster, Kafka writes data to the PVC as a specific user (e.g. **1001:0**). After migrating the PVC to the target (test2), the volume data often shows ownership as **nobody:nobody**. If the broker pod then tries to use that volume, it can fail due to permission errors because Kafka expects its usual user/group.

**What to do:** The permissions on the migrated PVC must be changed to match what Kafka uses. Currently Strimzi/Kafka uses **1001:0**, but this may change in future versions, so keep that in mind when automating or documenting.

For the Argo CD deployment on the target, this is handled by a **Kubernetes Job** that mounts the same PVC, runs as a privileged or suitable user, and updates file ownership/permissions on the volume to the user and group that Kafka uses (e.g. 1001:0). This job is included in the target overlay (`overlays/tgt/pvc-permission-job.yaml`) and runs when the app is synced, so the permissions are fixed before (or in tandem with) the broker starting. If Kafka’s user/group changes in the future, update this job accordingly.
