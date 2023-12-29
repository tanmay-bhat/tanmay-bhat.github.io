---
layout: post
title: Restricting Label Values in Prometheus via prom-label-proxy
date: 2023-12-29
tags: [prometheus]
---

You might have come across the situation where you want to restrict certain cluster or environment specific data from being queried by users, for example finance data or other business critical data and not every Grafana user should be able to see this data. This is where label-proxy comes in handy.

Label-proxy is a small proxy that sits between Grafana and Prometheus and restricts the label values that are being queried.

### Installation

The installation process is straightforward. Download the binary from [here](https://github.com/prometheus-community/prom-label-proxy/releases), and you're ready to get started.


### Restricting Access to Specific Label Values

If you want to have a separate datasource in Grafana to allow query only for a label values, we can use the below example config :

The `scrape_job` label has multiple values : 
```
curl -s https://prometheus.demo.do.prometheus.io/api/v1/label/scrape_job/values | jq .data
[
  "alertmanager",
  "blackbox",
  "caddy",
  "grafana",
  "node",
  "prometheus",
  "random"
]
```

Let's restrict the scrape_job label to only grafana values : 
```
./prom-label-proxy \
   -label scrape_job \
   -label-value grafana \
   -upstream http://demo.do.prometheus.io:9090 \
   -insecure-listen-address 127.0.0.1:8080
```

Now, let's try to query the metric `prometheus_target_scrape_pool_sync_total` which has that label and see if we can get the data :

```
 curl -s "http://127.0.0.1:8081/api/v1/query?query=prometheus_target_scrape_pool_sync_total" | jq '.data.result[]'

{
  "metric": {
    "__name__": "prometheus_target_scrape_pool_sync_total",
    "instance": "demo.do.prometheus.io:9090",
    "job": "prometheus",
    "scrape_job": "grafana"
  },
  "value": [
    1703860408.019,
    "4614"
  ]
}
```

As you can see, we are able to query the data for only `grafana` scrape_job label value and all other values are restricted.

We can also specify multiple label values to restrict, for example if we want to allow only `grafana` and `caddy` scrape_job label values, we can use the below config :

```
./prom-label-proxy \
   -label scrape_job \
   -label-value grafana \
   -label-value caddy \
   -upstream http://demo.do.prometheus.io:9090 \
   -insecure-listen-address 127.0.0.1:8080
```

If we try to query the same metric again, we should be able to see only grafana and caddy scrape_job label values :

```
curl -s "http://127.0.0.1:8081/api/v1/query?query=prometheus_target_scrape_pool_sync_total" | jq '.data.result[].metric | {__name__, scrape_job}'

{
  "__name__": "prometheus_target_scrape_pool_sync_total",
  "scrape_job": "caddy"
}
{
  "__name__": "prometheus_target_scrape_pool_sync_total",
  "scrape_job": "grafana"
}
```

### Enforcing Label Values for All Queries

We can also enforce a specific label value for all queries, for example if we want to enforce `scrape_job` label value for all queries, we can use the below config :

```
./prom-label-proxy \
   -label scrape_job \
   -upstream http://demo.do.prometheus.io:9090 \
   -insecure-listen-address 127.0.0.1:8080
```

What this does is, it checks for all incoming queries and if the query has `scrape_job` label, it will add the label value to the query and forward it to Prometheus. If the query doesn't have `scrape_job` label, it will return an error.

```
curl "http://127.0.0.1:8081/api/v1/query?query=node_memory_MemAvailable_bytes"

{"error":"The \"scrape_job\" query parameter must be provided.","errorType":"prom-label-proxy","status":"error"}
```

### Use Cases

- If you have a central Prometheus server where data is not stored in native multi-tenant mode (Thanos or Cortex) and you want to restrict certain data from being queried by different team users like financial data or namespace specific data.
- You are a multi tenant SaaS provider and want to expose certain internal Prometheus metrics via Grafana iframe to your customers via your portal but want to restrict them to query only specific metric and for their own tenant.
