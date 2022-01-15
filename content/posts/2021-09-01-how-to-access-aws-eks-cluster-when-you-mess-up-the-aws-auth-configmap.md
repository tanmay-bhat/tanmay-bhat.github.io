---
layout: post
title: How to access AWS EKS cluster when you mess up the aws-auth configmap
date: 2021-09-01 11:34:29.000000000 +00:00
background: /aws-mess.jpg
---
  
Hey people, this is not a complete solution article, but rather a cut story and a probable solution for the below problem statement when it comes to locked out issue in EKS cluster:   
  
> I wanted to add a user to my EKS, hence while adding the user to ```aws-auth configmap``` of my EKS cluster, I made some syntax mistakes and now neither I nor anyone can login to EKS cluster" whole cluster is gone, help me please !!!
  
*Straight forward solution which I found out :*   
  
  
  **Find out who created the EKS cluster ( owner) and ask them to edit the aws-auth configmap to correct your mistakes.**   
  
  
  The user who created the cluster is the root user for entity. Hence regardless of aws-auth configmap mess, he/she can login via kubectl anytime.   
  
  
  Read more [here](https://aws.amazon.com/premiumsupport/knowledge-center/eks-api-server-unauthorized-error/) on solution by AWS.   
  
  --- 
  I wrote this because I made this mistake in my company and spent hours searching for answer before finding this info.   
  
  
  Once I found out the creator, she corrected it in 1 min. :D    
  
  --- 
  
  <!-- wp:separator -->   
<hr class="wp-block-separator" />
<!-- /wp:separator -->
  
  **Long term solution :**   
  
  
  You might be saying ' Thats one solution to save my job, how do I make sure I dont do this mistake again ?'   
  
  
  Alright, so here's what you can follow from next time :   
  
  
- First get the configmap yaml file by typing :
  
 
  <!-- wp:paragraph {"align":"left"} -->   
```bash
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth-configmap.yml
```   
  
-  Once you get the yaml file, *edit* the file using your favorite text editor and update your changes.    
  
  
-  Now, update the configmap with your new updated file by typing :   
  
  
  ```bash
  kubectl apply -n kube-system -f aws-auth-configmap.yml
  ```  
  
  **Remember**, live editing is never a good option !!!   
  
  
  
