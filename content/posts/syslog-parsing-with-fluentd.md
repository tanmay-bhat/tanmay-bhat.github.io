---
layout: post
title: An Overview of Syslog Parsing with Fluentd
date: 2023-02-17
tags: ["Fluentd", "logging", "syslog"]
---

## Introduction

Syslog logging is a widely used method for collecting and storing log data. It is a standard that is supported by many applications and platforms. In this blog post, we will take a look at the basics of syslog parsing with Fluentd.

## What Is Syslog?

Syslog is a protocol used for collecting log data from various sources. It is usually used to collect log data from network devices such as routers and switches, as well as from OS and applications. The log data is then stored on a central syslog server.

## Using Fluentd for Syslog Logging

Fluentd is an open-source data collector that can be used to collect and store syslog data. Fluentd can be configured to filter and parse the log data, making it easier to analyze and process. It can also be used to forward log data to other systems, such as a SIEM or an analytics platform.

### Configure Rsyslog forward with Fluentd

1. Edit the `/etc/rsyslog.conf` file and update it to forward logs to fluentd:

```conf
# Old format to send log messages to Fluentd
#*.* @127.0.0.1:5140

# New format to send log messages to Fluentd
*.* action(type="omfwd" target="127.0.0.1" port="5140" protocol="tcp")
```

2. Restart the rsyslog service:

```bash
systemctl restart rsyslog
```


### Configure Fluentd for syslog input

If you don't have fluentd(td-agent) installed, you can install it using the following command:

```bash
# td-agent 4
curl -fsSL https://toolbelt.treasuredata.com/sh/install-ubuntu-focal-td-agent4.sh | sh
```

You can refer to the following link for more details on how to install fluentd on different platforms: https://docs.fluentd.org/installation

1. Edit the `/etc/td-agent/td-agent.conf` file and add the following configuration:

```conf
<system>
    log_level error
  </system>
<source>
  @type syslog
  port 5140
  bind 0.0.0.0
  tag syslog
  <transport tcp>
  </transport>
  <parse>
    @type syslog
    with_priority true
    message_format rfc3164
  </parse>
</source>

<match syslog.**>
  type stdout
</match>
```

Let's go through the configuration file and understand what each section does:

- `<system>` : sets the log level to error. This will prevent Fluentd from logging too much information into its log file.
- `<source>` : configures the syslog input plugin.
  - The `port` parameter is used to set the port that Fluentd will listen on for syslog messages. The default is 5140.
  - The `bind` is the interface on which fluentd listens for incoming syslog messages.
- The `<transport>` section is used to configure the protocol that Fluentd will use to receive the log data. Default is UDP. I've specified TCP since it ensures message delivery.
- `<parse>` is the configuration block the Fluentd will use to parse the incoming syslog messages. The `tag` parameter is used to set the tag for the log data.
   - `@type syslog` : specifies the parser to use. In this case, we are using the default syslog parser.
   - `with_priority` : specifies whether the syslog message contains a priority field.
   - `message_format` : specifies the format of the syslog message. The default is `rfc3164`. I've specified `rfc5424` since it is the newer syslog format.

- `<match>` : configures the stdout plugin. This plugin will print the parsed log data to the stdout. This is useful for testing / debugging purposes.

2. Restart the td-agent service:

```bash
systemctl restart td-agent
```

3. Validate that Fluentd is listening on port 5140:

```bash
netstat -tulpn | grep 5140
```

### Parsing rfc5424 syslog with Fluentd

To test the syslog ingestion with Fluentd, I'm using a tool called `flog` which will generate syslog messages in both rfc3164 and rfc5424 formats. You can download the binary from the following link:
https://github.com/mingrammer/flog/releases

1. Generate syslog rfc5424 messages using flog:

```bash
flog -f rfc5424 -l -n 1 -d 1
```

This will generate the rfc5424 format syslog messages and print them to the stdout. The `-n` parameter specifies the number of messages to generate. The `-d` parameter specifies the delay between each message.

Example output:

```bash
<100>2 2023-02-11T09:09:19.003Z customermesh.org cum 1023 ID922 - The SAS sensor is down, reboot the wireless hard drive so we can copy the IB port!
<9>1 2023-02-11T09:09:19.003Z dynamicconvergence.io facere 9279 ID665 -  You can't copy the protocol without indexing the wireless RAM program
```

2. Since fluentd is listening on port 5140, we can use netcat to send the generated syslog messages to fluentd:

```bash
sudo flog -f rfc5424 -l -n 1 -d 1 | nc localhost 5140
```

3. We can check if the syslog messages are being received at port 5140 using tcpdump:

```bash
sudo tcpdump -i any -A dst port 5140


09:16:46.596202 IP localhost.16564 > localhost.5140: Flags [P.], seq 163:309, ack 1, win 128, options [nop,nop,TS val 4039646268 ecr 4039645268], length 146
09:16:47.596379 IP localhost.16564 > localhost.5140: Flags [P.], seq 309:460, ack 1, win 128, options [nop,nop,TS val 4039647268 ecr 4039646268], length 151
........<20>1 2023-02-11T09:19:32.541Z dynamice-business.com iste 7111 ID497 - You can't copy the protocol without indexing the wireless RAM program
```

4. Lets verify if the syslog messages are being received by Fluentd from its logs:

```bash
tail -f /var/log/td-agent/td-agent.log

#output
2023-02-11 09:12:32.541000000 +0000 syslog.user.crit: {"host":"dynamicconvergence.io","ident":"corporis","pid":"7866","msgid":"ID616","extradata":"-","message":" You can't copy the protocol without indexing the wireless RAM program"}
2023-02-11 09:12:33.541000000 +0000 syslog.local2.notice: {"host":"chiefb2b.org","ident":"eum","pid":"1867","msgid":"ID551","extradata":"-","message":"You can't compress the array without backing up the bluetooth ADP card!"}
```

The above are the syslog messages that are parsed by Fluentd. That's why you can see the messages are segregated into multiple fields like `host`, `host` etc that will help in readability, complex search queries and faster indexing.


### Parsing rfc3164 syslog with Fluentd :

1. Generate syslog rfc3164 messages using flog:

```bash
flog -f rfc3164 -l -n 1 -d 1
```

2. Update the Fluentd configuration file to parse the rfc3164 syslog messages and restart the td-agent service:

```conf
  <parse>
    @type syslog
    with_priority true
    message_format rfc3164
  </parse>
```

3. Validate that the syslog messages are being parsed by Fluentd:

```bash
tail -f /var/log/td-agent/td-agent.log

#output
2023-02-11 10:50:08.000000000 +0000 syslog.local7.crit: {"host":"gulgowski7481","ident":"qui","pid":"6901","message":"Try to program the EXE panel, maybe it will connect the cross-platform system!"}
2023-02-11 10:50:09.000000000 +0000 syslog.local3.crit: {"host":"mertz1456","ident":"omnis","pid":"2294","message":"If we program the sensor, we can get to the COM panel through the cross-platform SDD application!"}
```

### Parsing custom format syslog with Fluentd :

Let's say we have a custom format syslog message that we want to parse with Fluentd. For example, the following is a custom format syslog message:

```bash
<83>Feb 11 11:26:19 steuber4240 compressing the back-end SSL firewall
```
The above log sample doesn't have `ident` and `msgid` fields. We can use Fluentd's `regex` parser to parse the custom format syslog messages.

A thing to note when it comes to parsing custom format syslog messages is that it expects the incoming logs to have priority field by default, if your log doesn't have a priority field, you can disable it by setting `with_priority` to `false` in the Fluentd configuration, or we can also use `tcp` input plugin as well since it's almost identical to syslog input plugin.

The following is the Fluentd configuration that will parse the custom format syslog messages:

```conf
<source>
  @type tcp
  port 5140
  bind 0.0.0.0
  @log_level trace
  <transport tcp>
  </transport>
  <parse>
    @type regexp
    expression /^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) *(?<message>.*)$/
    time_format %b %d %H:%M:%S
  </parse>
  tag tcp.events
</source>
```

Let's check if the custom format syslog messages are being parsed by Fluentd:

```bash
tail -f /var/log/td-agent/td-agent.log

#output
2023-02-11 11:26:19.000000000 +0530 tcp.events: {"pri":"83","host":"steuber4240","message":"compressing the back-end SSL firewall"}
```

### Parsing both rfc5424 and rfc3164 syslog with Fluentd :

If you're receiving both rfc5424 and rfc3164 syslog messages, then you can use `message_format`:`auto`in Fluentd configuration to parse both formats:

```conf
  <parse>
    @type syslog
    with_priority true
    message_format auto
  </parse>
```
---
### Resources : 
Fluentd syslog input plugin : https://docs.fluentd.org/input/syslog

Ruby regex validator : https://rubular.com/