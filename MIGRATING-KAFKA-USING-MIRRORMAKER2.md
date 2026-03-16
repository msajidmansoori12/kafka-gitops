# Migrating Kafka Using MirrorMaker2

A step-by-step guide for migrating a Kafka deployment from a **source** Kubernetes cluster to a **target** cluster using MirrorMaker2 and ArgoCD.

---

## Overview

This guide walks through:

1. Deploying Kafka on the source cluster via ArgoCD
2. Writing and verifying test data on the source
3. Deploying Kafka on the target cluster
4. Configuring MirrorMaker2 for cross-cluster replication
5. Verifying replication and live message flow
6. Stopping new messages and verifying replication with offset comparison
7. Cutover: promoting the target to primary and decommissioning the source

---

## Prerequisites

- Two Kubernetes clusters: **source** and **target**
- `kubectl` configured with two contexts (e.g. `src` and `tgt`, or `test1` and `test2`)
- ArgoCD installed on both clusters
- Strimzi Kafka Operator (managed via ArgoCD)
- Git repository with the kafka-gitops manifests

---

## Repo notes: demo app and target overlay

- **Demo app:** The base includes a sample app in `base/kafka`: `demo-app-configmap.yaml`, `demo-producer.yaml`, and `demo-consumer.yaml`. The producer sends a message every 10 seconds to the configured topic; the consumer reads from the beginning. Both use the ConfigMap `kafka-demo-app-config` (`KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_TOPIC`).
- **Target overlay patch:** For the target cluster, `overlays/tgt/demo-app-topic-patch.yaml` patches that ConfigMap so `KAFKA_TOPIC` is `source.test-topic` (the replicated topic name on the target). Without this patch, the demo app on the target would still point at `test-topic`, which is empty there.

---

## Step 1: Deploy Kafka on the Source Cluster

Create the ArgoCD application for Kafka on the source cluster:

```bash
kubectl create -f argocd/src-argocd-application.yaml
```

**Source cluster state:**

- Nodepool replicas: **1**
- Pause reconciliation patch: **false**

Wait for the application to be **Ready** and **Synced** in ArgoCD before continuing.

---

## Step 2: Write Test Data to the Source Cluster

Produce test messages to `test-topic` on the source:

```bash
kubectl --context src -n kafka run kafka-producer -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-producer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic
```

At the `>` prompt, type your test messages and press Enter after each. Use **Ctrl+C** to exit.

---

## Step 3: Verify Messages on the Source

Confirm that messages were written to the Kafka cluster:

```bash
kubectl --context src -n kafka run kafka-consumer -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-consumer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic \
     --from-beginning
```

You should see the messages you produced in Step 2.

---

## Step 4: Check PVCs on the Source

Verify PersistentVolumeClaims in the `kafka` namespace:

```bash
kubectl get pvc --context src -n kafka
```

Example output:

```
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-my-cluster-my-cluster-pool-0   Bound    pvc-fb8b00da-934c-43db-9549-741405b812ea   1Gi        RWO            standard      3m21s
```

---

## Step 5: Deploy ArgoCD on the Target Cluster

Install and configure ArgoCD on the target cluster in the same way as on the source (e.g. same version and setup).

---

## Step 6: Deploy the Kafka App on the Target Cluster

Create the ArgoCD application for Kafka on the target:

```bash
kubectl create -f argocd/tgt-argocd-application.yaml
```

Wait for the application to sync. The target Kafka cluster will be empty at this stage.

---

## Step 7: Verify the Target Cluster Is Empty

Confirm that the target has no messages in `test-topic` yet:

```bash
kubectl --context tgt -n kafka run kafka-consumer -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-consumer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic \
     --from-beginning \
     --timeout-ms 5000
```

**Expected:** No messages; the consumer times out after 5 seconds. This confirms the target is ready for replication.

---

## Step 8: Configure MirrorMaker2 for Cross-Cluster Replication

### 8.1 Expose the Source Kafka (Networking)

Get the NodePort and node IP for the source Kafka bootstrap service.

**Get the NodePort:**

```bash
kubectl get svc -n kafka my-cluster-kafka-external-bootstrap --context src
```

Example output:

```
NAME                                  TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
my-cluster-kafka-external-bootstrap   NodePort   10.103.134.137   <none>        9094:30297/TCP   38m
```

Note the port after the colon (e.g. **30297**).

**Get the node IP:**

```bash
kubectl get nodes -o wide --context src
```

Example output:

```
NAME   STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   ...
src    Ready    control-plane   69m   v1.35.1   172.18.0.2    <none>        ...
```

Note the **INTERNAL-IP** (e.g. **172.18.0.2**).

**Source bootstrap address for MirrorMaker2:** `INTERNAL-IP:NodePort` (e.g. `172.18.0.2:30297`).

---

### 8.2 Add MirrorMaker2 to the Git Repo

Ensure `overlays/tgt/mirrormaker2.yaml` exists and set the source cluster bootstrap address.

Under the `source` alias, set `bootstrapServers` to the address from Step 8.1:

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: mm2
  namespace: kafka
  annotations:
    argocd.argoproj.io/sync-wave: "4"
spec:
  version: 4.2.0
  replicas: 1
  connectCluster: "target"
  clusters:
    - alias: "source"
      bootstrapServers: "172.18.0.2:30297"   # <-- Your source IP:NodePort
      config:
        ssl.endpoint.identification.algorithm: ""
    # ... target cluster config ...
```

---

### 8.3 Deploy MirrorMaker2 via ArgoCD

Commit and push your changes, then sync the target ArgoCD application:

```bash
git add overlays/tgt/mirrormaker2.yaml
git add overlays/tgt/kustomization.yaml   # if updated
git commit -m "Add MirrorMaker2 for cross-cluster replication"
git push
```

ArgoCD will sync automatically, or trigger manually:

```bash
argocd app sync kafka-target
```

---

## Step 9: Verify MirrorMaker2 Is Replicating

Check the MirrorMaker2 resource status:

```bash
kubectl describe kafkamirrormaker2 mm2 -n kafka --context tgt
```

In the **Status** section, confirm:

| Connector                         | State   |
|-----------------------------------|---------|
| source→target.MirrorSourceConnector     | RUNNING |
| source→target.MirrorCheckpointConnector | RUNNING |
| source→target.MirrorHeartbeatConnector  | RUNNING |

**Conditions** should show `Type: Ready`, `Status: True`.

---

## Step 10: List Replicated Topics on the Target

```bash
kubectl --context tgt -n kafka run kafka-topics -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-topics.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --list
```

You should see internal MM2 topics and the replicated topic:

- `source.test-topic` — replicated topic (prefixed with `source.`)
- `mm2-configs.source.internal`, `mm2-offsets.source.internal`, etc.

---

## Step 11: Verify Data Replicated Successfully

Consume from the **replicated** topic on the target (note: `source.test-topic`, not `test-topic`):

```bash
kubectl --context tgt -n kafka run kafka-consumer -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-consumer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic source.test-topic \
     --from-beginning
```

You should see the same test messages that were written on the source.

---

## Step 12: Verify Live Replication (Optional)

Confirm that new messages are continuously replicated.

**Terminal 1 — consumer on target (leave running):**

```bash
kubectl --context tgt -n kafka run kafka-consumer-live -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-consumer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic source.test-topic
```

**Terminal 2 — producer on source:**

```bash
kubectl --context src -n kafka run kafka-producer-live -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-console-producer.sh \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic
```

Type a few messages at the `>` prompt. Within seconds they should appear in Terminal 1 on the target.

---

## Step 13: Stop new messages on the source (before cutover)

So that no new messages are written before cutover, scale the demo producer to 0 on the source cluster:

```bash
kubectl scale deployment kafka-demo-producer --replicas=0 -n kafka --context src
```

Use `--context test1` (or your source context name) if different from `src`. If you use other producers, stop or scale them down as well.

---

## Step 14: Verify replication with offset comparison

Confirm that the target has caught up by comparing end offsets on source and target. The counts should match.

**On source (`test-topic`):**

```bash
kubectl -n kafka run kafka-check-offsets -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  --context src \
  -- bin/kafka-run-class.sh kafka.tools.GetOffsetShell \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic test-topic \
     --time -1
```

Example output:

```
test-topic:0:57
test-topic:1:9
test-topic:2:0
```

**On target (`source.test-topic`, the replicated topic):**

```bash
kubectl --context tgt -n kafka run kafka-check-offsets-target -ti \
  --image=quay.io/strimzi/kafka:0.38.0-kafka-3.6.0 \
  --rm=true \
  --restart=Never \
  -- bin/kafka-run-class.sh kafka.tools.GetOffsetShell \
     --bootstrap-server my-cluster-kafka-bootstrap:9092 \
     --topic source.test-topic \
     --time -1
```

Example output (must match source partition counts):

```
source.test-topic:0:57
source.test-topic:1:9
source.test-topic:2:0
```

If the offset counts match, replication was successful. If the target lags, wait and re-run until they match before cutover.

---

## Step 15: Cutover — promote target to primary

1. **Stop all producers and consumers** that still point at the source cluster (scale down or update config).
2. **Wait for MirrorMaker2 to drain** any in-flight messages. Re-run the offset check on the target (Step 14) until it matches the source if needed.
3. **Update application configs** (ConfigMaps, env vars) to use the target cluster bootstrap address and, if using replicated topics, the topic name (e.g. `source.test-topic`).
4. **Remove MirrorMaker2 from the target overlay** if used: remove `overlays/tgt/mirrormaker2.yaml` from `overlays/tgt/kustomization.yaml` and delete the `mirrormaker2.yaml` file, then commit and push so ArgoCD syncs and removes the MirrorMaker2 resources.
5. **Redirect traffic** to the target cluster and scale producers/consumers back up pointing at the target.
6. **Decommission the source** after the target is stable: delete the source ArgoCD application (e.g. with cascade) and, when satisfied, delete the source cluster PVCs/namespace if desired.

---

## Summary

| Step   | Action |
|--------|--------|
| 1      | Deploy Kafka on source via ArgoCD |
| 2–3    | Write and verify test data on source |
| 4      | Check PVCs on source |
| 5–6    | Deploy ArgoCD and Kafka on target |
| 7      | Confirm target is empty |
| 8      | Expose source Kafka, add MirrorMaker2 manifest, push and sync |
| 9–10   | Verify MM2 connectors and replicated topics |
| 11–12  | Verify data and live replication using `source.test-topic` |
| 13     | Stop new messages on source (scale demo producer to 0) |
| 14     | Verify replication with offset comparison (source vs target) |
| 15     | Cutover: promote target, update configs, remove MM2, decommission source |

> **Topic naming:** Replicated topics on the target use the `source.` prefix (e.g. `source.test-topic`). When cutting over, either update application config to use `source.test-topic` or use MirrorMaker2 topic renaming if you need the original name.

For more background, see the [Kafka Migration Guide: ArgoCD + MirrorMaker2](https://gist.github.com/jwmatthews/e747a24a9341cc5537d1a73ea26d6886) (Part 5).
