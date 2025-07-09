# **Argo CD**

## 🔷 What is **Argo CD**?

**Argo CD (Argo Continuous Delivery)** is a **declarative GitOps tool** for Kubernetes. It continuously monitors your Git repositories and **automatically syncs** the changes (manifests) to your Kubernetes cluster.

> 📌 Think of Argo CD as a Kubernetes **deployment manager** that ensures your cluster always matches the **state defined in Git**.

---

## 🧠 Argo CD – **Core Concepts**

| Core Concept       | Description                                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| **Application**    | The main Argo CD resource. It defines **what to deploy**, **from where (Git repo)**, and **to which cluster/namespace**. |
| **Repository**     | Git repository containing your YAML/Helm/Kustomize manifests.                                                            |
| **Target Cluster** | Kubernetes cluster where the app is deployed. Argo CD can manage multiple clusters.                                      |
| **Sync**           | Syncing ensures your live cluster matches what's defined in Git.                                                         |
| **Rollback**       | You can revert to previous Git commits to rollback changes.                                                              |
| **Health Status**  | Indicates if Kubernetes resources (Pods, Services) are healthy.                                                          |
| **Sync Policy**    | Manual or Automated. Auto-sync applies changes as soon as Git is updated.                                                |

---

## 🏗️ Argo CD Architecture Overview

Argo CD follows a **controller-based architecture** with these components:

### **1. Argo CD API Server**
- **REST API & gRPC** for CLI/UI interactions.
- **Authentication (RBAC, SSO)** for security.
- **Manages Application CRDs**.

### **2. Repository Server**
- **Fetches manifests** from Git/Helm/OCI.
- **Generates Kubernetes YAML** (for Helm/Kustomize).

### **3. Application Controller**
- **Compares Git state vs. cluster state**.
- **Triggers sync operations** if drift is detected.
- **Self-heals** by reapplying Git state.

### **4. Redis Cache**
- **Stores application state** for fast lookups.
- **Improves performance** for large deployments.

### **5. Dex (Optional - SSO Integration)**
- **Enables OAuth2/OIDC** (GitHub, Google, LDAP).

---

```
              +----------------------------+
              |        Git Repository      |
              |  (App YAML/Helm/Kustomize) |
              +-------------+--------------+
                            |
                            v
         +------------------------+      +-----------------------+
         |     Repo Server        | ---> |   Application CRD     |
         +------------------------+      +-----------------------+
                            |
                            v
         +-----------------------------+
         |   Application Controller    |
         |  - Diff Git vs Cluster      |
         |  - Sync changes             |
         +-----------------------------+
                            |
                            v
              +----------------------+
              |  Kubernetes Cluster  |
              +----------------------+

```

## **Example: Deploying an App with Argo CD**
### **Step 1: Install Argo CD**
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### **Step 2: Access the UI**
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
- Open `https://localhost:8080` (default login: `admin`, password from `argocd-initial-admin-secret`).

### **Step 3: Deploy an Application**
1. **Via UI**:
   - Click **"New App"**.
   - Set Git repo: `https://github.com/argoproj/argocd-example-apps.git`.
   - Path: `guestbook`.
   - Destination: `https://kubernetes.default.svc` (current cluster).
   - Click **Create**.
![WhatsApp Image 2025-07-09 at 11 44 10_6746f746](https://github.com/user-attachments/assets/917aed49-2444-4796-91e4-e57b06df8826)

2. **Via CLI**:
   ```sh
   argocd app create guestbook \
     --repo https://github.com/argoproj/argocd-example-apps.git \
     --path guestbook \
     --dest-server https://kubernetes.default.svc \
     --dest-namespace default
   ```

3. **Sync the App** (if not auto-synced):
   ```sh
   argocd app sync guestbook
   ```

---

## 📦 Real-World Example: Deploying a Node.js App using Argo CD

### 1. Git Repo Structure:

```
node-app/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

### 2. Define Application in Argo CD:

![WhatsApp Image 2025-07-09 at 11 40 53_30b316f1](https://github.com/user-attachments/assets/56ccc152-423e-4678-8c73-5debe772cb4c)


```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: node-app
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  source:
    repoURL: https://github.com/deepak/node-app-gitops
    path: .
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Web-UI
![WhatsApp Image 2025-07-09 at 11 44 10_6746f746](https://github.com/user-attachments/assets/917aed49-2444-4796-91e4-e57b06df8826)

### Argo CLI

   ```sh
   argocd app create guestbook \
     --repo https://github.com/argoproj/argocd-example-apps.git \
     --path guestbook \
     --dest-server https://kubernetes.default.svc \
     --dest-namespace default
   ```


### 3. What Happens After You Push to Git?

* Argo CD fetches latest commit.
* It renders manifests and compares them with the cluster state.
* If there’s a drift, it syncs the new state.
* Your app is deployed/updated automatically.

---

## 🌟 Key Features of Argo CD

| Feature             | Benefit                                               |
| ------------------- | ----------------------------------------------------- |
| 🖥 Web UI + CLI     | Easy to view, manage, and sync Applications visually. |
| 🔁 Auto-sync        | Keeps cluster always in desired Git state.            |
| 🔐 RBAC & SSO       | Secure multi-user access.                             |
| 📊 Health & Status  | Shows app status, sync state, and health checks.      |
| 🔙 Rollbacks        | Restore previous Git commits easily.                  |
| 🧩 Helm & Kustomize | Supports popular templating tools.                    |

---

## 🔧 When to Use Argo CD?

* When you want **GitOps-style Kubernetes management**.
* For **multi-team, multi-environment** deployments.
* When **auditability and traceability** of deployments is needed.
* To **automate Kubernetes rollouts, syncs, and rollbacks**.


---

## ❓ Does Argo CD **pull** the changes from Git, or does Git **push** them?

### ✅ **Answer:**

**Argo CD *pulls* the changes** from Git.
There is **no push** from Git to Argo CD.

---

### 🔄 How it works (Pull-based model):

1. You make changes to your manifests in the Git repo (e.g., `deployment.yaml`).
2. You **`git push`** the changes to the repo.
3. Argo CD’s **Application Controller** and **Repo Server** periodically check (poll) the Git repo for changes (default: every 3 minutes).
4. If a change is detected:

   * It compares the Git state with the live cluster.
   * If **auto-sync** is enabled, Argo CD **applies the change automatically**.
   * If not, you can **manually sync** via CLI or UI.

---

### 🔁 Pull-Based Benefits (Argo CD’s Model):

| Benefit                 | Explanation                                                          |
| ----------------------- | -------------------------------------------------------------------- |
| ✅ **Security**          | No need to expose Kubernetes API externally. Git is passive.         |
| ✅ **Auditability**      | All changes are version-controlled.                                  |
| ✅ **Self-Healing**      | If something is changed manually in the cluster, Argo CD reverts it. |
| ✅ **Rollback-friendly** | Just revert a Git commit, Argo CD pulls and redeploys it.            |

---

### 🔁 Flow Summary:

```
[Git Repo] <-- (Argo CD Pulls) <-- [Repo Server & Controller] <---> [Kubernetes Cluster]
```

---

> 🔄 You **push** to Git → Argo CD **pulls** from Git → Changes **applied** to the cluster (if auto-sync is enabled)



