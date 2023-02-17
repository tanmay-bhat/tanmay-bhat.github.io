---
layout: post
title: How to Configure Alerting on Ingress-NGINX in Kubernetes
date: 2023-01-30
tags: ["ingress", "kubernetes", "prometheus"]
canonicalUrl: "https://www.aviator.co/blog/how-to-monitor-and-alert-on-nginx-ingress-in-kubernetes/"
---

In this blog post, we will be discussing how to set up monitoring and alerting for Nginx ingress in a Kubernetes environment.

We will cover the installation and configuration of ingress-nginx, Prometheus and Grafana, and the setup of alerts for key Ingress metrics. 

### Pre-requisites :

- A Kubernetes cluster
- Helm v3

## Install Prometheus and Grafana

In this step, we will install Prometheus to collect metrics, and Grafana to visualize and create alerts based on those metrics.

Let's install the **[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)** helm chart by copying the below-mentioned commands to your terminal. This will setup Grafana, Prometheus and other monitoring components.


```bash
# Add and update the prometheus-community helm repository.
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
  
```bash
cat <<EOF | helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
--create-namespace -n monitoring -f -

grafana:
  enabled: true
  adminPassword: "admin"
  persistence:
    enabled: true
    accessModes: ["ReadWriteOnce"]
    size: 1Gi
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.localdev.me
EOF
```

Let's see if the installed components are up and running : 

```bash
kubectl get pods -n monitoring

NAME                                                        READY   STATUS    RESTARTS        AGE
kube-prometheus-stack-grafana-7bb55544c9-qwkrg              3/3     Running   0               3m38s
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   0               3m14s
...
```
Looks great, let's proceed to the next section.

### Install & Configure Ingress Nginx

In this step, we will install and configure Nginx ingress controller and enable metrics that can be scraped by Prometheus.

1. Let's install **ingress-nginx** into our cluster using below command: 

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=true \
  --set controller.metrics.serviceMonitor.additionalLabels.release="kube-prometheus-stack"
```

Here we're specifying `serviceMonitor.additionalLabels` to be `release: kube-prometheus-stack` so that Prometheus can discover the service monitor and automatically scrape metrics from it.

2. Once the chart is installed, let’s deploy a sample app **podinfo** into `default` namespace.

```bash
helm install --wait podinfo --namespace default \
oci://ghcr.io/stefanprodan/charts/podinfo
```

3. Now, create an ingress for the deployed **podinfo** deployment  : 

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo-ingress
spec:
  ingressClassName: nginx
  rules :
  - host: podinfo.localdev.me
  defaultBackend:
    service:
      name: podinfo
      port:
        number: 9898
EOF
```

Let’s understand a bit about the above ingress config : 

- We’re using `ingress-nginx` as our ingress controller, hence the ingress class is defined as `nginx`.
- In the above config, I’ve used the host address for my Ingress as `podinfo.localdev.me`.
- The DNS `*.localdev.me` resolves to `127.0.0.1`, hence for any local testing this DNS can be used without the hassle of adding an entry into `/etc/hosts` file.
- Podinfo app serves HTTP API over port `9898`, hence it's specified for the backend port i.e. when the traffic arrives for the domain `http://podinfo.localdev.me`, it will be forwarded to `9898` of podinfo service.

4. Next, from your terminal, port-forward the `ingress-nginx` service so that you can send traffic from your local terminal. 

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80  > /dev/null &
```

- Port `80` on the host is a privileged port, so we’re not using that, instead we’re binding port `80` of nginx service to `8080` of host machine. You can specify any valid port of your choice.

Note:  If you’re running this in any cloud, port-forwarding is not required as LoadBalancer for **ingress-nginx** service will be auto-created since the service type is defined as `LoadBalancer` by default.

5. Now, you can run the below curl request to the podinfo endpoint, which should respond with : 

```bash
> curl http://podinfo.localdev.me:8080

"hostname": "podinfo-59cd496d88-8dcsx"
"message": "greetings from podinfo v6.2.2"
```
6. You can also get the prettier look in the browser with URL : [http://podinfo.localdev.me:8080/](http://podinfo.localdev.me:8080/)

![Podinfo web look](/podinfo-web.png)

## Configure Grafana Dashboards for Ingress Nginx Monitoring

To access Grafana, you can open the below URL in your browser with the credential `admin:admin` :  [http://grafana.localdev.me:8080/](http://grafana.localdev.me:8080/).

Copy the`nginx.json` from [here](https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/grafana/dashboards/nginx.json) and paste it into [http://grafana.localdev.me:8080/dashboard/import](http://grafana.localdev.me:8080/dashboard/import) to import the dashboard.

Once imported the dashboard should look like this :

![ingress-nginx Grafana dashboard](/nginx-ingress-dashboard.png)

## Alerting over SLI metrics

Now that we have the dashboard and metrics ready in our Grafana, it’s time to set alerting on important SLI like Error Rate & Latency.

### Generate sample loads

Inorder to get traffic on our my podinfo application, we’ll be using [vegeta](https://github.com/tsenart/vegeta) as a loadtesting tool. Please install it from [here](https://github.com/tsenart/vegeta).

Let's generate a sample HTTP 4xx traffic. To do that, you can run the below command which will run at a request rate of 10 RPS for 10 minutes.

```bash
echo "GET http://podinfo.localdev.me:8080/status/400" | vegeta attack -duration=10m -rate=10/s
```

- You can just change the status code from 400 to 500 and run as well for test 5xx throughput.

For latency tests, I’ve used the below command as `GET /delay/{seconds}
 waits for the specified period` :

```bash
echo "GET http://podinfo.localdev.me:8080/delay/3" | vegeta attack -duration=10m -rate=100/s
```

Note: You can read more on the endpoints available in podinfo app from [here](https://github.com/stefanprodan/podinfo).

### Grafana Alerting

The newer Grafana comes with its own alerting engine. That helps in keeping all config, rules and even firing alertsin one place. Let's configure alerts for common SLI.

### 4xx Error Rate

1. Let's create an alert by going to [http://grafana.localdev.me:8080/alerting/new](http://grafana.localdev.me:8080/alerting/new)
2. We can use the following formula to get 4xx error rate percentage :

`(total number of 4xx requests / total number of requests) * 100`

3. Add the below expression for the query :

```bash
(sum(rate(nginx_ingress_controller_requests{status=~'4..'}[1m])) by (ingress) / sum(rate(nginx_ingress_controller_requests[1m])) by (ingress)) * 100 > 5
```

4. In Expression `B`, use `Reduce` operation with the function `Mean`  for input `A`.

![ingress-nginx grafana alerting config](/nginx-ingress-alert-config.png)

5. In Alert Details, Name the alert as per your liking, I’ve named it `Ingress_Nginx_4xx`.
6. For Summary, we can keep it as short as possible, by just displaying the Ingress name with label `{{ $labels.ingress }}`.

```go
Ingress High Error Rate : 4xx on *{{ $labels.ingress }}*
```

7. For Description, I’ve used `printf "%0.2f"` to display upto two decimals on the percentage value.

```go
4xx : High Error rate : `{{ printf "%0.2f" $values.B.Value }}%` on *{{ $labels.ingress }}*.
```

8. Overall alert should look similar to the below snapshot :

![ingress-nginx grafana alert details](/nginx-ingress-alert-details.png)

9. In the end, you can add a custom label like `severity : critical`.

### 5xx Error Rate

Similar to 4xx alert config, 5xx error rate can also be queried with the below query :

```go
sum(rate(nginx_ingress_controller_requests{status=~'5..'}[1m])) by (ingress,cluster) / sum(rate(nginx_ingress_controller_requests[1m]))by (ingress) * 100 > 5
```

Note: I’ve configured the alert to be triggered then the 5xx/4xx percentage is > 5%. You can set it as per your error budget.

### High Latency (p95)

To calculate the 95th percentile of request durations over the last 15m we can use the `nginx_ingress_controller_request_duration_seconds_bucket` metric.

It gives you **The request processing time in milliseconds** and since its a bucket we can use `histogram_quantile` function.

Similarly create a alert to above examples and use the below query :

```go
histogram_quantile(0.95,sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[15m])) by (le,ingress)) > 1.5
```

I’ve set the threshold to 1.5 seconds, it can be updated as per your SLO.

### High Request rate

To get the request rate per second (RPS), we can use the below query :

```go
sum(rate(nginx_ingress_controller_requests[5m])) by (ingress) > 2000
```

The above query will trigger an alert when the request rate is greater than 2000 RPS.

### Other SLIs to consider

**Connection rate**: This measures the number of active connections to Nginx ingress, and can be used to identify potential issues with connection handling.

```bash
rate(nginx_ingress_controller_nginx_process_connections{ingress="ingress-name"}[5m])
```

**Upstream response time**: The time it takes for the underlying service to respond to a request, this will help to identify issues with the service and not just the ingress.
```bash
histogram_quantile(0.95,sum(rate(nginx_ingress_controller_response_duration_seconds_bucket[15m])) by (le,ingress)) 
```

### Slack Alert Template

To make alert messages meaningful, we can use [alert templates in Grafana.](https://grafana.com/docs/grafana/latest/alerting/contact-points/message-templating/)

1. In order to configure them, go to [http://grafana.localdev.me:8080/alerting/notifications](http://grafana.localdev.me:8080/alerting/notifications) and create a new template named `slack` by pasting the below code block :

```go
{{ define "alert_severity_prefix_emoji" -}}
	{{- if ne .Status "firing" -}}
		:white_check_mark:
	{{- else if eq .CommonLabels.severity "critical" -}}
		:fire:
	{{- else if eq .CommonLabels.severity "warning" -}}
		:warning:
	{{- end -}}
{{- end -}}

{{ define "slack.title" -}}
	{{ template "alert_severity_prefix_emoji" . }}  {{- .Status | toUpper -}}{{- if eq .Status "firing" }} x {{ .Alerts.Firing | len -}}{{- end }}  |  {{ .CommonLabels.alertname -}}
{{- end -}}

{{- define "slack.text" -}}
{{- range .Alerts -}}
{{ if gt (len .Annotations) 0 }}
*Summary*: {{ .Annotations.summary}}
*Description*: {{ .Annotations.description }}
Labels: 
{{ range .Labels.SortedPairs }}{{ if or (eq .Name "ingress") (eq .Name "cluster") }}• {{ .Name }}: `{{ .Value }}`
{{ end }}{{ end }}
{{ end }}
{{ end }}
{{ end }}
```

2. Configure a new contact point of type Slack. For this, you need to create an incoming webhook from Slack. Refer [this doc](https://api.slack.com/messaging/webhooks#create_a_webhook) for more detailed steps.

3. Edit the contact point **slack** and scroll down and select the option **`Optional Slack settings`.**

4. In the **Title,** enter the below to specify the template to use:

```
{{ template "slack.title" . }}
```

5. In the **Text Body,** enter the below and save it :

```
{{ template "slack.text" . }}
```

6. Go to [http://grafana.localdev.me:8080/alerting/routes](http://grafana.localdev.me:8080/alerting/routes) and configure the **Default contact point** to be **Slack**.

### Finally, the alert message arrives!

After configuring all the steps, finally we arrive at the end and below are the snapshots of how the alert will look on your slack.

4xx Error Rate : 

![4xx slack alert](/4xx-slack-alert.png)

5xx Error Rate : 

![5xx slack alert](/5xx-slack-alert.png)

Latency P95 : 

![p95 slack alert](/p65-slack-alert.png)

There are lots of things one can improve according to their requirements. For example, if you have mulltple Kubernetes clusters, you can add a `cluster` label which will help in identifying the source cluster for the alert.

---
### References

[https://grafana.com/docs/grafana/latest/alerting/](https://grafana.com/docs/grafana/latest/alerting/)

[https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/](https://kubernetes.github.io/ingress-nginx/user-guide/monitoring/)

[https://github.com/stefanprodan/podinfo](https://github.com/stefanprodan/podinfo)

---