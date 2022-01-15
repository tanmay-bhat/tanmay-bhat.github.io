---
layout: post
title: Using Single Load Balancer across multiple namespaces in Kubernetes 
date: 2021-12-19
 
---
One Loadbalancer to rule them all ? you heard it true, Its achievable !

## For AWS LoadBalancer Controller
Until couple weeks ago, we were creating a loadbalancer for each namespace ( by default from AWS), which was a waste of resources and money.
Hence we thought how can we use a **single loadbalancer ** across all the namespaces.

Here's an example of before migration, how ingress looked like for default namespace:

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-default
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: instance
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: deployment-A
                port:
                  number: 80
```


Lets take a look at ArgoCD  ingress :

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-argocd
  namespace: argocd
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: instance

spec:
  rules:
    - host: argocd.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: argo-server
                port:
                  number: 80
```

Now, the problem with the above configuration is, by default AWS loadbalancer contoller will create ALB for each namespace.
The cost of creating ALB for each namespace is $17.99 and data transfter charges are $0.02 per GB. If you got a lot of namespaces, the cost will be alot over the year. This is not Loadbalancer are supoosed to be used as ALB can handle alot of load in a single unit.

After hunting for a while, we found that the solution lies in using a ALB annotation called **group.name** on the Ingress object.

### The flow 

1. Assuming you got 5 different ingress with 5 ALB in backend in different namespaces, choose one ingress and add the annotation :
    ```
    alb.ingress.kubernetes.io/group.name: <ANYTHING_MEANINGFULL>
    ```
2. Now, apply the same group name annotation to all the ingress objects in all namespaces.
3. AWS LoadBalancer controller will use the first ingress Loadbalcner for all ingress objects and all other 4 ALB in this case  will be removed.

Below is an example for 2 different namespaces with single ALB : 

```yml
#default namespace ingress

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-default
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/group.name: staging-lb
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: instance
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: deployment-A
                port:
                  number: 80
```

```yml
#argocd namespace ingerss
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-argocd
  namespace: argocd
  annotations:
    alb.ingress.kubernetes.io/group.name: staging-lb
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: instance

spec:
  rules:
    - host: argocd.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: argo-server
                port:
                  number: 80
```                 

4. Notice how both ingress objects are using the same annotion with same Group name.

Whenever next time you need to create a new ingress in differnt namesapce, you can just add the same  *group.name * annotation and it works flawlessly.

---
## For NGINX Ingress Controller 
 If you are in any other cloud, lets say DigitalOcean, its cloud provider controller doesn't have any custom LB controller. But dont get sad there buddy, NGINX to the rescue.

- You can leverage NGINX ingress controller to create & manage LB for you.

- For installing NGINX ingress controller, you can refer thier [documentation](https://kubernetes.github.io/ingress-nginx/deploy/) which has all the steps to install it.

- Lets say you got 2 different namespaces with 2 different ingress objects same as above one for default and one for ArgoCD.

- The solution is to add ```ingressClassName: nginx``` in the ingress object **Spec ** section in each ingress file.

So the final ingress file looks like this for default namespace:

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-default
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: chartmuseum.example.com
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
Argocd Ingress namespace ingress looks like this:

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-argocd
  namespace: argocd
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80

```
Happy Loadbalancing :)