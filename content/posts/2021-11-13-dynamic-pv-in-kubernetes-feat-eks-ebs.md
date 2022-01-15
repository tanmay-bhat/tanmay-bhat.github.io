---
layout: post
title: Dynamic PV in Kubernetes feat. EKS (EBS)
date: 2021-11-13 20:13:38.000000000 +00:00
 
background: /aws-skeleton.png
---

### Definition
Let's Understand what's volume resizing mean for Persistent Volumes kin KUbernetes.   


Its the ability to dynamically increase the PV size as required ( EBS volume behind the scene ).   

#### Problem statement
**Up until v1.16 EKS, you can just increase any ( PV ) EBS volume size just by running command like** : ```kubectl edit pv your_PV``` **and just change the size, it used to work since you have storage class  of** ```kubernetes.io/aws-ebs```.   


Now, You can't resize your PV just by changing the size in the manifest file (> EKS v1.17)


what should you do as a Kubernetes admin if you wanna resize your PV with above mentioned version ?   

---

#### Prerequisites:
- AWS EKS cluster
- Eleveated IAM permissions
- Understanding of PV in Kubernetes
### Solution
- Simple, Kubernetes team has a new tool called **ebs-csi controller**. What does it do?    


*The Amazon Elastic Block Store (Amazon EBS) Container Storage Interface (CSI) driver provides a CSI interface that allows Amazon Elastic Kubernetes Service (Amazon EKS) clusters to manage the lifecycle of Amazon EBS volumes for persistent volumes.*   


- You can install the ebs-csi driver by referring to AWS [document](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) .   

--- 
- Once you installation is done, you should see the pods similar to :    

![](/ebs-csi.png)
 
  
- And the ebs-csi-controller pod logs should look like :    
  
![](/ebs-log.png)
 
  
- Looks good, now, for a test, lets edit a PV and increase its size. In my example, I'll just increase the alert-manager PV to 3GB, Initial size was 2GB. 
  
![](/pv-update.png)
 
  
- If you are thinking how did I beautify Kubernetes editing ? all thanks to [Lens IDE](https://k8slens.dev).   
  
  
- Lets verify the PV size now :   
  
![](/pv-describe.png)
 
  
- Here comes the real test to see if the actual EBS volume is resized or not. For that let's copy the volume id and search that volume size in AWS console or via AWS cli to verify the disk size.   
  
  
- To get the volume id of a PV, run the below command :   
  
  
  ```bash
  kubectl describe pv PV_NAME  | grep Volume
  ```   
  
  
- Now if you prefer AWS CLI, you can use the following command, else can be verified in AWS console :    
  
![](/ebs-describe.png)
  
