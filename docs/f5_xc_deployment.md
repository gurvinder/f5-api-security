# 🌐 F5 Distributed Cloud (F5XC) Deployment on OpenShift

> **Document Version:** 1.0  
> **Author:** Eric Ji  
> **Last Updated:** November 2025  

---

## 🎯 Objective

Integrate **F5 Distributed Cloud Mesh (Mesh)** with a **single Red Hat OpenShift (OCP)** cluster  
by deploying the **F5XC site as pods directly within the cluster**.  
This deployment automatically discovers cluster services through the Kubernetes API.

---

## 🧩 Prerequisites

| Requirement | Configuration (My Deployment) | Notes |
|--------------|-------------------------------|--------|
| **Cluster Node** | `api.gpu-ai.bd.f5.com` (Control-plane, master, worker, worker-hp) | ✅ Single-node OCP |
| **Kubernetes Version** | `v1.31.6` | Supported: OCP ≥ 4.7 |
| **Resources** | ≥ 4 vCPUs, 8 GB memory | Recommended |
| **StorageClass** | `lvms-vg1 (default)` | Dynamic PVC enabled |

> ⚙️ **Tip:** Ensure your cluster has a default StorageClass for automatic PVC binding.

---

## ⚙️ Step 1: Configure OpenShift Environment

### 🔍 1.1 Verify Cluster Health
```bash
oc get nodes
```
```
NAME                STATUS   ROLES                                    AGE   VERSION
api.gpu-ai.bd.f5.com   Ready    control-plane,master,worker,worker-hp   221d  v1.31.6
```

<details>
<summary>💬 No Pending Pods</summary>

```bash
oc get pod -A | egrep -vi 'Running|Completed'
```
> No pending or failed pods detected.
</details>

---

### 💾 1.2 Configure Kernel HugePages

1. Label the node:
   ```bash
   oc label node api.gpu-ai.bd.f5.com node-role.kubernetes.io/worker-hp=""
   ```
2. Apply configurations:
   ```bash
   oc apply -f hugepages-tuned-boottime.yaml
   oc apply -f hugepages-mcp.yaml
   ```
3. Verify allocation:
   ```bash
   oc describe node api.gpu-ai.bd.f5.com | grep HugePages
   ```

> 💡 **HugePages** are required for CE site pods to manage high-performance memory workloads efficiently.

---

### 🗄️ 1.3 Validate StorageClass
```bash
oc get sc
```
✅ Confirm that `lvms-vg1 (default)` exists and allows dynamic volume provisioning.

---

## 🚀 Step 2: Deploy F5XC Cloud Mesh Pod

### 🧱 2.1 Download CE Manifest
```bash
curl -O https://gitlab.com/volterra.io/volterra-ce/-/raw/master/k8s/ce_k8s.yml
```

> 🧩 **Edit for your environment:**  
> - Rename to `ce_ocp_gpu-ai.yml`  
> - Remove NodePort definitions if deploying single site.

---

### 📦 2.2 Apply Manifest
```bash
oc create -f ce_ocp_gpu-ai.yml
```

<details>
<summary>📋 Expected Output</summary>

```bash
namespace/ves-system created
serviceaccount/volterra-sa created
statefulset.apps/vp-manager created
...
```
</details>

---

### 🧾 2.3 Verify Persistent Volume Claims
```bash
oc -n ves-system get pvc
```
| PVC Name | Status | StorageClass |
|-----------|---------|---------------|
| data-vp-manager-0 | Bound | lvms-vg1 |
| etcvpm-vp-manager-0 | Bound | lvms-vg1 |
| varvpm-vp-manager-0 | Bound | lvms-vg1 |

---

### 🛠️ 2.4 Troubleshoot & Register Site
**Symptom:** `prometheus` pod in `CrashLoopBackOff`  
**Cause:** `hostPort` conflicts in Prometheus deployment.

Fix:
```bash
oc -n ves-system edit deploy/prometheus
```
➡️ Remove lines containing `hostPort: 65210–65221`

---

### ✅ Final Pod Status
```bash
oc get pod -n ves-system -o wide
```
All components should show **Running** status.

---

## 🌍 Step 3: Deploy Application (Hipster Shop)
Follow standard OpenShift app deployment procedures:
```bash
oc new-project z-ji
oc apply -f hipster-shop.yaml
```
> 🧩 The `frontend` service can be type `ClusterIP` since F5XC Mesh handles service discovery internally.

---

## 🌐 Step 4: Advertise Services via F5XC Console

### 🏗️ Create Origin Pool
1. Navigate → **Multi-Cloud App Connect → Origin Pools**
2. Add Pool → Type: `K8s Service Name of Origin Server`
3. Example: `frontend.z-ji`
4. Site: Select deployed Mesh site  
5. Network: `Outside Network`

### 🌎 Create HTTP Load Balancer
1. Navigate → **HTTP Load Balancers**
2. Define domain → Reference Origin Pool  
3. Save & Exit

> ✅ **Result:** Application pods appear as **origin servers** in F5XC Console, accessible via configured domain.

---

## 🧭 Summary

🎯 Mesh deployed as pods inside OCP → direct access to Kubernetes API  
💡 No NodePort or kubeconfig access needed  
🔒 Simplified service advertisement with F5 Distributed Cloud Mesh

---

> © 2025 F5 Networks – Internal Reference Guide
