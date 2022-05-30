---
layout: post
title: How to enable Google SSO in Kibana using OAuth2 Proxy [Kubernetes]
date: 2022-05-30
tags: ["Kubernetes", "Kibana"]
---

**OAuth2 Proxy** is a reverse proxy that provides authentication using Providers such as Google, GitHub, and others to validate accounts by email, domain or group.

In this article, we’ll setup and configure OAuth2 Proxy to setup Google SSO for Kibana in Kubernetes.

### Prerequisite:

1. Kibana in Kubernetes
2. Nginx Ingress

## Setup Google Credentials

1. In Google Cloud Console, select [OAuth consent screen](https://console.cloud.google.com/apis/credentials/consent)
2. Select the User Type as : “**Internal”**
3. Complete the app registration form with your ****Authorized domain****, then click **Save.**
4. In the left Nav pane, choose [Credentials](https://console.cloud.google.com/apis/credentials)
5. In the center pane, choose **"Credentials"** tab.
    - Open the **"New credentials"** drop down
    - Choose **"OAuth client ID"**
    - Choose **"Web application"**
    - Application name can be anything, for simplicity, let’s say Kibana.
    - Authorized JavaScript origins is your Kibana endpoint ex: `https://kibana.example.com`
    - Authorized redirect URIs is the location of **oauth2/callback** ex: `https://kibana.example.com.com/oauth2/callback`
    - Choose **"Create"**
6. Download the Credentials JSON file.

## Setup Oauth2 Proxy

1. Generate a random cookie secret with the below command and copy it to clipboard : 

```bash
docker run -ti --rm python:3-alpine python -c 'import secrets,base64; print(base64.b64encode(base64.b64encode(secrets.token_bytes(16))));'
```

2. Add the Helm repository : 

```bash
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
```

3. Install the helm chart with specified parameters as below : 

```bash
helm upgrade --install oauth2-proxy oauth2-proxy/oauth2-proxy \
--set config.clientID="GOOGLE_CLIENT_ID_HERE" \
--set config.clientSecret="GOOGLE_CLIENT_SECRET_HERE" \
--set config.cookieSecret="GENERATED_COOKIE_SECRET_HERE" \
--set extraArgs.provider="google"
```

- Use the Google Client_ID and SECRET from the downloaded JSON file.
4. Create and apply the Ingress for Oauth2 Proxy using the below YAML manifest : 

```yml
cat <<EOF | kubectl create -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: oauth2-proxy
spec:
  ingressClassName: nginx
  rules:
  - host: https://kibana.example.com
    http:
      paths:
      - path: /oauth2
        pathType: Prefix
        backend:
          service:
            name: oauth2-proxy
            port:
              number: 4180
EOF
```

## Setup Ingress annotation for Kibana

1. This article assumes you already have a Kibana setup running in Kubernetes with an Ingress.
2. Append the two nginx annotations mentioned below to existing Kibana Ingress. Once , the Ingress should look similar to : 

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  annotations:
    nginx.ingress.kubernetes.io/auth-signin: https://$host/oauth2/start?rd=$escaped_request_uri
    nginx.ingress.kubernetes.io/auth-url: https://$host/oauth2/auth
spec:
  ingressClassName: nginx
  rules:
  - host: kibana.example.com
    http:
      paths:
      - backend:
          service:
            name: kibana
            port:
              number: 5601
        path: /
        pathType: ImplementationSpecific
```

- Now if you hit the endpoint [kibana.example.com](http://kibana.example.com). it should ask you to login via Google.

![kibana-sso](/kibana-google.png)

- Since we have mentioned the OAuth type as **Internal** and also specified Authorised Domain, any google login except your specified Domain will be 401.

---

References : 

[https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/)

[https://oauth2-proxy.github.io/oauth2-proxy/](https://oauth2-proxy.github.io/oauth2-proxy/)