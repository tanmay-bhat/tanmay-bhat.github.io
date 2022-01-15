---
layout: post
title: Pulling private image from Docker hub in GitLab CI
date: 2021-08-23 20:20:33.000000000 +00:00
 
background: /gitlab.png
---  
Hey people ! I'm back this time with a how-to on GitLab CI to make your life easy being DevOps Engineer. I thought of  writing this since I spent hours searching and fixing this :/    
  
  
Lets look at the problem or the requirement. It goes like this :   
  
  
> I have a GitLab CI file integrated into my project which builds a Dockerfile and pushes that image into ECR. But the dockerfile has a base image which is from a private Docker hub repository. how do I pull from that repo ?
  
### Gitlab CI  
Lets consider the below ```gitlab-ci.yml``` file :   
```yml  
image: "python:3.6"     
                    
stages:                                   
  - publish_image                         

build and push docker image:        
  stage: publish_image
  only:                                   
    variables:
        - $CI_COMMIT_TAG =~ /^v&#091;0-9]+\.&#091;0-9]+\.&#091;0-9]+-&#091;0-9]+\.&#091;0-9]+\.&#091;0-9]+$/ 
        - $CI_COMMIT_TAG =~ /^v&#091;0-9]+\.&#091;0-9]+\.&#091;0-9]+$/     
  variables:
    DOCKER_HOST: tcp://docker:2375
  image: 
    name: amazon/aws-cli
    entrypoint: ""
  services:
    - docker:dind 
  before_script:
    - echo "$CI_COMMIT_TAG"
    - amazon-linux-extras install docker
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$DOCKER_TAG .
    - aws ecr get-login-password | docker login --username AWS --password-stdin $DOCKER_REGISTRY 
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_TAG
    - docker push $DOCKER_REGISTRY/$APP_NAME:$DOCKER_TAG
```
  
  
Here's how the above CI file works :   

- Uses base image ```python``` on which the stages will run.
- has a single stage which will build and push images to ECR
- *only* section tells gitlab to run the stage only if the git tag is done and it matched the regex mentioned. 
- in *before_script* section, we're displaying the commit tag and installing docker in aws-cli image since that image doesn't come preinstalled with docker.
- finally we're doing docker login to with our dockerhub account before building Dockerfile.
- Later we build the Dockerfile and then push it to ECR
  
 
  
### Configure login to Docker hub in GitLab CI
  

- To configure the Dockerhub credentials, go to your GitLab *project -&gt; settings -&gt; CI/CD*
- In Variables section, add the below Key and their value :
  
 
```sh
Key : CI_REGISTRY ||  Value : docker.io
Key : CI_REGISTRY_USER ||  Value : your_dockerhub_username
Key : CI_REGISTRY_PASSWORD || Value : your_dockerhub_password
```

![](/ci-var.png)
  
Now, to setup AWS credentials, configure the below values :   
  
```sh
Key : AWS_ACCESS_KEY_ID || Value : your_aws_accesskey
Key : AWS_SECRET_ACCESS_KEY || Value : your_aws_secretkey
```
![](/ci-aws.png)
 
  
 That's it, voila !! Now GitLab runner should get your docker credentials from variables and pull the image seamlessly.   
  
