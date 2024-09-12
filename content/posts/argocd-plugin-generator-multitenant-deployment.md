---
layout: post
title: Leveraging ArgoCD ApplicationSets with Custom Generators: Streamlining Multi-Tenant Deployments
date: 2024-09-12
tags: ["ArgoCD", "CD"]
---

In this article, you'll learn how to leverage ArgoCD ApplicationSets with custom generators to streamline multi-tenant deployments. By the end, you'll understand how to create a custom generator plugin, set up an ApplicationSet, and use features like selective deployments and Go templating.

### Prerequisite :

- ArgoCD installed in a Kubernetes cluster.


### Writing custom generator plugin

Generators are responsible for generating *parameters*, which are then rendered into the `template:` fields of the ApplicationSet resource. See the [Introduction](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/) for an example of how generators work with templates, to create Argo CD Applications.

The plugin can be written in any language as long as it responds to HTTP requests, and in our example, we will be writing it in Python.

Let’s assume we have a DB that keeps track of tenant’s state.

Now, we will write the generator plugin service which returns of tenant and thier status in list of dictionary as result : 

```python
import os
import json
from typing import List, Dict
from fastapi import FastAPI, HTTPException, Depends, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import psycopg2
from psycopg2.extras import RealDictCursor

# Plugin secret
PLUGIN_TOKEN = os.getenv("AUTH_TOKEN")

# PostgreSQL connection parameters
DB_PARAMS = {
    'dbname': os.getenv('DB_NAME', 'organization'),
    'user': os.getenv('DB_USER', 'postgres'),
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': os.getenv('DB_PORT', '5432')
}

app = FastAPI()
security = HTTPBearer()

async def query_postgres() -> List[Dict[str, str]]:
    try:
        conn = psycopg2.connect(**DB_PARAMS)
        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            cur.execute("SELECT tenant, status FROM organization")
            rows = cur.fetchall()
        conn.close()
        return [dict(row) for row in rows]
    except Exception as e:
        print(f"Database error: {e}")
        return []

def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    if credentials.scheme != "Bearer" or credentials.credentials != PLUGIN_TOKEN:
        raise HTTPException(status_code=403, detail="Invalid or missing token")
    return credentials

@app.post("/api/v1/getparams.execute")
async def get_params_execute(credentials: HTTPAuthorizationCredentials = Depends(verify_token)):
    tenants = await query_postgres()
    return {
        "output": {
            "parameters": tenants
        }
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=4355)

```

Let’s send a curl request to test the plugin response : 

```bash
 curl -X POST http://localhost:4355/api/v1/getparams.execute \
  -H "Authorization: Bearer $AUTH_TOKEN" \
  -H "Content-Type: application/json"
```

```bash
{"output":{"parameters":[{"tenant":"Acme Corp","status":"active"},
{"tenant":"Gamma Industries","status":"trial"}]}}
```

All the files, needed for this demo such as Dockerfile, k8s manifests and Postgres DB are in this GitHub repository : https://github.com/tanmay-bhat/argocd-plugin-generator-demo

### Creating ArgoCD Applicationset

Now that we have plugin ready, let’s create an applicationset with below contents : 

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-generator-plugin-demo
  namespace: argocd
spec:
  generators:
  - plugin:
      configMapRef:
        name: tenant-generator-plugin
      requeueAfterSeconds: 120
  goTemplate: true
  template:
    metadata:
      name: 'foobar-app-{{ .tenant | lower | replace "_" "-" | trunc 53 }}'
      labels:
        org: '{{ .tenant }}'
    spec:
      project: default
      destination:
        server: "https://kubernetes.default.svc"
        namespace: foobar-service
      source:
        chart: podinfo
        repoURL: https://stefanprodan.github.io/podinfo
        targetRevision: 6.7.0
        helm:
          parameters:
          - name: "fullnameOverride"
            value: 'foobar-app-{{ .tenant | lower | replace "_" "-" | trunc 53 }}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
          allowEmpty: true
        syncOptions:
          - ApplyOutOfSyncOnly=true
          - CreateNamespace=true
```
`spec.generators[0].plugin.configMapRef` : This is a config which has details on endpoint to connect to plugin and the auth token : 
 

```yaml
apiVersion: v1
data:
  baseUrl: http://tenant-generator-plugin:8080
  token: $api-credentials:plugin.auth.token
kind: ConfigMap
```

- For the token, the syntax is : `$secret_name:plugin.myplugin.token`.

- We can set `goTemplate: true` and then use templating, such has `lower | replace "_" "-"` 

- `requeueAfterSeconds` : The interval on which ArgoCD queries the plugin for updates. 

Here’s how the flow looks like at the end: 

```yaml
[Custom Generator Plugin] --> [ApplicationSet] --> [Argo CD Applications]
      ^                             |
      |                             v
[PostgreSQL Database]        [Kubernetes Cluster]
```

Once we apply the above applicationset, we can see the applications generated for each tenant : 

```bash
k get applications
NAME                             SYNC STATUS   HEALTH STATUS
foobar-app-global-tech           Synced        Healthy
foobar-app-beta-solutions        Synced        Healthy
foobar-app-jupiter-networks      Synced        Healthy
foobar-app-echo-innovations      Synced        Healthy
foobar-app-innova-systems        Synced        Healthy
foobar-app-acme-corp             Synced        Healthy
foobar-app-horizon-enterprises   Synced        Healthy
foobar-app-foxtrot-consulting    Synced        Healthy
foobar-app-gamma-industries      Synced        Healthy
foobar-app-delta-technologies    Synced        Healthy
```

As we can see from above, we have ArgoCD application for each tenant.

![ArgoCD Applications](/argocd-applications-ui.png)

### De-provisioning

As the application is created using ArgoCD, as soon as the plugin stops sending details for a deployed tenant, ArgoCD assumes that its not needed anymore and de-provisions it.

It happens not on request failure, but change in output parameters from plugin. 

### Selective deployments using output fIltering

If we have a requirement to deploy only tenants in `status: active`  and not in `trial` state, we can use the below to filter : 

```yaml
spec:
  generators:
  - plugin:
      configMapRef:
        name: tenant-generator-plugin
    selector:
      matchExpressions:
        - key: status
          operator: In
          values:
            - active
```

It's a standard Kubernetes selector and it's a list, multiple values can be provided.

### Passing Input parameters to Plugin
- Instead of above filtering that will happen on client i.e ArgoCD side, if you implement the logic in plugin service to accept parameters, ArgoCD can send that while fetching tenants, for example : 
```
curl -X POST http://localhost:4355/api/v1/getparams.execute \
-H "Authorization: Bearer $AUTH_TOKEN" \
-H "Content-Type: application/json" \
-d '{
  "applicationSetName": "fake-appset",
  "input": {
    "parameters": {
      "status": "active"
    }
  }
}'
```

Note : 

- As of now, the applicationSet controller doesn't provide built-in metrics for tracking request rates to plugin services. To address this and gain better visibility into your custom generator's performance, consider adding a small metrics exporter within your generator code.
- You can also combine generator plugin with existing generators while creating an application.

---
Reference : 

https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Plugin/

https://pkg.go.dev/github.com/plumber-cd/argocd-applicationset-namespaces-generator-plugin#section-readme

https://github.com/prometheus/client_python

https://github.com/tanmay-bhat/argocd-plugin-generator-demo