---
layout: post
title: ArgoCD Image Updater with Digital Ocean Container Registry
date: 2022-02-27
tags: ["ArgoCD", DigitalOcean]
---
## What's on Image Updater

> A tool to automatically update the container images of Kubernetes workloads
that are managed by [Argo CD](https://github.com/argoproj/argo-cd).
> 

### Capabilities :

- Argo CD Image Updater can check for new versions of the container images
that are deployed with your Kubernetes workloads and automatically update them
to their latest allowed version using Argo CD.
- It works by setting appropriate
application parameters for Argo CD applications, i.e. similar to
`argocd app set --helm-set image.tag=v1.0.1` - but in a fully automated
manner.

## Prerequisite :

1. Kubernetes Cluster
2. ArgoCD setup

## Steps

### Installation of ArgoCD Image Updater

- To install argocd image updater in your cluster ( same one as argocd), run the below command:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

- Once it's installed, let’s check the logs of the pod:

```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater

time="2022-01-16T06:21:00Z" level=info msg="Starting image update cycle, considering 0 annotated application(s) for update"
time="2022-01-16T06:21:00Z" level=info msg="Processing results: applications=0 images_considered=0 images_skipped=0 images_updated=0 errors=0"
```

Looks clean, let's move forward.

### Configure Remote Repository Secret

- One of the main features of image updater is to write back / update the new image tag to the remote git repo i.e. it will update the image tag, for example, v1.2.3 in the manifest file every time a new image is pushed to image registry.
- I’ll be using Gitlab in this case. Feel free to use Github also.

1. Obtain the Access token of your account from [Gitlab](https://gitlab.com/-/profile/personal_access_tokens) .
2. Once received, run the below command to create the secret :

```bash
kubectl --namespace argocd \
create secret generic gitlab-token \
-from-literal=username=GITLAB_USERNAME \
--from-literal=password=GITLAB_TOKEN
```

1. Verify the secret that has been created, by running the below command :

 

```bash
kubectl describe secret gitlab-token -n argocd
```


### Configure Container registry secret

- I’m using DigitalOcean Registry as my cloud provider for storing docker images.
- There are two ways to configure the secret that Kubernetes will use to authenticate to registry endpoint i.e `registry.digitalocean.com`.
1. **Automatic**: Follow the on-screen instructions on the DigitalOcean console to auto-integrate the Kubernetes secret into every namespace.

![digitalocean secrets auto integration](/do-secret-integration.png)

2. **Manual:** Download the Docker Credentials and follow steps on this [link](https://docs.digitalocean.com/products/container-registry/how-to/use-registry-docker-kubernetes/#create-secret-manually) to manually create a Kubernetes secret and then patch to service account of your namespace, **argocd** in this scenario.

![digitalocean secret download](/do-download-secrets.png)

---

### Configure Image updater to watch for Image updates.

- Image updater needs access to container registry to watch the Image updates.
- Out of the box below Registries are supported :
    - Docker Hub Registry
    - Google Container Registry (*gcr.io*)
    - RedHat Quay Registry (*quay.io*)
    - GitHub Docker Packages (*docker.pkg.github.com*)
    - GitHub Container Registry (*ghcr.io*)
    
- If you’re storing images in any other registry other than the above-mentioned, no need to worry, we can utilize the config map to add the registry configs.
- To Digital Ocean configs, Add the below to **configmap** : `argocd-image-updater-config` in `argocd` namespace:

```bash
registries:
- name: 'Digital Ocean'
  api_url: https://registry.digitalocean.com
  ping: no
  credentials: pullsecret:argocd/SECRET_NAME ( configured from above step )      
  defaultns: library
  prefix: registry.digitalocean.com
```

Here the credentials are in this order :  `credentials:pullsecret:namespace/secret-name`

There are other forms of secret configuration as well. Please refer to them [here](https://argocd-image-updater.readthedocs.io/en/stable/configuration/registries/#specifying-credentials-for-accessing-container-registries).

---

### Configure ArgoCD Application and Annotation

- Once the Gitlab secret is configured, let's move to the application configuration part.
- Create the Application manifest file for your application.
- Here’s a sample definition for kube-ops-view application :

{{< gist tanmay-bhat d4cdfe43462b3040ba4c86b553aeb765 >}}

Let’s understand the above file in detail :

1. We’re installing this resource in argocd namespace. ( this is mandatory for argocd resources)
2. `Annotations` :
    1. `image-list`: tells the image updater, which docker image to watch & update.
    2. `write-back-method`: is updating the image tag back to manifest files via git commit
    3. `git-branch`: we’ll write back to the branch: **main**.
3. `server`: we’re deploying this to the default cluster.
4. `namespace`: the actual namespace in which the application will be deployed.
5. `path` : the path of your helm chart inside the remote git repository.
6. `targetRevision`: the branch name to which argocd syncs apps.
7. Since we’re using helm, I’m updating to use values from `values.yaml` file.
8. `CreateNamespace`: if the mentioned namespace is not found, argocd will create it.

Once we’re good with the above configurations, to see the image updater in action...

1. Push a new image tag to your container registry :

```bash
> docker images

REPOSITORY                                           TAG       IMAGE ID       CREATED         SIZE
registry.digitalocean.com/tanmaybhat/kube-ops-view   latest    a645de6a07a3   21 months ago   253MB
registry.digitalocean.com/tanmaybhat/kube-ops-view   v1.2.3    a645de6a07a3   21 months ago   253MB

> docker tag registry.digitalocean.com/tanmaybhat/kube-ops-view:latest registry.digitalocean.com/tanmaybhat/kube-ops-view:v1.2.4

> docker images

REPOSITORY                                           TAG       IMAGE ID       CREATED         SIZE
registry.digitalocean.com/tanmaybhat/kube-ops-view   latest    a645de6a07a3   21 months ago   253MB
registry.digitalocean.com/tanmaybhat/kube-ops-view   v1.2.3    a645de6a07a3   21 months ago   253MB
registry.digitalocean.com/tanmaybhat/kube-ops-view   v1.2.4    a645de6a07a3   21 months ago   253MB

> docker push registry.digitalocean.com/tanmaybhat/kube-ops-view:v1.2.4

```

 Once the image is pushed, let's check the logs of *image updater pod* logs, it should look like this:

```bash
time="2022-01-18T13:09:11Z" level=info msg="git push origin main" dir=/tmp/git-kube-ops-view-demo405157222 execID=mvIL8
time="2022-01-18T13:09:13Z" level=info msg=Trace args="[git push origin main]" dir=/tmp/git-kube-ops-view-demo405157222 operation_name="exec git" time_ms=2058.666776
time="2022-01-18T13:09:13Z" level=info msg="Successfully updated the live application spec" application=kube-ops-view-demo
time="2022-01-18T13:09:14Z" level=info msg="Processing results: applications=1 images_considered=1 images_skipped=0 images_updated=1 errors=0
```

Here’s what image updater did : 

1. Detected a new image tag update in the registry
2. Cloned the repository
3. Made the image update in the manifest file 
4.  Pushed a commit back to the repository main branch

For further verification, let's go to the repo where the helm chart exists, we should be seeing a new commit : 

![argocd commit gitlab](/argocd-image-gitlab.png)

Let’s see what changed in the file : 

![argocd gitlab image tag update](/argocd-image-change-gitlab.png)

Now, all we have to do is wait for 3 min ( default sync period of argocd), and argocd notices that a new *image tag* has been updated and the same will be applied to the application.

![argocd app sync](/argocd-app-sync.png)
Let’s check the image tag as well : 

```bash
kubectl describe deploy kube-ops-view -n kube-ops-view | grep Image

Image: registry.digitalocean.com/tanmaybhat/kube-ops-view:v1.2.4
```
