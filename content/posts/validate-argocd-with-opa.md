---
layout: post
title: Validating ArgoCD Application Manifest With Open Policy Agent
tags: ["ArgoCD", "OPA"]
date: 2024-09-13
---

In the world of GitOps, even experienced DevOps engineers occasionally encounter issues stemming from simple typos or misconfigurations in these YAML files. To mitigate these risks and ensure compliance with organizational policies, we can bring in validation tools. 

This post explores two powerful options: Conftest, which leverages Open Policy Agent (OPA), and Kubeconform. We'll dive into how these tools can be implemented to streamline your validation processes.

### ArgoCD custom policy validation with Conftest (OPA)

Open Policy Agent's Conftest serves as an excellent tool for custom config rule validation, offering a more flexible alternative to script-based validation.

You can write any custom logic that is required as per your organization and enforce those rules easily.

To get started, download the Conftest binary from the Github [releases page.](https://github.com/open-policy-agent/conftest/releases/tag/v0.55.0)

Conftest policies are written in Rego, a declarative language built on top of Go. Here's a basic policy to validate the existence of a required label:

Here are a few example policies to demonstrate a couple of use cases :

#### Enforcing label on each application : 
```go
package main

required_labels := {"owner-team", "cost-center", "environment"}

deny[msg] {
    some required_label in required_labels
    not input.metadata.labels[required_label]
    msg := sprintf("Missing required label: '%v'", [required_label])
}
```
One can use iteration through the `some` keyword, allowing you to traverse YAML structures. The `input` object serves as the root of your YAML document, from which you can access nested fields using dot notation.

```
FAIL - kubeseal.yaml - main - Missing required label: 'owner-team'
FAIL - kubeseal.yaml - main - Missing required label: 'cost-center'
FAIL - kubeseal.yaml - main - Missing required label: 'environment'

3 tests, 0 passed, 0 warnings, 3 failures, 0 exceptions

```

#### Enforce SSH for git repositories :

```go
deny[msg] {
    not startswith(input.spec.source.repoURL, "git@github.com:")
    msg := sprintf("Invalid repoURL: '%v'. Must start with 'git@github.com:'", [input.spec.source.repoURL])
}
```

#### Restrict ArgoCD projects:

```go
allowed_projects := {"prod", "staging", "dev"}

deny[msg] {
    not allowed_projects[input.spec.project]
    msg := sprintf("Project '%v' is not allowed. Must be one of: %v", [input.spec.project, allowed_projects])
}
```
```
FAIL - kubeseal.yaml - main - Project 'foobar' is not allowed. Must be one of: {"dev", "prod", "staging"}
```

#### Warn on suggested sync changes
```go
warn[msg] {
    not "ApplyOutOfSyncOnly=true" in input.spec.syncPolicy.syncOptions
    msg := "ApplyOutOfSyncOnly=true is not set in syncOptions. This may lead to unnecessary syncs."
}

warn[msg] {
    not input.spec.syncPolicy.automated.selfHeal
    msg := "SelfHeal is not enabled in syncPolicy.automated. This may prevent automatic drift corrections."
}
```

### ArgoCD schema validation with Kubeconform

Kubeconform is a powerful Kubernetes resource schema validator that supports custom resources. It's particularly useful for catching common typos and structural errors in ArgoCD Application resources.
Download the Kubeconform binary from the official releases page.
Let's validate an example manifest with an incorrect structure:

To test the schema validation, let's have an example manifest with incorrect value saved as `kubeseal.yaml` : 
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  project: default
  sources:
    chart: sealed-secrets
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    targetRevision: 1.16.1
    helm:
      releaseName: sealed-secrets
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kubeseal
```
 When we run schema validation with kubeconform : 

```
kubeconform --output pretty -strict -exit-on-error \
 -schema-location default -schema-location \
 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' kubeseal.yaml
```

```
✖ argocd-applications/us-west-2/prod-us-new/bad-example.yaml: Application sealed-secrets is invalid: 

problem validating schema. Check JSON formatting: jsonschema: '/spec/sources' does not validate with https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/argoproj.io/application_v1alpha1.json#/properties/spec/properties/sources/type: expected array, but got object
```

The error message shows a schema violation: the `spec.sources` field should be an array, but in our example, it was incorrectly defined as a single object. This type of structural error is exactly what Kubeconform excels at catching.

Kubeconform can be run on directory as well, allowing for batch validation.

---

### Notable points
- Both conftest and kubeconform ar best suited to run inside CI to catch before its committed, just like any other test cases.
- Although some of these use cases can be caught if the application manifests are generated as part of helm  template, they still provide easy to configure validations.
- Conftest extends beyond kubernetes, can be applied to terraform, docker and other configurations as well.

---

### References : 
https://www.conftest.dev/

https://www.openpolicyagent.org/