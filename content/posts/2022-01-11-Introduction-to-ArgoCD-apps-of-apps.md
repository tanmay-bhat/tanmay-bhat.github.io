---
layout: post
title: 'Introduction to ArgoCD : apps of apps'
date: 2022-01-11
---

### What's Apps of Apps  or Cluster bootstrapping ?


> [The App of Apps](https://argoproj.github.io/argo-cd/operator-manual/cluster-bootstrapping/) Pattern helps us define a root Application. So, rather than point to an application manifest fort every application creation, the Root App points to a folder  which contains the Application YAML definition for each child App. Each child app’s Application YAML then points to a directory containing the actual application manifests be it in manifest file, Helm or Kustomize.

ArgoCD will watch the root application and synchronize any applications that it generates.

## Requirement 

- You may have this question as to why is this even needed, where as you can manaully create the application via argo cli or via argo UI.
- But once the number of application you manage increases, automation needs to be done.
- For example, if you have a new app that needs to be created & deployed in your cluster via ArgoCD, apart from argo CLI and UI ( both are manual ) we really dont have any automation on creation of apps except custom scripting.

## Behold : Apps of apps

Actually, it took me a while to understand what exactly it is because the official doc is not well written.

Since the concept is pretty new and as far as I know, not much of argocd users are utilising this.

Now that we know *why, what* lets go over to *how.*

### Structure

- **Components** 
    - Root app : the single app you need to deploy to your cluster ( parent/root)
    - child app : your actual microservice applications ( child)
    - child app template : argocd Application kind template describing your application
    - application manifests : the actual  micro-service definitions.

**High level flow**  

Git repo → root app → application templates → actual applications

**Prerequisites** 

1. Kubernetes cluster
2. Helm v3
3. ArgoCD installed

### Demonstration 

For the purpose of demonstration, here are the components described above : 

1. **root app template** : [root.yaml](https://gitlab.com/Tanmay-Bhat/argocd/-/blob/main/apps-of-apps/manifests/root.yaml)
2. **application templates** : [manifests](https://gitlab.com/Tanmay-Bhat/argocd/-/tree/main/apps-of-apps/manifests)
3. **application definitions**  : [charts/monitoring](https://github.com/tanmay-bhat/ArgoCD/tree/main/Applicationset/charts/monitoring)

Let’s take a look at `root.yaml` :

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  project: default
  source:
    path: ./apps-of-apps/manifests/
    repoURL: https://gitlab.com/Tanmay-Bhat/argocd.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- `Name` : suggests the name of my root application i.e root-app
- `namespace` : all argocd based resources should reside in argocd namespace itself
- `Finaliser` : its the kubernetes CRD finalizer which help in protection from accidental [deletion](https://argo-cd-docs.readthedocs.io/en/latest/user-guide/app_deletion/#about-the-deletion-finalizer) of resources.
- `Destination` : the cluster onto which the resources should deploy. If you have multiple clusters , mention the required one here.
- `namespace`:  the actual namespace in which you want to deploy your app
- `path` : path inside your Git Repo where the application manifests are.
- `repoURL` : the GIT repo address.
- `targetRevision`: if its HEAD, argocd will always sync to master branch, if you need to be a different one, update that here. For example, staging branch for Dev environment etc.
- `prune` : this enables argocd to auto remove your application and its created resources when you delete the manifest file from the Application templates directory.
- `selfHeal` : this enables argocd to auto patch the configs same as latest master branch changes even though new config updates has been done via kubectl.

Let’s look at one of the actual application template inside manifests, `kube-state-metrics.yaml` : 

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-state-metrics
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  project: default
  source:
    path: ./Applicationset/charts/monitoring/kube-state-metrics
    repoURL: https://github.com/tanmay-bhat/ArgoCD.git
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- `name`: name of my application
- `namespace` : I’m deploying the app to kube-system namespace.
- `path`: the path inside GIT repo where hem charts are for the app
- `targetRevision`: I’m synching the changes to master branch.
- `repoURL` : repo where the helm charts / manifests of app is stored.

Once, we understand the above, we can proceed to try them out :

1. Install the root app by running :

      `kubectl apply -f root.yaml -n argocd`

1. Once installed, go to your ArgoCD UI and you should be able to see:

      ![argocd UI](/ap1.png)

1. Now that our root app is created and healthy, give it a minute and any application template you create inside manifests folder (in my case), will be automatically synced and created for you in the specified namespace.

      ![argocd apps](/ap2.png)

1. In the above picture, you can see root app is managing multiple apps and auto-syncs when you make changes to the helm charts (github repo in my case)


1. There’s a little arrow in each of the app, if you click on it, it will take you to the applciation page of it in argocd :
      ![argocd app view](/ap3.png)

1. If you need to add a new application, the flow goes like this, add the actual helm chart/manifest file to your GIT repo → add an argocd template to your templates directory → sit back and enjoy !!


1. I’ve added the argocd template and bam !! app is automatically created :
      ![argocd app creation](/ap4.png)

---

### Problems 

- Synching apps across multiple clusters.
  - Lets say you want to have node-exporter in multiple clusters, its not possible via apps-of-apps directly. For that you can create a application template and in `destination`  update your cluster name as registred in argocd.
  - This problem is solved by [Aplicationset](https://github.com/argoproj-labs/applicationset) from ArgoCD.
- Support for multiple values.yaml file for helm charts. 
  - Let’s say you want to apply node-selector to 2 clusters dev and prod each with their own customized values which is not possible at the moment.
