# Tool Detection in Argo CD

## 🔍 What is **Tool Detection** in Argo CD?

**Tool detection** in Argo CD refers to its **automatic detection** of the **configuration/templating tool** used in your Git repository for Kubernetes manifests.

> 📌 Argo CD can **automatically detect and render** applications defined using:

* **Plain YAML**
* **Kustomize**
* **Helm**
* **Jsonnet**
* **Plugins**

---

## 🎯 Why Tool Detection Matters?

Because different tools structure and render manifests differently.
Argo CD needs to know **how to render** the manifests before applying them to the cluster.

If not specified manually, Argo CD tries to **auto-detect** the right tool based on the **files and folders** in the repository.

---

## 🧠 How Tool Detection Works

| Tool               | What Argo CD looks for in Git repo                   |
| ------------------ | ---------------------------------------------------- |
| **Kustomize**      | `kustomization.yaml` or `kustomization.yml` file     |
| **Helm**           | `Chart.yaml` file (indicates a Helm chart)           |
| **Jsonnet**        | `.jsonnet` or `jsonnetfile.json`                     |
| **YAML (default)** | No special files; assumes plain Kubernetes manifests |

---## 🔍 What is **Tool Detection** in Argo CD?

**Tool detection** in Argo CD refers to its **automatic detection** of the **configuration/templating tool** used in your Git repository for Kubernetes manifests.

> 📌 Argo CD can **automatically detect and render** applications defined using:

* **Plain YAML**
* **Kustomize**
* **Helm**
* **Jsonnet**
* **Plugins**

---

## 🎯 Why Tool Detection Matters?

Because different tools structure and render manifests differently.
Argo CD needs to know **how to render** the manifests before applying them to the cluster.

If not specified manually, Argo CD tries to **auto-detect** the right tool based on the **files and folders** in the repository.

---

## 🧠 How Tool Detection Works

| Tool               | What Argo CD looks for in Git repo                   |
| ------------------ | ---------------------------------------------------- |
| **Kustomize**      | `kustomization.yaml` or `kustomization.yml` file     |
| **Helm**           | `Chart.yaml` file (indicates a Helm chart)           |
| **Jsonnet**        | `.jsonnet` or `jsonnetfile.json`                     |
| **YAML (default)** | No special files; assumes plain Kubernetes manifests |

---
## **Defining the Tool in `Application` YAML**
You can explicitly specify the tool in the `Application` CRD under `spec.source`.

### **1. Plain Kubernetes YAML (No Tool)**
If your repo contains raw Kubernetes manifests (e.g., `deployment.yaml`), Argo CD applies them directly.

#### **Example:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-k8s-app
spec:
  source:
    repoURL: https://github.com/my-repo/k8s-manifests.git
    targetRevision: main
    path: manifests/  # Contains deployment.yaml, service.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
```

---
### **2. Helm Charts**
If your repo has a `Chart.yaml`, Argo CD treats it as a Helm chart.


## 🔰 Example 1: ✅ Basic Helm Deployment with `values.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-basic-helm-app
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  source:
    repoURL: https://github.com/deepak/helm-demo
    path: mychart
    targetRevision: main
    helm: {}   # <- uses default values.yaml
  project: default
```

---

## 🔰 Example 2: ✅ Using `valueFiles` (like `values-dev.yaml`)

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/helm-demo
    path: mychart
    targetRevision: main
    helm:
      valueFiles:
        - values-dev.yaml
```

> 🎯 Use case: Different environments (dev/staging/prod) have different configs.

---

## 🔰 Example 3: ✅ Overriding Values via `parameters`

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/helm-demo
    path: mychart
    targetRevision: main
    helm:
      parameters:
        - name: image.repository
          value: "deepak/banking-app"
        - name: image.tag
          value: "v2.1.0"
```

> 🎯 Use case: Override default Helm values inline.

---

## 🔰 Example 4: ✅ Using `releaseName` to Set a Custom Helm Release

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/helm-demo
    path: mychart
    targetRevision: main
    helm:
      releaseName: deepak-banking-prod
```

> 🎯 Use case: When you want to clearly label your release (visible in `helm list`).

---

## 🔰 Example 5: ✅ Combining All Helm Options (Recommended)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-helm-app
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  source:
    repoURL: https://github.com/deepak/helm-demo
    path: mychart
    targetRevision: main
    helm:
      releaseName: deepak-banking-app
      valueFiles:
        - values-dev.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🧠 Optional: What About `helm.version:`?

```yaml
helm:
  version: v2
```

> ✅ **Only needed if you're using Helm v2**, which is deprecated.
> 🔴 Not needed for Helm 3+ (default in Argo CD now).

---

## ✅ Summary Table:

| Use Case                     | Field               | Example              |
| ---------------------------- | ------------------- | -------------------- |
| Use default values           | `helm: {}`          | `values.yaml`        |
| Use custom values file       | `valueFiles`        | `values-dev.yaml`    |
| Override values inline       | `parameters`        | `image.tag: v2.0.1`  |
| Set custom Helm release name | `releaseName`       | `deepak-banking-app` |
| Use specific Helm version    | `version` (v2 only) | `version: v2`        |

---
### **3. Kustomize**
If your repo has a `kustomization.yaml`, Argo CD uses Kustomize to build manifests.

In **Argo CD**, when using **Kustomize**, you can **explicitly define the Kustomize version** in the `Application` manifest.

This is useful when:

* You want a specific version (e.g., due to a feature or behavior change).
* You are using syntax or features supported only in that version.

---

## ✅ Syntax Example: Using Kustomize with Specific Version

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kustomize-app
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  source:
    repoURL: https://github.com/deepak/kustomize-demo
    targetRevision: main
    path: overlays/dev
    kustomize:
      version: v3.5.4
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🔍 Why specify `version:`?

* Argo CD uses **Kustomize binaries internally**.
* Different versions may handle features differently, e.g.:

  * Helm transformer support
  * Secret generators
  * Patch strategies
* Specifying ensures **consistency** and **avoids unexpected behavior** during render time.

---

## 🧠 Default Behavior (if version is not specified)

* Argo CD uses the **default embedded version** of Kustomize shipped with it (might vary by Argo CD version).

---

## 🧪 Real Scenario Example:

Let’s say your `kustomization.yaml` uses:

```yaml
configurations:
  - kustomizeconfig.yaml
```

This feature requires **Kustomize v3.5.4+** — so you'd explicitly specify it in your Argo CD application to avoid errors.

---

## ✅ Summary

| Property            | Purpose                                     |
| ------------------- | ------------------------------------------- |
| `kustomize.version` | Specifies which Kustomize binary to use     |
| Required?           | Optional, but recommended for compatibility |
| Example             | `version: v3.5.4`                           |

---

## ✅ Overview: Argo CD Supported Tools

| Tool               | Description                                   | Auto-Detected? | Requires Plugin? |
| ------------------ | --------------------------------------------- | -------------- | ---------------- |
| **Plain YAML**     | Raw Kubernetes manifests                      | ✅ Yes          | ❌ No             |
| **Kustomize**      | Patchable YAML layering                       | ✅ Yes          | ❌ No             |
| **Helm**           | Templating with values/config                 | ✅ Yes          | ❌ No             |
| **Jsonnet**        | Data templating language (like advanced JSON) | ✅ Yes          | ❌ No             |
| **Custom Plugins** | Any tool via scripts                          | ❌ No           | ✅ Yes            |

---

## 🔹 1. Plain YAML – *(Default fallback)*

### 📁 Repo Structure

```
app/
├── deployment.yaml
├── service.yaml
```

### 📜 Application YAML

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/plain-yaml-app
    path: app
    targetRevision: main
```

> 🎯 No extra config needed — Argo CD applies raw manifests as-is.

---

## 🔹 2. Kustomize – *(Detected via `kustomization.yaml`)*

### 📁 Repo Structure

```
app/
├── kustomization.yaml
├── deployment.yaml
```

### 📜 Application YAML

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/kustomize-app
    path: app
    targetRevision: main
    kustomize:
      version: v3.5.4
```

> 🎯 Used for environment overlays or reusable YAML bases.

---

## 🔹 3. Helm – *(Detected via `Chart.yaml`)*

### 📁 Repo Structure

```
app/
└── Chart.yaml
    templates/
    values.yaml
```

### 📜 Application YAML

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/helm-app
    path: app
    targetRevision: main
    helm:
      releaseName: deepak-prod
      valueFiles:
        - values-prod.yaml
```

---

## 🔹 4. Jsonnet – *(Detected via `.jsonnet` files)*

### 📁 Repo Structure

```
app/
├── main.jsonnet
├── lib/
│   └── utils.libsonnet
```

### 📜 Application YAML

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/jsonnet-app
    path: app
    targetRevision: main
    directory:
      recurse: true
      jsonnet:
        extVars:
          - name: environment
            value: "dev"
```

> 🎯 Use when generating complex Kubernetes configs programmatically.

---

## 🔹 5. Custom Plugins – *(Run any tool: Ansible, Terraform, etc.)*

When built-in support doesn’t exist (e.g., for **Terraform**, **Pulumi**, **Ansible**), you can register a **custom plugin** in Argo CD.

### 🧩 Example Plugin Use: Ansible or Terraform

**Plugin Configuration (on Argo CD controller):**

```yaml
configManagementPlugins:
- name: terraform
  generate:
    command: ["/bin/bash", "-c"]
    args: ["terraform init && terraform plan -out plan && terraform show -json plan | your-k8s-renderer"]
```

**Application YAML:**

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/terraform-k8s
    path: .
    targetRevision: main
    plugin:
      name: terraform
```

> 🎯 This is powerful but requires changes in Argo CD controller config map (`argocd-cm`).

---

## 🔄 Auto-Detection Priority (if multiple tools exist)

1. If explicitly declared (e.g., `helm:`, `kustomize:`) → Argo CD uses that
2. If not:

   * Detects `kustomization.yaml` → uses Kustomize
   * Else detects `Chart.yaml` → uses Helm
   * Else detects `.jsonnet` → uses Jsonnet
   * Else uses plain YAML

---

## ✅ Summary Table with YAML Examples

| Tool           | Detection Method       | Argo CD Config Section     | Plugin Needed? | Example |
| -------------- | ---------------------- | -------------------------- | -------------- | ------- |
| **Plain YAML** | No config files needed | `source.path` only         | ❌ No           | ✔️      |
| **Kustomize**  | `kustomization.yaml`   | `source.kustomize`         | ❌ No           | ✔️      |
| **Helm**       | `Chart.yaml`           | `source.helm`              | ❌ No           | ✔️      |
| **Jsonnet**    | `.jsonnet` files       | `source.directory.jsonnet` | ❌ No           | ✔️      |
| **Plugin**     | None (manual setup)    | `source.plugin.name`       | ✅ Yes          | ✔️      |

---

