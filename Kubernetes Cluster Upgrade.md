# Kubernetes Cluster Upgrades

## 📑 Table of Contents
1. [Pre-requisites](#1-pre-requisites)
2. [What "Fully Managed" Means in EKS](#what-fully-managed-means-in-eks)
3. [Process](#2-process)
4. [Functional Validation Script](#5-functional-validation-script)
5. [IAM OIDC Provider](#5-IAM OIDC Provider)
6. [Pod Disruption Budget (PDB)](#4-pod-disruption-budget-pdb)

---

## 1. Pre-requisites  

- **Cordon Nodes**: Mark a node as *unschedulable* so that no new Pods are scheduled onto it, while existing Pods continue to run without interruption.  

- **Release Notes**:  
  - Show any big changes or removed features that might break your apps.  
  - Tell you about new tools, improvements, and security fixes you can use after upgrading.  
  - List bugs that got fixed and changes in main parts of Kubernetes (like API server, kubelet, scheduler).  

- **Downgrade Limitation**:  
  - You **cannot downgrade** a Kubernetes cluster version once upgraded.  
  - Kubernetes only supports moving forward in version numbers.  
  - If something breaks after an upgrade, you can’t just roll back — you’d have to restore from backups or rebuild the cluster.  

- **Test in Lower Environment**: Always test the upgraded Kubernetes version in lower environments first (like dev or staging). Keep a buffer time of 1–2 weeks.  

- **Align Control Plane and Node Versions** before moving to the next release.  

- **Cluster Autoscaler & Kubelet**: The version of the cluster autoscaler should match with the control plane version.  

- **Subnet IPs**: Ensure at least five IP addresses are available in the subnet.  

---

## 2. What "Fully Managed" Means in EKS  

AWS takes care of:  

- **Control plane provisioning and management** (API server, etcd, scheduler, controller manager).  
  - AWS creates and runs the main Kubernetes brain for you.  
  - For **minor version upgrades** (e.g., `1.26 → 1.27`), you must **initiate** the upgrade manually via AWS Console, CLI, or eksctl.  

- **High availability & fault tolerance** for the control plane (across multiple AZs).  
  - AWS runs the control plane in multiple data centers so it won’t go down if one fails.  

- **Automatic patching** of the control plane for security and bug fixes.  
  - AWS applies updates automatically without user intervention.  

- **Cluster API endpoint availability and scaling**.  
  - AWS ensures the API endpoint scales as your workloads grow.  

---

## 3. Process  

1. **Upgrade the control plane** (via Console or eksctl).  

2. **Upgrade Node Groups / Nodes / Fargate**:  
   - **Managed Node Groups (Launch Templates)**: Choose rollout strategy for zero downtime (nodes upgraded sequentially).  
   - **Custom Launch Templates (self-managed)**: Manually cordon nodes (unschedule workloads) and upgrade them one by one.  
   - **Hybrid Approach**: Combination of both.  

3. **Upgrade Add-ons** (e.g., kube-proxy, CoreDNS, VPC CNI).  

4. **Monitoring Stack (Helm, Prometheus, Grafana)**:  
   - No need to stop these before upgrade.  
   - Validate after upgrade to ensure monitoring is working in lower environments.  

---

## 4. Functional Validation Script  

After a Kubernetes cluster upgrade, run a validation script to test workloads, networking, and services.  

### cluster-validation.sh  

```bash
#!/bin/bash
# Simple functional validation script for upgraded Kubernetes cluster

set -e

NAMESPACE="upgrade-test"
APP_NAME="hello-app"

echo "Starting cluster validation tests..."

# 1. Check cluster health
echo "Checking cluster nodes..."
kubectl get nodes -o wide

echo "Verifying all nodes are Ready..."
kubectl get nodes | grep -v "Ready" && echo "Some nodes not Ready" || echo "All nodes Ready"

# 2. Create test namespace
kubectl create ns $NAMESPACE || true

# 3. Deploy a simple test app
echo "Deploying test app..."
kubectl -n $NAMESPACE create deployment $APP_NAME --image=nginx:stable --replicas=2

# 4. Expose the app via ClusterIP service
kubectl -n $NAMESPACE expose deployment $APP_NAME --port=80 --target-port=80

# 5. Wait for pods to be ready
echo "Waiting for pods to be ready..."
kubectl -n $NAMESPACE rollout status deployment/$APP_NAME

# 6. Run a test curl from inside the cluster
echo "Testing in-cluster connectivity..."
kubectl -n $NAMESPACE run curl-pod --image=busybox:1.28 --restart=Never -it --rm -- \
    wget -qO- http://$APP_NAME.$NAMESPACE.svc.cluster.local

# 7. Clean up
echo "Cleaning up..."
kubectl delete ns $NAMESPACE

echo "Cluster validation completed successfully!"

# What This Script Tests

After running the **cluster-validation.sh** script, the following checks are performed:

1. **Node Health**  
   - Ensures all Kubernetes nodes are in `Ready` state.

2. **Deployment**  
   - Deploys a sample Nginx application with multiple replicas.

3. **Service Discovery & Networking**  
   - Exposes the app via a ClusterIP service.  
   - Tests DNS resolution and internal service communication.

4. **Pod-to-Pod Communication**  
   - Uses a BusyBox pod to make a curl request to the Nginx service.

5. **Cleanup**  
   - Deletes the test namespace and resources after validation.


---

## 5.IAM OIDC Provider:  
    The **OIDC provider** in Amazon EKS connects **Kubernetes service accounts** with **AWS IAM roles**.  

    This enables secure, fine-grained permissions for pods using **IRSA (IAM Roles for Service Accounts)**.

## How It Works

1. Pod uses a **service account**.  
2. The service account is linked with an **OIDC token**.  
3. The OIDC token is mapped to an **IAM Role**.  
4. The IAM Role provides temporary credentials to access **AWS services** (e.g., S3, DynamoDB, SQS).

   ✅ No hardcoded credentials  
   ✅ Least-privilege access  

   👉 In simple words: OIDC is the **trust bridge** that lets pods securely call AWS services without storing secrets.  


---

## 6. Pod Disruption Budget (PDB)  

A **PDB** is a safeguard that tells Kubernetes:  
👉 *“Keep at least X pods running, even during upgrades or maintenance.”*  

Example:  

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: my-app
