---
layout: post
title: How to migrate a Node-Group from Multi AZ to single AZ in AWS EKS
date: 2021-10-11 12:46:58.000000000 +00:00
 
---
### Question   
  After reading the above title you maybe thinking why though? moving the complete worker node fleet into single Availability Zone (AZ) is not a good solution when it comes to high availability of your Kubernetes cluster workload.   
  
  
  There's a reason at least why I had this requirement, **Cost optimization in AWS**.   
  
### Background  
  When you create a EKS cluster, it'll have 3 subnets each correcting to a single AZ i.e 3 AZ in a region. Now for staging / testing clusters the Inter Availability Zone data transfer fees we were getting was a hefty one, which was unnecessary as HA is not needed for the testing environment.   
  
  
  I couldn't find this anywhere else, so with an outage at staging cluster :D ( shhhhh!)      
  I found out that the solution is to create a new node group with AZ hard-coded while creating it and any node you spawn in that node group using ASG (Auto Scaling Group) will be in that single AZ only keeping your inter AZ data transfer cost to 0.    
  
### Solution  
```bash
eksctl create nodegroup --cluster=staging_cluster  \
           --region=ap-south-1 \
           --node-zones=ap-south-1a \
           --name=M5.2xlarge_NG \
           --node-type=m5.2xlarge
```
  
  In the above snippet, I'm creating NG in Mumbai Region in AZ ap-south-1a with m5.2xlarge instance.   
  
  
  If you want to go with **GUI** way, then :   
  
  
  - Go to your cluster in EKS and then click on **Add Node Group** :   
  
![](/add-ng.png)
 
  
  - Go with usual flow of giving it a name, taint if required, and IAM role.   
  
  
  - Select AMI, disk size, instance family etc.   
  
  
  - In **Networking section**, by default 3 subnets will be selected, untick 2 of them and keep 1 ( any desired AZ 'a/b/c'). If you're unsure about the name and AZ, you can verify that by going to VPC -> subnets.   
  
![](/ng-subnet.png)
 
  
  That's it, create the node group and all the instances will be spawned in that AZ only.   
 
  
 -   You can also do this in a hackish way by editing the ASG corresponding to the node group and removing 2 subnets from there.
   

 -  It works but your node group will become Unhealthy and AWS wont do anything to the node groups which is having health issue.
So better to create a new node group.   
  
 ### eksctl 
  There's one more way which I found out i.e to use ekctl command line and create from config file.   
  
  
 - You can read more about **eksctl** and configure it by referring [here](https://eksctl.io/introduction) .   
  
  
- Let's create a config file to create the node group : ```ap-south-la-NG.yaml``` :   

```yaml
  apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: Your_Cluster_Name
  region: ap-south-1

managedNodeGroups:
  - name: demo-nodegroup
    labels: { role: worker-nodes }
    instanceType: m5.xlarge
    desiredCapacity: 1
    volumeSize: 50
    availabilityZones: &#091; ap-south-1a ]
    minSize: 1
    maxSize: 2
    volumeType: gp3
    privateNetworking: true

```  
  
  Then you can apply using the below command :    
  
```bash
eksctl create nodegroup --config-file ap-south-la-NG.yaml
```
    
  
  **Finally**, once the new Node groups is created, you can scale down your existing node group to **0** so that AWS will drain the nodes gracefully and all your workloads will be moved to newly created node groups.   
  
