# Kafka GitOps

Kustomize manifests and guides for running Strimzi Kafka with Argo CD and migrating between clusters.

## Documentation
- **[Minikube same-Docker-network setup](MINIKUBE-SAME-DOCKER-NETWORK-SETUP.md)** — Full guide for two Minikube clusters on a shared Docker network with static IPs, used for Crane transfer-pvc and Argo CD multi-cluster deployment.


- **[Kafka migration with Argo CD and Crane](MIGRATING-KAFKA-WITH-CRANE.md)** — Step-by-step guide for migrating a Strimzi Kafka cluster from one Kubernetes cluster to another using Argo CD and [Crane](https://github.com/konveyor/crane) for PVC transfer.

- **[Migrating Kafka using MirrorMaker2](MIGRATING-KAFKA-USING-MIRRORMAKER2.md)** — Step-by-step guide for migrating Kafka from a source to a target cluster using MirrorMaker2 and Argo CD (application-level replication, no PVC transfer).

