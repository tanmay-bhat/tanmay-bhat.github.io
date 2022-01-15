---
layout: post
title: It happened again in production !!
date: 2021-10-13 21:07:07.000000000 +00:00
 
background: /egg-worry.jpg
---
  
  Previously I have written     https://wp.me/pcknFJ-2F">article  about how AWS pushed broken image to Docker hub and we got screwed as we were using *latest* as image tag.   
  
  
  Welp, this happened again in our CI/CD pipeline as we were using     https://github.com/chartmuseum/helm-push">push  plugin from helm and using that to push charts to     https://chartmuseum.com/">chartmuseum .   
  
  
<h4>How it happened?</h4>
   
  
  So we were using the below line to pull the helm push plugin :   
  
```
helm plugin install https://github.com/chartmuseum/helm-push.git
```
  
  And were pushing to Chartmuseum via command :    

```helm push app-name repo-name```
  
  It turns out that command is not valid and as per their latest (v0.10.0) changes to the plugin, its been renamed to **cm-push** and we gotta use like ```helm cm-push app-name repo-name```. Else we can use the same command with old version of plugin.    
  
  
  Hence our pipeline got screwed and I've fixed by pulling specific version from their repo by using -version argument. It goes like this :    
  
```helm plugin install https://github.com/chartmuseum/helm-push.git --version v0.9.0```

  
  The better solution to this is to replace the hard-coded version above to GitLab CI variable and update the version from there later.   
  
