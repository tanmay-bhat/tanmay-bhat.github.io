---
layout: post
title: Monitoring K8S resource changes with kubewatch
date: 2021-10-15 21:01:04.000000000 +00:00
 
background: /img/k8s-bg.png
---
 ## Definition 
 what's kubewatch ?   
> [kubewatch]( https://github.com/bitnami-labs/kubewatch">) is a Kubernetes watcher that currently publishes notification to Slack. Deploy it in your k8s cluster, and you will get event notifications in a slack channel.   
  
  
  Lets see how we can deploy it to our cluster.   
  
  
  **Pre-requisites :**   
 

- Kubernetes 1.12+ cluster
- Helm v3
- A slack app and a channel to integrate kubewatch
---
## Steps 
1. Add Bitnami repo to your helm :
  
 
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```
  
   2. Verify that kubewatch chart is available in the repo :   
  
```console
demo> helm search repo kubewatch
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/kubewatch       3.2.16          0.1.0           Kubewatch
```
  
  3. Customize the values like slack integration and enabling RBAC. If you directly do **helm install chart-name** you wont get any event notification as RBAC is set to **false** by default kin the helm chart.   
  
  4. Run the below command to get the values.yaml to local :
  ```bash
  helm show values bitnami/kubewatch > updated-values.yaml
  ```   
  
  
  Now edit the yaml file as per your requirement.  Here's what I've changed :   
  
  
  **Slack Integration :**   

```sh  
slack:
enabled: true
channel: "kubewatch"           #your slack channel name
## Create using: https://my.slack.com/services/new/bot and invite the bot to your channel using: /join @botname
##
token: "your slack bot token here"
```
  
  **Enable RBAC**:   

```yml
## @section RBAC parameters
## @param rbac.create Whether to create   use RBAC resources or not
##
rbac:
  create: true
```

  5. Now lets deploy using the below command :   

```sh
 helm install kubewatch bitnami/kubewatch -f ./updated-values.yaml
```
  
  6. Verify that kubewatch pod is running :   
  
```console
demo> kubectl get pod
  
NAME       READY STATUS RESTARTS AGE
kubewatch-c86656645-8znwk 1/1 Running 0 2m19s
```

  7. To test it out, lets create a nginx deployment with command :   
  
  
  ```bash
  kubectl create deploy nginx --image=nginx
  ```   
  
  
  8. Check your slack channel for notifications :   
  
![](/slack-alert.png)

  
  The indication is as follows :   
  
 

- Green : for resources created
- Yellow : for resources updated
- Red : for resources deleted

--- 
  You can customize the notification a lot, for example, which namespace to monitor to ( default value is all namespace) , which resource to monitor to like deployment, pod, PV, service etc.   
  
  
  You can just edit the configmap : **kubewatch-config** and change the resources to monitor.   
  
  
  Happy monitoring !!!   
  
  
  
  
  
