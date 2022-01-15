---
layout: post
title: Story of keeping CI pipeline from getting screwed when AWS pushes broken docker image to Docker hub
date: 2021-09-19 20:15:04.000000000 +00:00
 
background: /aws-fire.jpg
--- 
## History 
  If you're using aws-cli docker image in your CI pipeline then this story could be useful   amusing for you.   
  
  
  On Thursday, I started receiving alerts that our CI pipeline is failing.   
  
  
  I started checking the failed job error and it pointed out to docker is unable to install killing the pipeline.   
  
```console
Installing docker
Installation failed. Check that you have permissions to install.
Cleaning up file based variables
ERROR: Job failed: command terminated with exit code 1
```
 
  
  After scratching the head for sometime, I found that the latest ```aws-cli ```image from amazon Docker hub repository is causing the issue as I haven't changed anything else in the CI file in few weeks.   
  
  
  So I went to Docker hub and I saw on that day there was a new version pushed which was **2.2.39** tagged as *latest*. Since in our CI file, we didn't mention specific image version to pull so it always assumes the tag to pull is *latest*.   
  
  
  As a temporary fix, I changed the image version to older one which was **2.2.38** and it worked fine.   
  
  --- 
  If you ask me for a better a better solution, it would be always good to use a **specific version** in production since you know it will work for sure instead of using *latest* tag which could change every single day.   
  
  
  Else push that image to your private container repositories like ECR and pull from there.   
  
  
  I'm pretty sure AWS broke few thousand CI pipelines over the world whoever used latest as the image tag :D   
     
  
  To give an idea about how to install docker inside ```aws-cli``` image, you can just run the below command which should install docker from AWS hosted repo for a faster install :   
  
  
  ```bash
  amazon-linux-extras install docker
  ```   
  
   DevOps story ends here. I'll update more stories like this in future :)    
  
  
  
  
  
