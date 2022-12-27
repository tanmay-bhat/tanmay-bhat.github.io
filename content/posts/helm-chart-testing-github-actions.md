---
layout: post
title: Automate Your Helm Chart Testing Workflow with GitHub Actions
date: 2022-12-27
tags: ["Helm", "Kubernetes"]
---

Helm is a popular open-source package manager for Kubernetes that simplifies the process of installing, upgrading, and managing applications on a Kubernetes cluster. Helm packages, called charts, contain all of the necessary resources and configuration for deploying an application on a Kubernetes cluster.

As with any software project, it's important to test charts before deploying them to ensure that they are reliable and function as intended. Chart testing is the process of verifying the functionality and correctness of a Helm chart before it is deployed to a Kubernetes cluster.

There are a variety of tools and approaches available for testing Helm charts, ranging from simple shell scripts to more advanced testing frameworks. In this blog post, we'll explore how to use the [chart-testing](https://github.com/helm/chart-testing) tool, which is maintained by the Helm project, to test Helm charts with Github Actions.

## Helm Chart Testing for Pull Requests

1. Lets create a new file in the `.github/workflows` directory of your repository called `chart-testing.yml`. This file will define the workflow and specify the steps required to run the chart tests.

Update the `chart-testing.yml` file with below contents : 

```
name: Lint and Test Charts

on: pull_request

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Set up Helm
        uses: azure/setup-helm@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: 3.7

      - name: Set up chart-testing
        uses: helm/chart-testing-action@v2.3.1

      - name: Run chart-testing (list-changed)
        id: list-changed
        run: |
          changed=$(ct list-changed --config ct.yaml)
          if [[ -n "$changed" ]]; then
            echo "{changed}={true}" >> $GITHUB_OUTPUT
          fi

      - name: Run chart-testing (lint)
        run: ct lint --config ct.yaml

      - name: Create kind cluster
        uses: helm/kind-action@v1.5.0
        if: ${{ needs.list-changed.outputs.changed }} == 'true'

      - name: Run chart-testing (install)
        run: ct install --config ct.yaml
```

Let’s understand the above config in detail.

- `on: pull_request` : This indicates when the workflow should trigger, in our case whenever a pull request is opened.
- `Checkout`: This step uses the `actions/checkout@v3` action to check out (clone) the code in the repository so that the workflow can access it. The `fetch-depth` parameter is set to `0` to fetch the entire repository history i.e all history for all branches and tags.
- `Set up Helm`: This step uses the `azure/setup-helm@v3` action to install and set up Helm v3 latest on the machine.
- `Set up Python`: This step uses the `actions/setup-python@v4` action to install and set up Python on the machine. The `python-version` parameter is set to `3.7`.
- `Set up chart-testing`: This step uses the `helm/chart-testing-action@v2.3.1` action to install and set up chart-testing, a tool for testing Helm charts.
- `Run chart-testing (list-changed)`: This step runs the `ct list-changed` command using chart-testing to list the charts that have changed since the last commit by referring to the `main` target branch. The `id` field is used to give this step an identifier, which can be used to reference it later in the workflow execution. If there are changed charts, the step sets an output named `changed` to `true` using the `echo "{changed}={true}" >> $GITHUB_OUTPUT` syntax.
- `Run chart-testing (lint)`: This step runs the `ct lint` command using chart-testing to lint the charts in the repository.
- `Create kind cluster`: This step uses the `helm/kind-action@v1.5.0` action to create a Kubernetes cluster using kind (Kubernetes IN Docker). The `if` field is used to specify that this step should only be run if the `changed` output of the `list-changed` step is `true`. I.e this step will only run if there are changes in the helm charts compared to main branch helm charts, basically git diff between them result should not be empty.
- `Run chart-testing (install)`: This step runs the `ct install` command using chart-testing to install the charts in the repository into the kind cluster.

2. Now, create a file called `ct.yaml` which will store the configs for the chart-testing : 

```bash
target-branch: main
chart-dirs:
  - deploy/helm/charts
helm-extra-args: --timeout 600s
check-version-increment: false
```

- The `target-branch` field specifies the target branch used to identify changed charts.

- The `chart-dirs` field specifies the directories where the charts to be tested are located. In this case, the value is `deploy/charts`, which means that the charts are located in the `deploy/charts` directory. Update this according to your directory structure.

- The `helm-extra-args` field specifies additional arguments to pass to the Helm command when installing or upgrading charts. In this case, the value is `--timeout 600s`, which means that the Helm command will have a timeout of 600 seconds (10 minutes).

- The `check-version-increment` field specifies whether chart-testing should check that the chart version is incremented in `Chart.yaml`. In this case, the value is `false`

The directory structure finally looks like this : 

```bash
Repository
└── ct.yaml
└── Deploy
		└── Charts
		    └── example-helm-chart
		        ├── Chart.yaml
		        ├── templates
		        │   ├── deployment.yaml
		        │   ├── ingress.yaml
		        │   ├── service.yaml
		        └── values.yaml
```

Now that we have configure, we can see the flow in action.

Consider a case where I wanted to enable Ingress for my app, but didn’t complete the all correct values in the helm chart and pushed to my `test-branch` .

```bash
#supplied values for my-app
ingress:
  enabled: true
  annotations: {}
  hosts:
    - host: chart-example.local
      paths: []
#I didn't specify the path and helm install should fail.
```

As you can see the below, the Action failed :

![helm-actions-fail-image](/helm-actions-fail.png)

Since the problem is with helm templating, the lint step passses correctly.

```bash
Run ct lint --config ct.yaml
Linting charts...

------------------------------------------------------------------------------------------------------------------------
 Charts to be processed:
------------------------------------------------------------------------------------------------------------------------
my-app => (version: "0.2.8", path: "charts/my-app")
------------------------------------------------------------------------------------------------------------------------

Linting chart "my-app => (version: \"0.2.8\", path: \"charts/my-app\")"
Validating /home/runner/work/helm-actions-demo/my-app/Chart.yaml...
Validation success! 👍
Validating maintainers...

==> Linting /charts/my-app

1 chart(s) linted, 0 chart(s) failed
------------------------------------------------------------------------------------------------------------------------
 ✔︎ example-v2 => (version: "0.2.8", path: "charts/my-app")
------------------------------------------------------------------------------------------------------------------------
All charts linted successfully
```

- Helm templating and helm install should fail since the Ingress config is incomplete :

```bash
Run ct install --config ct.yaml
Installing charts...

------------------------------------------------------------------------------------------------------------------------
 Charts to be processed:
------------------------------------------------------------------------------------------------------------------------
 my-app => (version: "0.2.8", path: "charts/my-app")
------------------------------------------------------------------------------------------------------------------------

Installing chart "my-app => (version: \"0.2.8\", path: \"my-app\")"...

Error: INSTALLATION FAILED: Ingress.extensions "my-app-ingress" is invalid: spec.rules[0].http.paths: Required value
```

### Trigger Github Actions for Push and other Events

Similar to the above cconfig, you can just configure Github actions to trigger whenever a push to the main ( or any other) branch happens. Or you can combine the trigger with push and pull reqeuest.

```bash
#For both 
on: [pull_request,push]

#For Push event
on: push

#For push to specifc branches
on:
  push:
    branches:
      - main
      - test-branch
#For PR on specific branches
on:
  pull_request:
    branches:
      - main
      - develop
```

## Specifying Multiple Helm Values Files

You can specify values in a folder called `ci`located in the root of your chart repository. The `ci` folder should contain a `values.yaml`file with the desired values, and chart-testing will automatically use these values when running the `ct install`command. 

For example, if you have a chart repository with the following structure:

```bash
Repository
└── ct.yaml
└── Deploy
		└── Charts
		    └── example-helm-chart
		        ├── ci
		        │   ├── values-us-east-1.yaml
		        │   ├── values-us-west-2.yaml
		        │   └── values-eu-west-1.yaml
		        ├── Chart.yaml
		        ├── templates
		        │   ├── deployment.yaml
		        │   ├── ingress.yaml
		        │   ├── service.yaml
		        └── values.yaml
```

It will iterate over every `values-*.yaml` and the chart is installed and tested for each of these files.

### Helm Upgrades

To test an in-place upgrade of a chart you can use the `upgrade` flag with the `ct install`
 command. This flag will cause chart-testing to test an in-place upgrade of the chart from its previous revision.

Add `upgrade: true` into your `ct.yaml` to enable the upgrade of the Helm charts.

If the current version should not introduce a breaking change according to the SemVer specification, the upgrade will be tested.

### Handling Dependencies in Chart Testing

When testing Helm charts that have dependencies on other charts, it is important to ensure that these dependencies are properly installed and configured in order for the tests to be successful. By default, `ct install` handles dependency of helm charts i.e it will run `helm dependency build` to fetch the chart specified.

- You need add the repository in `ct.yaml` so that repository index can be fetched, Bitnami in this example :
```
chart-repos:
  - bitnami=https://charts.bitnami.com/bitnami
```
- Next you need to specify the required chart in your `Chart.yaml` of your specific helm chart, Redis for Example : 
```
dependencies:
  - name: redis
    version: 17.4.0
    repository: https://charts.bitnami.com/bitnami
```
- Then you can use your custom values for the Redis chart in the `values.yaml`, for example : 

```
redis:
  enabled: true
  cluster:
    enabled: false
```
---

### References :

[https://github.com/helm/chart-testing](https://github.com/helm/chart-testing)

[https://github.com/marketplace/actions/helm-chart-testing](https://github.com/marketplace/actions/helm-chart-testing)

[https://docs.github.com/en/actions/using-jobs/using-jobs-in-a-workflow#overview](https://docs.github.com/en/actions/using-jobs/using-jobs-in-a-workflow#overview)