---
layout: post
title: How to SSH into docker in PWD (Play With Docker)
date: 2021-01-22 10:28:28.000000000 +00:00
background: /docker.png

---
 
  
  Hey all ! For those of you who don't  know what PWD is below is short explanation :   
  
  
  So, PWD stands for Play With Docker. You can deploy   learn docker at free with time limit of each instance up-to 10 Hrs!   
  
  
  For more info, go to : [Docker Labs](https://labs.play-with-docker.com/)    
  
---
  There are two ways you can access the docker instance.   
  
  - Use the web based console.
 - SSH into that instance.
  
 
  
  I always love to do ssh as it gives me more freedom.   
  
  ---

  If you go straight away and do ssh from your terminal, you will get :   
  
```console
lab-suse:~/.ssh # ssh ip-x-x-x-@direct.labs.play-with-docker.com
ip-x-x-x-@direct.labs.play-with-docker.com: Permission denied (publickey).
```
  
  Why are we getting this ? because there is no fresh key generated in your host.   
  
  
  Lets create a fresh key, run the below command and use default values :   
  
```sh
lab-suse:~/# ssh-keygen
```
  
  After you complete the above command, try ssh again, it should work:    
  
```console
lab-suse:~/.ssh# ssh ip-x-x-x-@direct.labs.play-with-docker.com
 The authenticity of host 'direct.labs.play-with-docker.com (40.76.55.146)' can't be established.
 RSA key fingerprint is SHA256:UyqFRi42lglohSOPKn6Hh9M83Y5Ic9IQn1PTHYqOjEA.
 Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
 Warning: Permanently added 'direct.labs.play-with-docker.com,40.76.55.146' (RSA) to the list of known hosts.
 Connecting to Ip-x-x-x:8022

###########################
# WARNING!!!!                          
# This is a sandbox environment. Using personal credentials 
# is HIGHLY! discouraged. Any consequences of doing so are completely
# the user's responsibilites.
# The PWD team                                                                                                     

node1 root@192.168.0.28
```

  Note : if you are using any ssh applications, save that key you generated to a file and load that file in authentication section.    
  