---
layout: post
title: 'How to Add Custom Label or Key to Records in Fluentd'
date: 2023-07-30
tags: ["Fluentd", "logging"]
---

Fluentd is a powerful log collection and processing tool. In this blog post, I will walk you through the process of adding custom label and key to the log records, so that you can better understand your logs and filter them more effectively.

For example, you might want to add a custom label to your records to indicate the environment in which the log was generated, or the cluster name. This would allow you to filter your logs by environment or cluster name in Elasticsearch or another storage destination.


### Adding a custom label or key into the records

Say you have a Nginx service which is sending logs to fluentd, and you want to add a custom label i.e., cluster name into the records.

For this, I'm using the below docker-compose file to spin up a nginx service and a fluentd service :

```yaml
services:
  fake-logger:
    container_name: flog
    image: tanmaybhat/flog-multiarch
    command: -d 1
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: nginx

  fluentd:
    container_name: fluentd
    image: fluent/fluentd-kubernetes-daemonset:v1.16.2-debian-elasticsearch8-1.0
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224"
      - "24224:24224/udp"
``````

Here's what the docker-compose file is doing:

1. Spinning up a logger service which sends fake nginx logs to stdout one line at a time.
2. We are using fluentd as the logging driver for the logger service and add the tag as nginx to easily identify the logs later.
3. We are running a fluentd service which is using the Docker image which has the elasticsearch plugin installed. (Optional)
4. Finally we are mounting the fluentd configuration file from the host to the container.

Now, lets see how the fluentd configuration file `./fluentd/conf/fluent.conf` looks like:

```yaml
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<match **>
  @type stdout
</match>
```

In the above configuration file, we are using the forward plugin to receive the logs from the Docker logger service and then using the stdout plugin to print the logs to the stdout.

Let's run both the services using the below command:

```bash
docker-compose up
```

If you observe the logs this is how its going to look like:

```bash
flog | 217.32.48.100 - [30/Jul/2023:18:05:00] "GET /synergize/platforms HTTP/1.0" 200 14231

fluentd | 2023-07-30 18:05:01 nginx: {"container_name":"flog","source":"stdout","log":"217.32.48.100 - [30/Jul/2023:18:05:00 +0000] \"GET /synergize/platforms HTTP/1.0\" 200 14231","container_id":"xxx"}
```

We can now add a custom label or key into the records using the `record_transformer`` plugin. Let's see how the configuration file looks like:

```yaml
<filter **>
  @type record_transformer
  enable_ruby true
  auto_typecast true
  renew_record true
  <record>
    cluster "lab_cluster"
  </record>
</filter>
```

Once the configuration file is updated, the logs will now look like this:

```bash
flog     | 91.143.216.150 - walker6126 [30/Jul/2023:18:13:26 +0000] "PATCH /relationships HTTP/1.0" 200 8286
fluentd  | 2023-07-30 18:13:27.000000000 +0000 nginx: {"cluster":"lab_cluster"}
```

Interesting right? fluentd is now adding an extra key called `cluster` into the records. When you use the `record_transformer` plugin and add an extra key, Anything you add in `<record>` will be replaced as the actual record.

In our case, the whole record is replaced with the key `cluster`. But this is not what we want. We'd like the cluster name to be added to the existing log records.

If you observe the previous output from fluentd, the logs are inside a key called `log`. So we need to add the cluster name, keep the log as it is along with the tag. Let's see how we can do that:

```yaml
  <record>
    cluster "lab_cluster"
    tag ${tag}
    log ${record["log"]}
  </record>
```

Here, we are setting log key as the value of the `log` key from the record. `record["<key>"]` is used to get the value of the key from the record.

Any key which is present in the original record can be extracted and be used to add a new key into the record or manipulate the record.

```bash
flog     | 154.255.224.80 - - [30/Jul/2023:18:31:45] "POST /back-end/efficient HTTP/2.0" 405 6598
fluentd  | 2023-07-30 18:31:46 nginx: {"cluster":"lab_cluster","tag":"nginx","log":"154.255.224.80 - - [30/Jul/2023:18:31:45 +0000] \"POST /back-end/efficient HTTP/2.0\" 405 6598"}
```

And here's how it looks in Elasticsearch:

![elasticsearch](/elastic-search-fluentd-output.png)


### Merging the custom label or key with the existing record key

I stumbled upon a unique use case where I had to merge the custom label, say `cluster` into existing record key of `text`. This was needed as we were sending the logs to Coralogix which uses OpenSearch as the backend and any additional key other than text will be ignored.

In this case, I used the `merge` function of ruby to merge the custom label with the existing record key. Let's see how the configuration file looks like :

```yaml
  <record>
    text ${record.merge('cluster' => 'lab_cluster')}
  </record>
```

In production, I suggest to set keys like cluster as environment variables and use them in the configuration file instead of hard-coding them.

```yaml
  <record>
    text ${record.merge('cluster' => ENV['CLUSTER'])}
  </record>
```

---
### Resources:

https://docs.fluentd.org/filter/record_transformer

