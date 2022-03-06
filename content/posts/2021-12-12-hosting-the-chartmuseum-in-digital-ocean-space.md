---
layout: post
title: Hosting the Chartmuseum in DigitalOcean Spaces
date: 2021-12-12 21:26:48.000000000 +00:00
 
background: /img/do-clean.jpg
---
![image info](/vlz-1.png)

Most DevOps engineers who use Chartmuseum to store/host their helm charts use S3 as their storage medium. Well, I wanted to try Digital Ocean spaces as its S3 compatible storage option.
  
  
Well, there's an obvious reason why to use S3 in the first place. Beautiful integration with AWS other services, cheap, easy to access, versioning, MFA delete protection etc.
  
  
However, if you're an early developer / DevOps engineer or in a small startup who doesn't wanna go through 1000 configurations in AWS just to create one single storage bucket in the cloud and again go through 1000 more security hurdles in case you want this bucket to be public, you should use DO Spaces. I'll list down why :
  
1. Super easy to set up.
1. Damn cheap. $5 for 250GB storage.
2. One-click public/private button.
1. Easy to integrate CDN if you have any.

 
  
That being said, Let's look at how we can utilize Spaces as an AWS S3 replacement to host our helm charts.

### Setup & Configuration 
#### 1. Create Spaces
   

- Log in to your console and click on Spaces and select *Create spaces* for $5
- The name has to be unique and make the *permission* (**File Listing**) private.
 
![](/d1.png)
 
- Once done, you should have the endpoint of the spaces created like :

 
![](/d2.png)
 

- Now, go to API settings and create Access keys for Spaces.
- Please note that the *secret key* will be displayed only once and hence keep it safe copied somewhere.
- With that said, you're ready to move to the next step.
 

  
#### 2. Install Chartmuseum
   
 
- Add helm repo :
```bash

helm repo add chartmuseum https://chartmuseum.github.io/charts
```  
 

- Update the repo :
```bash
helm repo update
```
  
 
- Here, we can't just run helm install since we need to tell chartmuseum to use Space as a holy place to store charts instead of local PVC ( default).
- Download the chart to your local system by running :

 
```bash
helm pull chartmuseum/chartmuseum —untar
```

Update the below fields in values.yaml file :

```yml
STORAGE: "amazon"
STORAGE_AMAZON_BUCKET: < your space name >
STORAGE_AMAZON_REGION: < REGION in which your space is created>
STORAGE_AMAZON_ENDPOINT: "https://REGION_NAME.digitaloceanspaces.com"
#enable API to interact with endpoint 
DISABLE_API: false
#secret section
BASIC_AUTH_USER: admin  
# password for basic http authentication
BASIC_AUTH_PASS: secret_password123
#Spaces access key
AWS_ACCESS_KEY_ID: "YOUR KEY"
#spacess secret key
AWS_SECRET_ACCESS_KEY: "YOUR SECRET KEY"
```

That's it, run the below command to install the chart :
```bash
helm install chartmuseum chartmuseum/ -f chartmuseum/values.yaml
```
 Next, add an entry in ingress to point to your Repository URL. For example :
  
```yml   
host: chartmuseum.tanmaybhat.tk
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: chartmuseum
                port:
                  number: 80
```    
  
Please note that you need to set *False* for Key *Disable_API*, else you cant send any data to your endpoint.
  
  
That's it, you can now add your repo to your helm by typing :
  
```bash  
helm repo add chartmuseum chartmuseum.example.com -u USERNAME -p PASSWORD
```  
  
If you want to push chart to this repo, you can do that by installing the helm push plugin:
  
```bash  
helm plugin install https://github.com/chartmuseum/helm-push.git --version v0.9.0
```  
  
Once installed, push the chart by running :
  
```bash  
helm push chart_directory chartmuseum
```  
  
Happy Helming !
  
---
   
  
### References
  
  
[Artifact Hub](https://artifacthub.io/packages/helm/chartmuseum/chartmuseum)
  
  
[Chart Museum](https://chartmuseum.com)
  
