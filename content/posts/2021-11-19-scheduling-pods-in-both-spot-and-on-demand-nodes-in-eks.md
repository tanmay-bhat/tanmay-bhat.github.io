---
layout: post
title: Scheduling pods in both Spot and On-demand nodes in EKS
date: 2021-11-19 01:46:58.000000000 +00:00
 
---

### History
You may have faced this scenario where you wanna keep scaling up apps  nodes but also under-keeping costs at a limit.  Spot Instance is the way for that task. Now,  how do we do that? let's see.   
 

As you know there are mainly 2 types of instances in AWS, called **On-demand** and **Spot**. As the name suggests On-demand is priced highest because it's literally on demand from your side to AWS about node requirement.    


Spot instances are a bit different. Spot instance is the unused capacity in AWS cloud sitting idle and AWS gives that to you at an extremely low price for like **80-90%** cheaper than on-demand. The difference being whenever AWS needs the capacity to handle on-demand, it *takes back the spot instances with ~2 min notice* via notification to you.   

### Node groups
  Now that we've learned about what spot offers, it makes total sense to include that in your workload to save quite a lot of money. So let's learn about the node groups for on-demand and spot.   
  
#### Designing node groups :</h5>

- Since spot can go down anytime, you should always run your critical workloads in on-demand instances.
- All stateful- sets should run in on-demand instances.
- Have multiple nodegroups for spot so that you can maximize chance of getting spot instances.
- Use CA for scaling up / down.
 
 
### Real-world scenario </h3>
   
  
  Now, let's say you have critical apps running in on-demand NG and other cronjobs or monitoring stacks are in spot NG.  If you wanted to schedule pods with the below architecture I  got the answer.   
  
  
  **Required architecture**: If your app has 8 replicas, 4/5 of them should run in on-demand NG and 3/4 of them in spot NG such that even if the spot goes down, ondemand can handle the load until the new spot comes in and takes the load. In this way, you'll have 'Zero Downtime'.   
  
  
  Now for the above problem, there isn't a straightforward or clear-cut solution out of the box. But I'll explain the way I've implemented it.   
  
##### Solution
   
  
  Once both the NG are created, let's take a look at the label of a spot node.   
  
```console
kubectl describe no ip-100-45-51-226.ap-south-1.compute.internal | grep SPOT
eks.amazonaws.com/capacityType=SPOT
```
  
Any node created by spot will have the above label and any node with ondemand will have label : 
```
eks.amazonaws.com/capacityType=ON_DEMAND
```   
  
Once, they are done, you can create a sample Nginx deployment with the below configs:    

{{< gist tanmay-bhat 8f7777b4846bdb0b173ceb1c6a295c03 >}}  

All the other configs are pretty easy to understand and come under basic Kubernetes concepts. Since nodes created by spot and on-demand NG will have the above-mentioned labels, we can utilize that and request scheduler to try its best effort to schedule 40% of pods in this deployment to SPOT and 60% to ON_DEMAND.   
  
  
You can change the above weight as per your needs. Once the above YAML is deployed, let's take a look at the way pods are scheduled.    
  
 
![](/nginx-pod.png)
 
  
In my case, nodes with names **25** and **226** are Spot instances. If calculated correctly,  6 pods are running in on-demand and 4 pods are running in spot NG which is exactly as we expected.   
  
**NOTE**: This may not be always exactly the ratio you need since the scheduler gives pods to nodes on a best effort basis.  But it'll be almost a similar result.   
  
  
Happy Kuberneting !!!!   
  
