Great question, Deepak! 🔄
**Argo CD Sync Policies** control **how and when** your Kubernetes applications get **synchronized** (i.e., deployed or updated) from Git to your cluster.

Let’s break it down:

---

## 🔁 What is Sync in Argo CD?

> **Sync** = Applying changes from Git (desired state) to the Kubernetes cluster (actual state).

Argo CD can:

* Sync **automatically** when Git changes
* Or you can **manually trigger** it from UI/CLI
* Sync **selectively** or **fully**, depending on settings

---

## 🧠 Types of Sync Policies

There are **2 main policies**:

| Type          | Description                               |
| ------------- | ----------------------------------------- |
| **Manual**    | You trigger sync manually (default)       |
| **Automated** | Argo CD auto-syncs on Git change or drift |

---

## ✅ 1. Manual Sync Policy

Default mode. You control sync via:

* Argo CD UI (click **Sync**)
* Argo CD CLI (`argocd app sync`)
* Argo CD API

**No `syncPolicy` block needed** in YAML.

---

## ✅ 2. Automated Sync Policy

Argo CD syncs automatically **whenever Git changes** or **drift is detected**.

### 🧾 YAML:

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```

Let’s break down each part 👇

---

## 🔧 `syncPolicy.automated`

### 🧩 `prune: true`

> Automatically **delete** cluster resources **removed from Git**.

🔧 Use case:

* You removed a `Job` or `Service` from Git — Argo will auto-delete it from the cluster.

---

### 🧩 `selfHeal: true`

> Automatically **re-sync** if cluster is modified manually.

🔧 Use case:

* Someone manually changes a Deployment (e.g., `kubectl edit`) → Argo will detect drift and fix it.

---

## 🧩 Example with `prune` and `selfHeal`

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

🔁 Argo CD will:

* Auto-sync on Git change
* Auto-fix drift
* Auto-delete removed resources

---

## 🔧 `syncPolicy.syncOptions`

This field provides **fine-grained control** of sync behavior.

### 🧾 All Common Sync Options:

| Option                          | Description                                                                       |
| ------------------------------- | --------------------------------------------------------------------------------- |
| `CreateNamespace=true`          | Auto-create the destination namespace if it doesn't exist                         |
| `ApplyOutOfSyncOnly=true`       | Only apply resources that are out of sync (faster sync)                           |
| `PruneLast=true`                | Apply all resources **before** pruning — avoids dependency issues during deletion |
| `ServerSideApply=true`          | Use `kubectl apply --server-side` for syncs                                       |
| `RespectIgnoreDifferences=true` | Skip diff fields defined in `ignoreDifferences` config                            |
| `Replace=true`                  | Use `kubectl replace` instead of apply (delete + recreate)                        |
| `Validate=false`                | Skip schema validation (e.g., to apply CRDs before CRs)                           |

---

### 📘 Example: Create namespace if missing

```yaml
syncOptions:
  - CreateNamespace=true
```

🔧 Use case: You want Argo CD to manage `dev`, `qa`, `prod` namespaces automatically.

---

### 📘 Example: Speed up sync by applying only changed resources

```yaml
syncOptions:
  - ApplyOutOfSyncOnly=true
```

🔧 Use case: Large app — avoid applying unchanged resources again.

---

### 📘 Example: Respect ignored fields

```yaml
syncOptions:
  - RespectIgnoreDifferences=true
```

🔧 Use case: You’ve configured `ignoreDifferences` for specific fields (like pod annotations) and want sync to skip them.

---

## 🎯 Full YAML Example: Automated Sync with All Options

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: banking-app
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  source:
    repoURL: https://github.com/deepak/banking-app
    path: k8s
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
      - PruneLast=true
```

---

## 🔐 Pro Tips

* Use `selfHeal: true` cautiously in **prod** if you're also managing via `kubectl`.
* Always use `prune: true` in **GitOps** to avoid orphaned resources.
* Combine with **resource hooks** (`Sync`, `PreSync`, `PostSync`) for blue-green or canary deployments.

---

## ✅ Summary Table

| Field            | Purpose                                    | Default |
| ---------------- | ------------------------------------------ | ------- |
| `automated`      | Enables auto-sync from Git/drift           | ❌       |
| `prune: true`    | Deletes cluster resources removed from Git | ❌       |
| `selfHeal: true` | Fixes manual drift                         | ❌       |
| `syncOptions`    | Fine-tune sync behavior                    | ❌       |

---

# **Argo CD Sync Policies & Options: Complete Guide with Examples**

Sync policies in Argo CD control **how and when** applications are updated from Git to Kubernetes. They determine automation behavior, resource pruning, and failure handling.

---

## **1. Sync Policy Basics**
Defined in the `Application` CRD under `spec.syncPolicy`, sync policies have 3 main components:

| Parameter | Description | Default |
|-----------|-------------|---------|
| **`automated`** | Enables auto-syncing when Git changes | Disabled |
| **`syncOptions`** | Fine-grained sync behaviors | None |
| **`retry`** | Retry failed syncs automatically | Disabled |

---

## **2. Automated Sync (`spec.syncPolicy.automated`)**
Enables **automatic synchronization** when the Git repo changes.

### **Key Parameters**
| Parameter | Description | Example Use Case |
|-----------|-------------|------------------|
| `prune: true/false` | Delete old resources no longer in Git | Clean up unused Services |
| `selfHeal: true/false` | Auto-correct cluster drift | Fix manual `kubectl` changes |
| `allowEmpty: true/false` | Allow empty manifests (dangerous) | Temporary app removal |

### **Example: Automated Sync with Pruning & Self-Healing**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-autosync-app
spec:
  syncPolicy:
    automated:
      prune: true       # Delete removed resources
      selfHeal: true    # Revert manual changes
      allowEmpty: false # Block empty applies
  source:
    repoURL: https://github.com/my-repo/prod-config.git
    path: kubernetes/
  destination:
    server: https://kubernetes.default.svc
```

---

## **3. Sync Options (`spec.syncPolicy.syncOptions`)**
Fine-tune sync behavior with these flags:

| Option | Description | Use Case |
|--------|-------------|----------|
| `Validate=false` | Skip `kubectl` validation | Fast syncs for trusted manifests |
| `CreateNamespace=true` | Auto-create target namespace | Avoid manual namespace creation |
| `PruneLast=true` | Delete resources last | Prevent service interruptions |
| `ApplyOutOfSyncOnly=true` | Only sync out-of-sync resources | Optimize performance |
| `RespectIgnoreDifferences=true` | Honor `ignoreDifferences` rules | Preserve manual tweaks |

### **Example: Sync with Namespace Auto-Creation**
```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true  # Auto-create namespace if missing
```

### **Example: Skip Validation for Large Manifests**
```yaml
syncPolicy:
  syncOptions:
    - Validate=false  # Bypass kubectl validation checks
```

---

## **4. Retry Policy (`spec.syncPolicy.retry`)**
Automatically retry failed syncs.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `limit` | Max retry attempts | 5 |
| `backoff.duration` | Initial delay between retries | 5s |
| `backoff.factor` | Multiplier for exponential backoff | 2 |
| `backoff.maxDuration` | Maximum delay between retries | 3m |

### **Example: Aggressive Retry for Flaky Clusters**
```yaml
syncPolicy:
  retry:
    limit: 10
    backoff:
      duration: 10s
      factor: 3
      maxDuration: 5m
```

---

## **5. Manual vs. Automated Sync**
### **Manual Sync (Default)**
- Changes require explicit user approval via UI/CLI.
- **Best for**: Production environments needing approvals.

```yaml
# No automated block = manual sync
syncPolicy: {}
```

### **Automated Sync**
- Argo CD applies changes immediately after Git commits.
- **Best for**: Dev/Test environments.

```yaml
syncPolicy:
  automated: {}
```

---

## **6. Advanced Use Cases**
### **Use Case 1: Zero-Downtime Deployments**
```yaml
syncPolicy:
  syncOptions:
    - PruneLast=true       # Delete old resources last
    - ApplyOutOfSyncOnly=true
  automated:
    selfHeal: true
```

### **Use Case 2: Preserving Manual Annotations**
```yaml
spec:
  ignoreDifferences:
    - group: ""
      kind: Deployment
      jsonPointers:
        - /metadata/annotations/owner
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
```

### **Use Case 3: Multi-Cluster Sync with Pruning**
```yaml
syncPolicy:
  automated:
    prune: true
  syncOptions:
    - CreateNamespace=true
```

---

## **7. Full Example: Production-Grade Sync Policy**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
spec:
  project: production
  source:
    repoURL: https://github.com/company/prod-configs.git
    targetRevision: main
    path: apps/backend
  destination:
    server: https://prod-cluster.example.com
    namespace: backend
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 8
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 10m
```

---

## **Summary Table**
| Feature | Parameter | Recommended For |
|---------|-----------|------------------|
| **Auto-Sync** | `automated: {}` | Dev/Test environments |
| **Pruning** | `prune: true` | Cleaning up old resources |
| **Self-Healing** | `selfHeal: true` | Enforcing Git as source of truth |
| **Namespace Creation** | `CreateNamespace=true` | Avoiding manual namespace setup |
| **Safe Deletion** | `PruneLast=true` | Zero-downtime deployments |

---

## **Key Takeaways**
✅ **Automated syncs** reduce manual intervention but require careful pruning rules.  
✅ **Sync options** optimize for safety (`PruneLast`) or speed (`Validate=false`).  
✅ **Retry policies** make deployments resilient to temporary failures.  
🚨 **Production Tip**: Combine `automated.prune` with `syncOptions.PruneLast` for safe deletions.  


---
# **Argo CD syncOptions**

Let’s explain **each sync option**, its **use case**, and provide **full YAML examples**. This is crucial for production-grade GitOps deployments.

---

## 🔧 What is `syncOptions`?

`syncOptions` is a field under `spec.syncPolicy` in an Argo CD Application that allows you to customize how sync happens.

```yaml
spec:
  syncPolicy:
    syncOptions:
      - <key>=<value>
```

Each option controls behavior like:

* Namespace creation
* Selective sync
* Replace vs apply
* Server-side apply
* Validation toggle

---

## ✅ All Argo CD `syncOptions` (with use cases)

---

### 1. `CreateNamespace=true`

**🔍 Description:**
Creates the destination namespace if it doesn’t exist before applying manifests.

**🧪 Use Case:**
When deploying to a namespace not created yet by infra (e.g., `dev`, `staging`).

**✅ Example:**

```yaml
syncOptions:
  - CreateNamespace=true
```

---

### 2. `ApplyOutOfSyncOnly=true`

**🔍 Description:**
Sync only the resources that are currently out of sync (skip unchanged ones).

**🧪 Use Case:**
Large apps where you want faster syncs by skipping unchanged resources.

**✅ Example:**

```yaml
syncOptions:
  - ApplyOutOfSyncOnly=true
```

---

### 3. `PruneLast=true`

**🔍 Description:**
Apply all new/changed resources **before** pruning deleted ones (to avoid dependency issues).

**🧪 Use Case:**
Avoid sync errors when dependent resources are deleted before new ones are created.

**✅ Example:**

```yaml
syncOptions:
  - PruneLast=true
```

---

### 4. `ServerSideApply=true`

**🔍 Description:**
Use `kubectl apply --server-side` instead of client-side apply.

**🧪 Use Case:**
Useful for advanced use cases with CRDs, RBAC, or to resolve apply conflicts.

**✅ Example:**

```yaml
syncOptions:
  - ServerSideApply=true
```

---

### 5. `RespectIgnoreDifferences=true`

**🔍 Description:**
Skips diffs during sync for fields defined under `ignoreDifferences`.

**🧪 Use Case:**
When ignoring autogenerated fields (e.g., `metadata.annotations`) or controller-managed fields.

**✅ Example:**

```yaml
syncOptions:
  - RespectIgnoreDifferences=true
```

---

### 6. `Replace=true`

**🔍 Description:**
Use `kubectl replace` instead of apply (effectively deletes and recreates).

**🧪 Use Case:**
Force recreation of a resource when apply fails (e.g., CRDs, immutable fields).

**✅ Example:**

```yaml
syncOptions:
  - Replace=true
```

---

### 7. `Validate=false`

**🔍 Description:**
Disables OpenAPI schema validation before applying manifests.

**🧪 Use Case:**
Useful when installing CRDs and CRs in the same sync — prevents validation errors for unknown resources.

**✅ Example:**

```yaml
syncOptions:
  - Validate=false
```

---

### 8. `SkipDryRunOnMissingResource=true`

**🔍 Description:**
Skips dry-run for resources not yet known to the cluster (e.g., new CRDs or missing API versions).

**🧪 Use Case:**
CRD + CR applied in same sync — avoids dry-run failure before CRD is registered.

**✅ Example:**

```yaml
syncOptions:
  - SkipDryRunOnMissingResource=true
```

---

### 9. `ApplyTimeMutations=true` (experimental)

**🔍 Description:**
Enables mutation hooks during apply time (relevant for custom Argo workflows, not common).

---

## 📘 Full YAML Example: Application with All Common `syncOptions`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: banking-app
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  source:
    repoURL: https://github.com/deepak/banking-app
    path: k8s
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
      - PruneLast=true
      - ServerSideApply=true
      - Replace=false
      - Validate=false
      - RespectIgnoreDifferences=true
```

---

## ✅ Summary Table

| Option                             | Default | Purpose                                                  |
| ---------------------------------- | ------- | -------------------------------------------------------- |
| `CreateNamespace=true`             | false   | Auto-create namespace if missing                         |
| `ApplyOutOfSyncOnly=true`          | false   | Apply only resources that have changed                   |
| `PruneLast=true`                   | false   | Apply new resources first, then prune                    |
| `ServerSideApply=true`             | false   | Use server-side apply logic                              |
| `Replace=true`                     | false   | Delete & recreate resource instead of patching           |
| `Validate=false`                   | true    | Skip schema validation (for early CRDs etc.)             |
| `RespectIgnoreDifferences=true`    | false   | Honors diff ignore settings during sync                  |
| `SkipDryRunOnMissingResource=true` | false   | Avoid dry-run errors for unknown resources like new CRDs |

---
