## 🔍 What is `recurse` in Argo CD?

The `recurse: true` option is used in Argo CD to tell it to:

> **Recursively traverse all subdirectories** under the specified `path` and include all valid Kubernetes manifests (YAML/JSON files) in them.

---

### ✅ Use Case

When your Git repo has a structure like:

```
my-app/
├── base/
│   └── deployment.yaml
├── overlays/
│   └── dev/
│       └── config.yaml
├── common/
│   └── secrets.yaml
```

By default, Argo CD **only looks at files directly inside `path:`**
But with `recurse: true`, it **goes into subfolders** and applies everything.

---

## 🧪 Example without recurse

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/multi-dir-app
    path: my-app
    targetRevision: main
```

👉 Only files like `my-app/deployment.yaml` will be considered.
Files in `my-app/base/`, `my-app/overlays/`, etc., are ignored.

---

## ✅ Example **with** `recurse: true`

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/multi-dir-app
    path: my-app
    targetRevision: main
    directory:
      recurse: true
```

👉 Argo CD will now include all manifests in:

* `my-app/base/`
* `my-app/overlays/dev/`
* `my-app/common/`
* and more...

---

## 💡 When to Use `recurse: true`

| Situation                                                           | Use it?                                |
| ------------------------------------------------------------------- | -------------------------------------- |
| You manually organize YAMLs in subdirectories                       | ✅ Yes                                  |
| You are using multiple team folders                                 | ✅ Yes                                  |
| You want one Argo CD Application to manage multiple apps/components | ✅ Yes                                  |
| You're using Kustomize or Helm                                      | ❌ No *(use their own pathing instead)* |

> 🔥 Avoid using it in Helm/Kustomize apps where `path` should be specific to a chart or overlay.

---

## ✅ Real Example

### Git Repo:

```
k8s-manifests/
├── frontend/
│   └── deployment.yaml
├── backend/
│   └── service.yaml
├── monitoring/
│   └── prometheus.yaml
```

### Argo CD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: full-stack-app
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  source:
    repoURL: https://github.com/deepak/k8s-manifests
    path: .
    targetRevision: main
    directory:
      recurse: true
  project: default
```

> ✅ Argo CD will apply all resources under `frontend/`, `backend/`, and `monitoring/`.

---

## ✅ Summary

| Field                     | Purpose                                                     |
| ------------------------- | ----------------------------------------------------------- |
| `directory.recurse: true` | Recursively loads all YAML/JSON manifests in subdirectories |
| Default                   | `false` — Only files in root `path` are used                |
| When to use               | When managing large multi-dir manifest repos                |

---

# **Projects in Argo CD**

---

## 🧱 What is a **Project** in Argo CD?

In Argo CD, a **Project** is a **logical grouping of Applications**.

> Think of it as a **namespace or boundary** for applications — to **control access, destinations, source repos, and resource limits**.

**Projects** in Argo CD are a way to **group and manage applications** with shared configurations, permissions, and restrictions. They help:
- **Organize applications** (e.g., by team, environment, or business unit).
- **Enforce security policies** (RBAC, namespace restrictions).
- **Control sync and deployment behavior** (e.g., allow only certain Git repos or clusters).

---

## ✅ Why Are Projects Needed?

| Purpose                       | Benefit                                                            |
| ----------------------------- | ------------------------------------------------------------------ |
| ✅ **Multi-team isolation**    | Separate access for dev, QA, and prod teams                        |
| ✅ **Access control (RBAC)**   | Restrict who can view/deploy/manage apps                           |
| ✅ **Source repo restriction** | Allow only specific Git repos                                      |
| ✅ **Destination restriction** | Limit to specific clusters/namespaces                              |
| ✅ **Mandatory Field**         | All Argo CD Applications must belong to a project (`project: xyz`) |

---

## 🔧 Default Behavior

* Argo CD ships with a **default project** called `default`.
* If you don’t specify a project in an Application, it uses `default`.

---

## 🧪 Example Use Case

### Scenario:

You have 3 environments:

* `dev`
* `staging`
* `prod`

You want:

* Different teams to manage each environment.
* Apps in `dev` to only deploy to `dev` namespace.
* Dev team to only deploy from `https://github.com/deepak/dev-repo`.

Use Argo CD Projects to enforce all that.

---

## 📘 How to Create a Project in Argo CD (via **Web UI**)

### 🔄 Step-by-step:

1. 🖥 Go to Argo CD Web UI → login
2. In left menu, click **“Projects”**
3. Click **“+ New Project”**

### 📝 Fill the details:

#### Project Name:

```
dev-team
```

#### Description:

```
Project for Dev Team Applications
```

#### Source Repositories (Allowed Git repos):

```
https://github.com/deepak/dev-repo
```

#### Destinations:

```
Server: https://kubernetes.default.svc
Namespace: dev
```

#### Cluster Resource Whitelist (optional):

Limit access to only certain kinds of K8s objects (e.g., only `Deployments`, `Services`)

---

## ✅ After Project is Created...

Now create an Application and assign it to the project:

```yaml
spec:
  project: dev-team
```

Argo CD will **enforce** the restrictions:

* Block deployments if the Git repo is not allowed.
* Block if target cluster/namespace is not in destination list.

---
or

## **Creating a Project in Argo CD (Web UI)**
### **Step 1: Log in to Argo CD UI**
- Access Argo CD (`https://localhost:8080` if using `kubectl port-forward`).
- Log in as `admin` (or a user with admin privileges).

### **Step 2: Navigate to Projects**
1. Click **"Settings"** (⚙️) in the left sidebar.
2. Select **"Projects"**.
3. Click **"New Project"**.

### **Step 3: Define Project Basics**
- **Name**: `my-team-prod` (must be unique).
- **Description**: "Production applications for My Team".
- Click **"Create"**.

### **Step 4: Configure Project Settings**
#### **1. Source Repositories (Allowed Git Repos)**
- Restrict where apps can pull manifests from.
- Go to **"Sources"** tab → **"Add Source"**.
  - Enter `https://github.com/my-org/prod-manifests.git` (only allow this repo).

#### **2. Destination Clusters & Namespaces**
- Restrict where apps can deploy.
- Go to **"Destinations"** tab → **"Add Destination"**.
  - **Cluster**: `https://kubernetes.default.svc` (or external cluster).
  - **Namespace**: `prod` (only allow deployments here).

#### **3. Roles & Permissions (RBAC)**
- Define who can do what in this project.
- Go to **"Roles"** tab → **"Add Role"**.
  - **Role Name**: `prod-deployer`.
  - **Policies**:
    - `p, proj:my-team-prod:prod-deployer, applications, sync, my-team-prod/*, allow`
    - `p, proj:my-team-prod:prod-deployer, applications, get, my-team-prod/*, allow`
  - Click **"Save"**.

#### **4. Sync Windows (Optional)**
- Define when syncs are allowed (e.g., only during business hours).

---

## **Assigning Users/Groups to Project Roles**
### **Step 1: Go to "Projects" → Select `my-team-prod` → "Roles" Tab**
### **Step 2: Click "Add Group" or "Add User"**
- Example: Assign GitHub team `my-org/dev-team` to `prod-deployer` role.
- Or assign a user: `user:john@example.com`.

---

## **Example: Deploying an App Inside a Project**
### **Step 1: Create an Application in the Project**
1. Go to **"Applications"** → **"New App"**.
2. **Project**: Select `my-team-prod`.
3. **Source**: `https://github.com/my-org/prod-manifests.git` (must match allowed sources).
4. **Destination**: Cluster `https://kubernetes.default.svc`, Namespace `prod` (must match allowed destinations).
5. Click **"Create"**.

### **Step 2: Verify Restrictions**
- If a user tries to deploy to `dev` namespace, Argo CD **blocks it** (project policy violation).
- If a user without `prod-deployer` role tries to sync, they **get "Forbidden"**.

---


## 🔐 What are **Project Roles** in Argo CD?

Each **Project** can define **custom roles**, which are used in **RBAC policies** to restrict access **per project** and **per action**.

---

## ✅ Why Project Roles?

| Feature               | Example                                                       |
| --------------------- | ------------------------------------------------------------- |
| 👤 Per-project access | Allow Devs to deploy apps only within `dev-team` project      |
| 🔑 Token-based auth   | Generate JWT tokens for automated CI/CD tools (e.g., Jenkins) |
| 🚫 Restrict actions   | Allow read-only access, or only sync, not delete              |

---

## 📘 Create a Project Role via Web UI

1. Go to **Projects**
2. Select your project (e.g., `dev-team`)
3. Click **"Edit YAML"**
4. Add roles under `spec.roles`

---

### 🔐 Example: Role with read-only access

```yaml
spec:
  roles:
  - name: dev-readonly
    description: Read-only access for dev team
    policies:
      - p, proj:dev-team:dev-readonly, applications, get, dev-team/*, allow
    groups:
      - dev-team
```

> 🔑 You can also generate a **JWT token** for this role to be used in CI/CD tools (click “Generate Token” in UI).

---

## ✅ Summary of Project Role Fields

| Field         | Purpose                                      |
| ------------- | -------------------------------------------- |
| `name`        | Role name (used in RBAC policies and tokens) |
| `description` | Description for human reference              |
| `policies`    | RBAC-style permissions                       |
| `groups`      | Map to SSO/LDAP groups                       |

---

## 🛠 Example Use Case with Roles

You create 2 roles in `prod-team` project:

* `prod-read`: Only view apps
* `prod-admin`: Full control (create, sync, delete)

CI pipeline uses `prod-admin` token, developers use `prod-read`.

---

## ✅ Summary Table

| Concept           | Description                                                           |
| ----------------- | --------------------------------------------------------------------- |
| **Project**       | Logical grouping of apps for RBAC, repo, and cluster restrictions     |
| **Mandatory?**    | Yes — must be specified in every Argo CD Application (`project: xyz`) |
| **Project Roles** | Custom RBAC permissions within a project                              |
| **JWT Tokens**    | Tokens tied to roles, used in automation tools like CI/CD             |

---


# 🧱 Argo CD Project: Declarative Definition

Argo CD Projects are Kubernetes **Custom Resources** of kind:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
```

---

## ✅ Scenarios We'll Cover:

1. Basic project with default settings
2. Restricting to specific **Git repos** and **Kubernetes namespaces**
3. Defining allowed **Kubernetes resource kinds**
4. Enabling **orphaned resource monitoring**
5. Defining **roles and JWT token access**
6. Adding **sync windows** (time-based restrictions)

---

## 📁 Project YAML: Basic

```yaml
# project-basic.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: dev-team
  namespace: argocd
spec:
  description: Dev team project for all dev apps
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: '*'
```

> ✅ Allows deploying from **any repo** to **any cluster/namespace** (like default).

---

## 🔐 Restrict Repos and Clusters

```yaml
# project-restricted.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: prod-team
  namespace: argocd
spec:
  description: Project for production-only apps
  sourceRepos:
    - https://github.com/deepak/prod-apps
  destinations:
    - namespace: prod
      server: https://kubernetes.default.svc
```

> ✅ Only allows deploying apps from a specific Git repo to `prod` namespace in current cluster.

---

## 🧩 Restrict Resource Kinds

```yaml
# project-kinds.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: secure-project
  namespace: argocd
spec:
  description: Only allow limited resource kinds
  sourceRepos:
    - '*'
  destinations:
    - namespace: secure
      server: '*'
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: apps
      kind: Deployment
  namespaceResourceBlacklist:
    - group: ''
      kind: Secret
```

> ✅ Allows only Deployments and Namespace creation, but **blocks creating Secrets**.

---

## 🚨 Enable Orphaned Resource Monitoring

```yaml
# project-orphaned.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-audit
  namespace: argocd
spec:
  orphanedResources:
    warn: true
```

> ✅ Argo CD will **warn if it finds orphaned resources** (not in Git but still in cluster).

---

## 🔐 Add Project Roles + JWT Token Access

```yaml
# project-with-roles.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ci-access
  namespace: argocd
spec:
  sourceRepos:
    - '*'
  destinations:
    - namespace: ci
      server: '*'
  roles:
    - name: read-only
      description: View-only access
      policies:
        - p, proj:ci-access:read-only, applications, get, ci-access/*, allow
      groups:
        - ci-team
    - name: deployer
      description: Full access for CI/CD
      policies:
        - p, proj:ci-access:deployer, applications, sync, ci-access/*, allow
        - p, proj:ci-access:deployer, applications, get, ci-access/*, allow
```

> ✅ You can later generate JWT tokens using Argo CD CLI or UI for these roles.

---

## 🕒 Define Sync Windows

```yaml
# project-sync-window.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: ops-maintenance
  namespace: argocd
spec:
  sourceRepos:
    - '*'
  destinations:
    - namespace: ops
      server: '*'
  syncWindows:
    - kind: allow
      schedule: "Mon-Fri 09:00-18:00"
      duration: 9h
      applications:
        - "*"
```

> ✅ Allows syncing only during working hours (Mon-Fri, 9 AM to 6 PM IST).

---

## ✅ Apply All Projects

Assuming you're in the `argocd` namespace:

```bash
kubectl apply -f project-basic.yaml
kubectl apply -f project-restricted.yaml
kubectl apply -f project-kinds.yaml
kubectl apply -f project-orphaned.yaml
kubectl apply -f project-with-roles.yaml
kubectl apply -f project-sync-window.yaml
```

---

## ✅ Summary

| Feature                       | Field(s) Used                                            |
| ----------------------------- | -------------------------------------------------------- |
| Git repo restriction          | `sourceRepos`                                            |
| Namespace/cluster restriction | `destinations`                                           |
| Resource kind restriction     | `clusterResourceWhitelist`, `namespaceResourceBlacklist` |
| Orphaned resource monitoring  | `orphanedResources`                                      |
| Project-specific RBAC roles   | `roles`                                                  |
| Time-based sync restrictions  | `syncWindows`                                            |

---



