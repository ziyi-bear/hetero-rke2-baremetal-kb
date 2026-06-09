---
title: Rancher Continuous Delivery (Fleet) Backup and State Management
tags: [rancher, continuous-delivery, fleet, backup, disaster-recovery, etcd, rke2, gitops]
summary: Explains how Rancher Continuous Delivery manages its state natively within Kubernetes etcd, why local cluster etcd backups are sufficient for disaster recovery, and the role of Git as the source of truth.
---

# Rancher Continuous Delivery: State Management & Backup Strategy

## Overview
Rancher Continuous Delivery is powered by **Fleet**, a GitOps engine. Unlike traditional CI/CD tools that require standalone external databases (like PostgreSQL or MySQL), Fleet is completely Kubernetes-native. 

**Core Concept:** Fleet stores all of its configuration, deployment status, and operational state directly as **Kubernetes Custom Resources within the local cluster's `etcd`**.

## What is Stored in etcd?
Because Fleet operates natively within the cluster, everything configured in the Rancher Continuous Delivery UI is saved in `etcd`. This includes:
* **GitRepos:** URLs, paths, and polling configurations for target repositories.
* **Clusters & ClusterGroups:** Definitions and mapping of target downstream clusters.
* **Bundles:** The compiled Helm charts and deployment manifests Fleet is preparing to distribute.
* **Authentication:** SSH keys and access tokens used to connect to private Git repositories (stored as Kubernetes `Secrets` in the `fleet-default` or `fleet-local` namespaces).

## Backup Strategy & S3 Integration
If your local RKE2 management cluster is configured to push `etcd` snapshots to an external storage location (such as an S3 bucket), **your Rancher Continuous Delivery configurations are fully backed up.**

### Disaster Recovery (DR) Scenario
If the management cluster experiences a catastrophic failure:
1. Restoring the RKE2 `etcd` snapshot from S3 onto a new cluster will immediately restore the entire operational state of Rancher and Fleet.
2. Downstream clusters will automatically check back in with the restored management server.
3. Fleet will resume its GitOps syncing processes automatically.

## The GitOps Source of Truth
While the `etcd` backup protects the Rancher/Fleet *configuration* and cluster bindings, the ultimate source of truth in a GitOps model is **Git**. 

A complete DR strategy relies on two distinct backups:
1. **Infrastructure/Platform State:** RKE2 `etcd` backups in S3 (Protects Rancher/Fleet configurations).
2. **Application/Deployment State:** Git repository redundancy provided by the Git host (e.g., GitHub, GitLab) (Protects application manifests, Helm values, and infrastructure-as-code).

## Note on the Rancher Backup Operator
The **Rancher Backup Operator** is an optional Helm chart designed to back up only Rancher-specific and Fleet-specific resources as a `.tar.gz` file. 

* **Is it required?** No. 
* If a system is already performing full RKE2 `etcd` cluster backups to S3, the Rancher Backup Operator is redundant. The `etcd` snapshot provides a comprehensive, lower-level backup that encompasses everything the Operator would capture, plus the fundamental state of the cluster.