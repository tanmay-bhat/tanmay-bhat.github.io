---
layout: post
title: How to scale down Kubernetes cluster workloads during off hours
date: 2022-01-06
 
tags: ["Kubernetes"]
---
### You heard it right, everyone needs to rest once a while, even our little Kubernetes cluster.

Before we begin, here are the prerequisites : 

- Kubernetes cluster
- Cluster autoscaler
- Bit of patience

## Usecase :

- One of the most important aspect when it comes to running workload in cloud is to keep cost under control or tune it such that you can save extra.
- You maybe hosting workload in Kubernetes where you wont get traffic post business hours.
- Or in weekends, you just want to scale down as no traffic flows to your apps during that time.
- The cost to keep those worker nodes at off hours are pretty high if you calculate for a quarter or for a year.

## Solution :

Though there isn't any one click solution, Kubernetes finds a way or always Kubernetes Admin does !!

Strangely, there isn't any tool out of the BOX from AWS side, heck not even a blog on how can customers achieve that. Ahem, GCP aced in this [scenario](https://cloud.google.com/architecture/reducing-costs-by-scaling-down-gke-off-hours).

Behold .......................................

## Kube downscaler :

[Kube downscaler](https://codeberg.org/hjacobs/kube-downscaler) is a FOSS project by Henning Jacobs who’s the creator of famous project  kube-ops-view.

This project fits exactly to our requirement as it can scale down below resources in specified timeframe :

1. Statefulsets
2. Deployments (HPA too)
3. Cronjobs

### Installation :

- Clone the repository: 

```bash
    git clone https://codeberg.org/hjacobs/kube-downscaler
```
- Update the configmap file deploy/config.yaml to your Cluster TIMEZONE and desired uptime, here’s mine :

```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: kube-downscaler
    data:
    # timeframe in which your resources should be up
    DEFAULT_UPTIME: "Mon-Fri 09:30-06:30 Asia/Kolkata"
```

- Apply the manifest files :

```bash
    kubectl apply -f deploy/
```

## Working and configuration:

1. As soon as the downscaler pod runs, you can check the logs of it, it should look like *‘Scale down deployment/myapp to replicas 0 (dry-run)`*

2. As a safety-plug no scaling operations will happen when the pod starts as `--dry-run` argument is enabled. Remove that by patching the deployment to start the scaling activity: 

```bash
  kubectl patch deployment kube-downscaler --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "--interval=60"
]}]'
```

3. Once, dry-run argument is removed, all the resources ( deployment, Statefulset & cronjob) wil be scaled down to 0 ( default replica) if **current_time ≠ default_uptime** mentioned in above mentioned configmap.

4. Incase you need to exclude any app from being scaled down, you can annotate that depoyment/statefulset/cronjob with :

```bash
    kubectl annotate deploy myapp 'downscaler/exclude=true'
```

5. If you want to have minimum 1 replica after scale down activity, you can annotate the resource :

```bash
    kubectl annotate deploy myapp 'downscaler/downtime-replicas=1'
```

**Note** : No need to annotate uptime value in each deployment or statefulset since by default all pods will be scaled down.

Additional tunings like namespace based annotation etc are available at the readme [Here](https://codeberg.org/hjacobs/kube-downscaler).

### Achieving node scale down :

Once the pods are scaled down, assuming you have cluster autoscaler configured, it should automatically remove the nodes that are unused or empty from your nodegroup.

### Note: Cluster autoscaler is mandatory since at the end of the day that’s what removes worker nodes to save your bill.
---