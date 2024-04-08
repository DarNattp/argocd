---
runme:
  id: 01HTZB7Q8JS1R6X8FWHCFJSETB
  version: v3
---

# Self Managed Argo CD - App of Everything

**Table of Contents**

- [Self Managed Argo CD - App of Everything](#self-managed-argo-cd---app-of-everything)
- [First install ssh key for argocd access git origin repo](#first-install-ssh-key-for-argocd-access-git-origin-repo)
- [Introduction](#introduction)
- [Clone Repository](#clone-repository)
- [Create Local Kubernetes Cluster](#create-local-kubernetes-cluster)
- [Git Repository Hierarchy](#git-repository-hierarchy)
- [Create App Of Everything Pattern](#create-app-of-everything-pattern)
- [Intall Argo CD Using Helm](#intall-argo-cd-using-helm)
- [Demo With Sample Application](#demo-with-sample-application)
- [Cleanup](#cleanup)
- [SSH Key](#ssh-key)

# First install ssh key for argocd access git origin repo

```sh {"id":"01HT7533J2JVN7JWD9SPP0K27N"}
server:
  ## ArgoCD config
  ## reference https://github.com/argoproj/argo-cd/blob/master/docs/operator-manual/argocd-cm.yaml
  configEnabled: true
  config:
    repositories: |
      - type: git
        url: git@gitlab.creativebox.dev:creativebox/gitops/argocd.git
        sshPrivateKeySecret:
          name: argocd-repo-ssh-key
          key: sshPrivateKey
      - type: git
        url: git@gitlab.creativebox.dev:creativebox/gitops/gitops-dev.git
        sshPrivateKeySecret:
          name: argocd-repo-ssh-key
          key: sshPrivateKey     
```

# Introduction

This project aims to install a self-managed Argo CD using the App of App pattern. Full instructions and explanation can be found in the Medium article [Self Managed Argo CD — App Of Everything](https://medium.com/devopsturkiye/self-managed-argo-cd-app-of-everything-a226eb100cf0).

# Clone Repository

Clone kurtburak/argocd repository to your local device..

```sh {"id":"01HT74ZB1CX0SQ7E53S09N6JSN"}
git clone https://github.com/kurtburak/argocd.git
```

# Create Local Kubernetes Cluster

Intall kind.

```sh {"id":"01HT74ZB1CX0SQ7E53S3QV9G0S"}
brew install kind
```

Create local Kubernetes Cluster using kind.

```sh {"id":"01HT74ZB1CX0SQ7E53S6AMAQG9"}
kind create cluster — name my-cluster
```

Check cluster is running and healthy

```sh {"id":"01HT74ZB1CX0SQ7E53S8WHG9CG"}
kubectl cluster-info — context kind-my-cluster

Kubernetes control plane is running at https://127.0.0.1:50589
KubeDNS is running at https://127.0.0.1:50589/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use ‘kubectl cluster-info dump’.
```

# Git Repository Hierarchy

Folder structure below is used in this project. You are free to change it.

```ini {"id":"01HT74ZB1CX0SQ7E53SBNX39MM"}
argocd/
├── argocd-appprojects      # stores ArgoCD App Project's yaml files
├── argocd-apps             # stores ArgoCD Application's yaml files
├── argocd-install          # stores Argo CD installation files
│ ├── argo-cd               # argo/argo-cd helm chart
│ └── values-override.yaml  # custom values.yaml for argo-cd chart
```

# Create App Of Everything Pattern

Open *argocd-install/values-override.yaml* with your favorite editor and modify related values.

```sh {"id":"01HT74ZB1CX0SQ7E53SD7ZG8WA"}
vi argocd-install/values-override.yaml
```

Or update it with your values.

```yaml {"id":"01HT74ZB1CX0SQ7E53SG0M9Y7T"}
cat << EOF > argocd-install/values-override.yaml
server:
  configEnabled: true
  config:
    repositories: |
      - type: git
        url: https://github.com/kurtburak/argocd.git
      - name: argo-helm
        type: helm
        url: https://argoproj.github.io/argo-helm
  additionalApplications: 
    - name: argocd
      namespace: argocd
      destination:
        namespace: argocd
        server: https://kubernetes.default.svc
      project: argocd
      source:
        helm:
          version: v3
          valueFiles:
          - values.yaml
          - ../values-override.yaml
        path: argocd-install/argo-cd
        repoURL: https://github.com/kurtburak/argocd.git
        targetRevision: HEAD
      syncPolicy:
        syncOptions:
        - CreateNamespace=true
    - name: argocd-apps
      namespace: argocd
      destination:
        namespace: argocd
        server: https://kubernetes.default.svc
      project: argocd
      source:
        path: argocd-apps
        repoURL: https://github.com/kurtburak/argocd.git
        targetRevision: HEAD
        directory:
          recurse: true
          jsonnet: {}
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
    - name: argocd-appprojects
      namespace: argocd
      destination:
        namespace: argocd
        server: https://kubernetes.default.svc
      project: argocd
      source:
        path: argocd-appprojects
        repoURL: https://github.com/kurtburak/argocd.git
        targetRevision: HEAD
        directory:
          recurse: true
          jsonnet: {}
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
  additionalProjects: 
  - name: argocd
    namespace: argocd
    additionalLabels: {}
    additionalAnnotations: {}
    description: Argocd Project
    sourceRepos:
    - '*'
    destinations:
    - namespace: argocd
      server: https://kubernetes.default.svc
    clusterResourceWhitelist:
    - group: '*'
      kind: '*'
    orphanedResources:
      warn: false
EOF
```

# Intall Argo CD Using Helm

Go to argocd directory.

```sh {"id":"01HT74ZB1CX0SQ7E53SJY4YEXH"}
cd argocd/argocd-install/
```

Intall Argo CD to *argocd* namespace using argo-cd helm chart overriding default values with *values-override.yaml* file. If argocd namespace does not exist, use *--create-namespace* parameter to create it.

```sh {"id":"01HT74ZB1CX0SQ7E53SPRNFEC8"}
helm install argocd ./argo-cd \
    --namespace=argocd \
    --create-namespace \
    -f values-override.yaml
```

Wait until all pods are running.

```sh {"id":"01HT74ZB1CX0SQ7E53SQHKYC60"}
kubectl -n argocd get pods

NAME                                            READY   STATUS    RESTARTS
argocd-application-controller-bcc4f7584-vsbc7   1/1     Running   0       
argocd-dex-server-77f6fc6cfb-v844k              1/1     Running   0       
argocd-redis-7966999975-68hm7                   1/1     Running   0       
argocd-repo-server-6b76b7ff6b-2fgqr             1/1     Running   0       
argocd-server-848dbc6cb4-r48qp                  1/1     Running   0
```

Get initial admin password.

```sh {"id":"01HT74ZB1CX0SQ7E53STB82FD7"}
kubectl -n argocd get secrets argocd-initial-admin-secret \
    -o jsonpath='{.data.password}' | base64 -d
```

Forward argocd-server service port 80 to localhost:8080 using kubectl.

```sh {"id":"01HT74ZB1CX0SQ7E53SWNAYFHQ"}
kubectl -n argocd port-forward service/argocd-server 8080:80
```

Browse http://localhost:8080 and login with initial admin password.

# Demo With Sample Application

Create an application project definition file called *sample-project*.

```yaml {"id":"01HT74ZB1CX0SQ7E53T07S06QZ"}
cat << EOF > argocd-appprojects/sample-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: sample-project
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - namespace: sample-app
    server: https://kubernetes.default.svc
  orphanedResources:
    warn: false
  sourceRepos:
  - '*'
EOF
```

Push changes to your repository.

```sh {"id":"01HT74ZB1CX0SQ7E53T1X9QMAP"}
git add argocd-appprojects/sample-project.yaml
git commit -m "Create sample-project"
git push
```

Create a saple applicaiton definition yaml file called *sample-app* under argocd-apps.

```yaml {"id":"01HT74ZB1CX0SQ7E53T5RFT4B3"}
cat << EOF >> argocd-apps/sample-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  destination:
    namespace: sample-app
    server: https://kubernetes.default.svc
  project: sample-project
  source:
    path: sample-app/
    repoURL: https://github.com/kurtburak/argocd.git
    targetRevision: HEAD
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    automated:
      selfHeal: true
      prune: true
EOF
```

Push changes to your repository.

```sh {"id":"01HT74ZB1CX0SQ7E53T71TRAFH"}
git add argocd-apps/sample-app.yaml
git commit -m "Create application"
git push
```

# Cleanup

Remove application and applicaiton project.

```sh {"id":"01HT74ZB1CX0SQ7E53T7XXP8FT"}
rm -f argocd-apps/sample-app.yaml
rm -f argocd-appprojects/sample-project.yaml
git rm argocd-apps/sample-app.yaml
git rm argocd-appprojects/sample-project.yaml
git commit -m "Remove app and project."
git push
```

# SSH Key

```sh {"id":"01HT74ZB1CX0SQ7E53TBD02TTS"}
kubectl create secret generic argocd-repo-ssh-key --from-file=sshPrivateKey=/path/to/your/argocd/private/key -n argocd
```