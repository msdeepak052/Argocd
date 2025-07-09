# Private Git repositories
To deploy apps from **private Git repositories** in Argo CD, you need to **register them securely** with authentication — either via **SSH** or **HTTPS with username/password/token**.

Let's cover both **declarative** and **Web UI** methods step-by-step with examples.

---

## 🔧 Argo CD Supports Git Authentication via:

| Protocol | Auth Method                        | Notes                                    |
| -------- | ---------------------------------- | ---------------------------------------- |
| `HTTPS`  | Username + Password / PAT (token)  | Used for GitHub, GitLab, Bitbucket, etc. |
| `SSH`    | SSH Private Key (usually `id_rsa`) | More secure; avoids storing passwords    |

---

## ✅ 1. 📁 Declarative Method (Recommended for GitOps)

You can register private Git repos declaratively using a Kubernetes **Secret** of type `argoproj.io/repo-creds`.

---

### 📘 YAML: HTTPS with Token (GitHub, GitLab, etc.)

```yaml
# https-git-repo.yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-private-https
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://github.com/deepak/private-repo
  username: your-username
  password: your-personal-access-token
```

> 💡 Replace `your-username` and `your-personal-access-token` with actual credentials.

---

### 📘 YAML: SSH with Private Key

```yaml
# ssh-git-repo.yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-private-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: git@github.com:deepak/private-repo.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    <your-private-key>
    -----END OPENSSH PRIVATE KEY-----
```

> ✅ Make sure your public key is added to the Git repo's deploy keys.

---

### 🔁 Apply the YAML:

```bash
kubectl apply -f https-git-repo.yaml
kubectl apply -f ssh-git-repo.yaml
```

Once applied, Argo CD will be able to pull from these private repositories securely.

---

## ✅ 2. 🌐 Register Private Git Repo via Argo CD Web UI

### 🔄 Steps:

1. Open Argo CD Web UI (e.g., `https://argocd.yourdomain.com`)
2. Go to **Settings** → **Repositories**
3. Click **"Connect Repo"** (top-right)
4. Fill in the details:

---

### 📥 For HTTPS Repos:

* **Type:** Git
* **Repository URL:** `https://github.com/deepak/private-repo`
* **Username:** `your-username`
* **Password / Token:** Your GitHub/GitLab PAT

Then click **"Connect"**

---

### 🔐 For SSH Repos:

* **Type:** Git
* **Repository URL:** `git@github.com:deepak/private-repo.git`
* **SSH Private Key:** Paste your full private key (`id_rsa`)
* **Username:** *(leave blank)*
* **Enable Strict Host Key Check:** *(optional)*

Click **"Connect"**

---

## ✅ 3. Use the Repo in Argo CD Application YAML

Once the repo is registered (declaratively or via UI), you can reference it in your app:

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/private-repo
    path: k8s/app
    targetRevision: main
```

---

## 🔐 Important Notes

| Item                    | Recommendation                                                    |
| ----------------------- | ----------------------------------------------------------------- |
| 🔑 Use tokens           | Prefer PAT over password for HTTPS repos                          |
| 🔐 Use SSH for security | SSH keys are safer and revocable                                  |
| 🔁 Declarative GitOps   | Always store secret YAMLs in a secure repo (e.g., Sealed Secrets) |
| 🔁 autoSync supported?  | Yes, Argo CD pulls automatically even from private repos          |

---

## ✅ Summary

| Method      | How?                                | Use When?                      |
| ----------- | ----------------------------------- | ------------------------------ |
| Declarative | Create Secret with repo credentials | GitOps-style, secure, scalable |
| UI-based    | Settings → Repositories → Connect   | Quick testing and setup        |

---
# **Argo CD supports private Helm repositories**

Let’s cover both **declarative** and **Web UI** methods for registering a **private Helm repo**, with **full examples**.

---

## 🔧 Supported Helm Repo Types in Argo CD

| Repo Type      | URL Format                           | Auth Options             |
| -------------- | ------------------------------------ | ------------------------ |
| HTTP(S)        | `https://charts.mycompany.com/helm/` | Basic Auth, Token, TLS   |
| OCI (Helm v3+) | `oci://ghcr.io/myorg/helm-charts`    | Docker login/token-based |
| S3/GCS         | `s3://my-bucket/charts`              | IAM / static credentials |

---

## ✅ 1. 📁 Declarative Registration of Private Helm Repo

### 🔐 Example 1: **HTTP(S) + Username/Password**

```yaml
# helm-repo-https.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-private-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  name: private-helm
  type: helm
  url: https://charts.mycompany.com/helm/
  username: deepak
  password: mySecurePassword
```

---

### 🔐 Example 2: **OCI Helm Repo with Bearer Token**

```yaml
# helm-repo-oci.yaml
apiVersion: v1
kind: Secret
metadata:
  name: oci-helm-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  name: my-oci-charts
  type: helm
  url: oci://ghcr.io/deepak/charts
  password: ghp_xxx_yourgithubtoken
```

> 💡 For OCI-based Helm repos like GitHub Container Registry (GHCR), use a GitHub token as password.

---

### 🧪 Apply the Secret

```bash
kubectl apply -f helm-repo-https.yaml
kubectl apply -f helm-repo-oci.yaml
```

---

## ✅ 2. 🌐 Register via Web UI

1. Go to Argo CD UI → **Settings** → **Repositories**
2. Click **"Connect Repo"**
3. Fill the form:

---

### For HTTP(S) Helm Repo

* **Type:** Helm
* **Repository URL:** `https://charts.mycompany.com/helm/`
* **Username / Password**: Provide as needed
* Click **Connect**

---

### For OCI Helm Repo

* **Type:** Helm
* **Repository URL:** `oci://ghcr.io/deepak/charts`
* **Password:** GitHub Token (no username needed)

---

## 🧠 Using Registered Helm Repo in Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: private-helm-app
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  source:
    repoURL: https://charts.mycompany.com/helm/
    chart: mychart
    targetRevision: 1.2.3
    helm:
      releaseName: deepak-release
      valueFiles:
        - values-dev.yaml
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

> 💡 For OCI, use the repo URL as `oci://ghcr.io/deepak/charts`.

---

## 🔐 TLS & CA Configuration (Optional)

If the Helm repo uses **custom TLS** (e.g., internal CA), you can add:

```yaml
  insecure: "false"
  enableLfs: "false"
  tlsClientCertData: <base64 PEM cert>
  tlsClientCertKey: <base64 PEM key>
  insecureSkipVerify: "true" # optional
```

---

## ✅ Summary

| Method      | Secret Type                | Use When                             |
| ----------- | -------------------------- | ------------------------------------ |
| HTTPS       | `username + password`      | Basic-auth secured Helm repos        |
| OCI         | `GitHub token` or login    | GitHub/GitLab/Harbor OCI Helm charts |
| Declarative | YAML via Secret            | GitOps, automation                   |
| UI-Based    | Argo CD → Settings → Repos | Manual setup, quick test             |

---
## Credential templates

**Credential templates** in Argo CD help you **avoid repeating the same credentials** across multiple Git or Helm repositories. They’re especially useful when:

* Multiple private repos share the **same authentication**
* You want **reusability** and **separation of secrets**

Let’s go deep into:

* What they are
* How to use them declaratively
* How to configure them via Web UI
* And full examples for Git and Helm

---

## 🔐 What are Credential Templates in Argo CD?

**Credential templates** are Kubernetes `Secret` objects with a special label:

```yaml
argocd.argoproj.io/secret-type: repo-creds
```

They apply **globally** and are **automatically used** by any repo matching the template's `url` prefix.

> 🧠 Example: A `repo-creds` secret for `https://github.com/deepak/` will apply to all repos under that org, like:
>
> * `https://github.com/deepak/project-1`
> * `https://github.com/deepak/helm-charts`

---

## ✅ Declarative: Credential Template for HTTPS Git Repo

```yaml
# git-creds-template.yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  url: https://github.com/deepak/
  username: deepak
  password: <your-personal-access-token>
```

> ✅ This will apply to **all Git repos** under `https://github.com/deepak/` automatically — no need to define separate secrets for each repo.

---

## ✅ Declarative: Credential Template for SSH Git Repo

```yaml
# ssh-creds-template.yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-ssh-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  url: git@github.com:deepak/
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

> 🧠 Any repo under `git@github.com:deepak/` will now **automatically use** this SSH key.

---

## ✅ Declarative: Credential Template for Private Helm Repo

```yaml
# helm-creds-template.yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-helm-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  url: https://charts.mycompany.com/helm/
  username: deepak
  password: myStrongPassword
  type: helm
```

> 🧠 Automatically used for **any Helm repo** under that URL prefix.

---

## ✅ Web UI: Add Credential Template

### Steps:

1. Go to Argo CD **Web UI**
2. Click **Settings** → **Repositories**
3. Click **"Connect Repo using Credential Template"** (⚙️ icon at top)
4. Fill in:

* **Type:** Git or Helm
* **Repository URL Prefix:** e.g., `https://github.com/deepak/`
* **Auth:** Username + Token or SSH Key
* Click **"Save"**

🧠 Argo CD will now apply this credential **to any repo whose URL starts with the prefix**.

---

## ✅ Application YAML Using Credential Template

If your Git or Helm repo matches the `url` prefix of the credential template, just reference the repo — **no need to include credentials in Application YAML**:

```yaml
spec:
  source:
    repoURL: https://github.com/deepak/project-1
    path: k8s/
    targetRevision: main
```

> ✅ Argo CD will auto-use the matching `repo-creds` template.

---

## ✅ Summary Table

| Feature          | Value                                               |
| ---------------- | --------------------------------------------------- |
| Secret type      | `argocd.argoproj.io/secret-type: repo-creds`        |
| Match by         | `url` prefix (e.g., `https://github.com/org/`)      |
| Use for          | Git or Helm repos                                   |
| Benefits         | Reusable credentials for multiple repos             |
| Declarative YAML | Yes                                                 |
| Web UI support   | Yes (under Settings → Repos → Credential Templates) |

---

## 🧪 Tips

* Name the `Secret` meaningfully (`github-creds`, `helm-org-creds`)
* Don't confuse with per-repo secrets (`repository` label vs `repo-creds`)
* Use one `repo-creds` secret per URL prefix for clarity
* Works great with sealed secrets for GitOps

---



