# Stack in Argo CD (App of Apps)
---

## 🧩 **1️⃣ The Core Idea — A “Stack” = A Group of Related Applications**

In **Argo CD**, a *stack* usually refers to a **logical grouping of applications** that together form a complete environment or service.

Think of it as a **collection of Argo CD Applications** managed together — for example:

```
Banking Stack (Dev)
├── frontend-app
├── backend-app
├── database-app
└── monitoring-stack
```

Each of those (frontend, backend, etc.) is a separate `Application` in Argo CD, but together they make up your **Dev stack**.

---

## 🧠 **2️⃣ Why “Stack” Matters in Argo CD**

Argo CD manages Kubernetes manifests from Git.
But large environments are made up of multiple components — databases, microservices, monitoring tools, etc.
Instead of deploying each manually, you can manage them as **a stack** for:

* Environment isolation (dev, stage, prod)
* Easier CI/CD control
* Rollback and promotion consistency
* DRY (don’t repeat yourself) GitOps structure

---
Let’s go step-by-step so you fully understand **how**, **why**, and **where** it’s used (with your GitHub setup included).

---

## 🧩 **1️⃣ What Is the App-of-Apps Pattern?**

**Definition:**
It’s a GitOps design pattern where **one Argo CD Application (the parent)** manages and deploys **multiple other Applications (the children)** — each of which represents a microservice, component, or subsystem.

In other words:

> You deploy *one parent YAML* → it automatically deploys several apps → each app manages its own Kubernetes manifests.

---

## 🧠 **2️⃣ Why We Use App-of-Apps**

| Reason                 | Description                                                                             |
| ---------------------- | --------------------------------------------------------------------------------------- |
| 🧹 Centralized control | Manage multiple related applications (frontend, backend, DB) from one parent.           |
| 🔁 GitOps consistency  | Each child app can have its own repo or directory, but all changes still flow from Git. |
| 🔄 Automated syncing   | Parent auto-creates, updates, and prunes child apps.                                    |
| 🧩 Modular design      | You can reuse components (e.g., monitoring stack) across environments.                  |

---

## 📦 **3️⃣ Folder Structure on GitHub**

Here’s the exact structure to follow in your GitHub repo (e.g. `msdeepak052/argocd-banking-app`):

```
argocd-banking-app/
├── environments/
│   ├── dev/
│   │   ├── frontend-app.yaml
│   │   ├── backend-app.yaml
│   │   └── database-app.yaml
│   └── prod/
│       ├── frontend-app.yaml
│       ├── backend-app.yaml
│       └── database-app.yaml
├── manifests/
│   ├── frontend/
│   │   └── deployment.yaml
│   ├── backend/
│   │   └── deployment.yaml
│   └── database/
│       └── deployment.yaml
└── parent-app/
    └── dev-stack.yaml
```

---

## 🪣 **4️⃣ Step-by-Step Workflow**

### 🧭 Step 1 — Create GitHub Repo

Create a new repo (public or private):

```
https://github.com/msdeepak052/argocd-banking-app
```

Upload the folder structure above.

Each `deployment.yaml` (frontend, backend, database) is a simple Kubernetes deployment + service file.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: dev
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

---

### 🧭 Step 2 — Create the *child* Argo CD Application manifests

For example, `environments/dev/frontend-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-app
spec:
  project: default
  source:
    repoURL: https://github.com/msdeepak052/argocd-banking-app.git
    path: manifests/frontend
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

Do the same for `backend-app.yaml` and `database-app.yaml`.

---

### 🧭 Step 3 — Create the *parent* Application (App-of-Apps)

File: `parent-app/dev-stack.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-stack
spec:
  project: default
  source:
    repoURL: https://github.com/msdeepak052/argocd-banking-app.git
    path: environments/dev
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

This parent points to the folder (`environments/dev`) that holds the **child application definitions**.

---

### 🧭 Step 4 — Add the Parent App to Argo CD

Apply it with `kubectl`:

```bash
kubectl apply -f https://raw.githubusercontent.com/msdeepak052/argocd-banking-app/main/parent-app/dev-stack.yaml
```

Argo CD will:

1. Create the `dev-stack` parent Application.
2. Sync automatically.
3. Deploy the 3 child apps.
4. Each child app then syncs its own manifests (frontend, backend, DB).

---

### 🧭 Step 5 — Verify in Argo CD UI

In the Argo CD dashboard, you’ll see:

```
dev-stack
 ├── frontend-app
 ├── backend-app
 └── database-app
```

Each one can be synced individually or via the parent.

---

### 🧭 Step 6 — (Optional) Extend to Multiple Environments

Just duplicate the folder:

```
environments/prod/
   ├── frontend-app.yaml
   ├── backend-app.yaml
   └── database-app.yaml
```

Then create a new parent `prod-stack.yaml`.

---

## 🚀 **5️⃣ Visual Summary**

```
          ┌────────────────────────┐
          │      dev-stack (Parent)│
          │        (App of Apps)   │
          └────────────┬───────────┘
                       │
     ┌─────────────────┼──────────────────┐
     │                 │                  │
frontend-app      backend-app        database-app
(child App)        (child App)         (child App)
```

Each child app manages its own resources independently, but the parent manages them all together.

---

## 🌍 **6️⃣ Real-World Analogy**

Think of the **parent app** as a *project manager*, and each **child app** as a *team lead* for frontend, backend, and DB.
When you promote a release — the project manager (parent) triggers all team leads (children) automatically.

---



# 🚀 Complete GitHub + Argo CD Stack Project (Step-by-Step)

Let’s call your repository:

📁 **`argocd-banking-stack`**

You’ll create it in GitHub — just a **public repo** so Argo CD can read it.
URL example:

```
https://github.com/msdeepak052/argocd-banking-stack.git
```

---

## 🏗️ Step 1 — Git Repository Folder Structure

You’ll have this **exact structure locally** before pushing to GitHub:

```
argocd-banking-stack/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── database/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── monitoring/
│       ├── prometheus.yaml
│       └── grafana.yaml
└── environments/
    └── dev/
        ├── frontend-app.yaml
        ├── backend-app.yaml
        ├── database-app.yaml
        ├── monitoring-app.yaml
        └── dev-stack.yaml
```

Now let’s create each file, **step-by-step** 👇

---

## 📁 Step 2 — Create Files Under `/apps` Folder

Each folder under `/apps` represents one microservice.

---

### 🟦 **apps/frontend/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:latest
        ports:
        - containerPort: 80
```

### 🟦 **apps/frontend/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

---

### 🟨 **apps/backend/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: python:3.9
        command: ["python", "-m", "http.server", "5000"]
        ports:
        - containerPort: 5000
```

### 🟨 **apps/backend/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 5000
    targetPort: 5000
```

---

### 🟩 **apps/database/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mssql-db
  labels:
    app: mssql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mssql
  template:
    metadata:
      labels:
        app: mssql
    spec:
      containers:
      - name: mssql
        image: mcr.microsoft.com/mssql/server:2019-latest
        env:
          - name: ACCEPT_EULA
            value: "Y"
          - name: SA_PASSWORD
            value: "Pass@word123"
        ports:
        - containerPort: 1433
```

### 🟩 **apps/database/service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mssql-svc
spec:
  type: ClusterIP
  selector:
    app: mssql
  ports:
  - port: 1433
    targetPort: 1433
```

---

### 🟧 **apps/monitoring/prometheus.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus
        ports:
        - containerPort: 9090
```

### 🟧 **apps/monitoring/grafana.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana
        ports:
        - containerPort: 3000
```

---

## 📁 Step 3 — Create `/environments/dev/` Folder

This folder defines **Argo CD Applications**.

Each `.yaml` here is a *child ArgoCD Application* pointing to one app.

---

### 🧩 **environments/dev/frontend-app.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-app
spec:
  project: default
  source:
    repoURL: https://github.com/msdeepak052/argocd-banking-stack.git
    path: apps/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: banking-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 🧩 **environments/dev/backend-app.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-app
spec:
  project: default
  source:
    repoURL: https://github.com/msdeepak052/argocd-banking-stack.git
    path: apps/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: banking-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 🧩 **environments/dev/database-app.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: database-app
spec:
  project: default
  source:
    repoURL: https://github.com/msdeepak052/argocd-banking-stack.git
    path: apps/database
  destination:
    server: https://kubernetes.default.svc
    namespace: banking-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 🧩 **environments/dev/monitoring-app.yaml**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring-app
spec:
  project: default
  source:
    repoURL: https://github.com/msdeepak052/argocd-banking-stack.git
    path: apps/monitoring
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

### 🧩 **environments/dev/dev-stack.yaml** (Parent App)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dev-stack
spec:
  project: default
  source:
    repoURL: https://github.com/msdeepak052/argocd-banking-stack.git
    path: environments/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🧠 Step 4 — Push Everything to GitHub

Run these commands:

```bash
cd argocd-banking-stack
git init
git add .
git commit -m "Initial commit - ArgoCD Banking Stack"
git branch -M main
git remote add origin https://github.com/msdeepak052/argocd-banking-stack.git
git push -u origin main
```

Your repo now contains everything Argo CD needs. ✅

---

## 🌍 Step 5 — Connect Argo CD to the Repo

In Argo CD UI →
click **New App** → fill in details:

| Field            | Value                                                            |
| ---------------- | ---------------------------------------------------------------- |
| Application Name | `dev-stack`                                                      |
| Project          | default                                                          |
| Sync Policy      | Automatic                                                        |
| Repository URL   | `https://github.com/msdeepak052/argocd-banking-stack.git`        |
| Path             | `environments/dev`                                               |
| Cluster          | [https://kubernetes.default.svc](https://kubernetes.default.svc) |
| Namespace        | argocd                                                           |

Then click **Create**.

---

## 🚀 Step 6 — Observe in Argo CD UI

You’ll see one parent app → `dev-stack`
Expanding it will show 4 child apps:

```
dev-stack
├── frontend-app
├── backend-app
├── database-app
└── monitoring-app
```

Once they sync successfully, you’ll have 4 running workloads in your cluster:

```bash
kubectl get pods -n banking-dev
kubectl get pods -n monitoring
```

---

## ✅ Step 7 — Verify Deployment

Expected output:

```
NAME                          READY   STATUS    RESTARTS   AGE
frontend-xxxx                 1/1     Running   0          1m
backend-xxxx                  1/1     Running   0          1m
mssql-db-xxxx                 1/1     Running   0          1m
prometheus-xxxx               1/1     Running   0          1m
grafana-xxxx                  1/1     Running   0          1m
```

---

## 💡 Step 8 — Understanding the Flow

| Component                  | Description                                                    |
| -------------------------- | -------------------------------------------------------------- |
| `/apps/`                   | Contains actual Kubernetes manifests                           |
| `/environments/dev/*.yaml` | Defines Argo CD Applications for each component                |
| `dev-stack.yaml`           | Parent application — the “stack”                               |
| Argo CD                    | Watches the Git repo → syncs manifests → deploys automatically |

---

## 🧱 Next Step (Optional, Advanced)

We can extend this project to support:

* **Multiple environments** (dev, stage, prod) using **ApplicationSet**
* **Parameterization** (e.g., image tags per env)
* **Helm or Kustomize** for templating

---
