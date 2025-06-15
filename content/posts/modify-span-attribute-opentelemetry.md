---
layout: post
title: How to modify span attributes in OpenTelemetry using Hooks and Span Processors
date: 2025-06-15
tags: ["opentelemetry", "spans"]
---

# Introduction
In OpenTelemetry, spans are the building blocks of distributed tracing. They represent a single operation within a trace and can have various attributes (labels) associated with them. Sometimes, you may need to modify these attributes such as adding extra attribute or even removing them. In this post, we will explore how to modify span attributes in OpenTelemetry.

# Motivation
I was recently working on adding OpenTelemetry to a Python ETL application. The application was fetching objects from an S3 bucket, and I wanted to modify the span attributes to include the bucket name in the span. As the application was using the `boto3` library, and OpenTelemetry has out of the box instrumentation for `botocore` using `BotocoreInstrumentor` which adds a few span attributes like `cloud.region, rpc.method, rpc.service` but not the bucket name. So I had to modify the span attributes to include the bucket name.


## How to add S3 bucket name to span attributes in OpenTelemetry

Let's say we have the code below to list objects in an S3 bucket using `boto3`:

```python
def list_s3_objects(bucket_name: str, max_keys: int = 100) -> List[Dict[str, Any]]:
    s3_client = boto3.client('s3')
    try:
        response = s3_client.list_objects_v2(Bucket=bucket_name, MaxKeys=max_keys)
        if 'Contents' in response:
            return response['Contents']
        else:
            return []
    ... # rest of the code
```

And let's say we have already instrumented the `boto3` library using OpenTelemetry's `BotocoreInstrumentor`:

```python
from opentelemetry.instrumentation.botocore import BotocoreInstrumentor
BotocoreInstrumentor().instrument()
```

This will produce a span for the `list_objects_v2` operation as shown below:

```json
"spans": [
        {
            "operationName": "S3.ListObjectsV2",
            "spanID": "a13024b2575f7873",
            "tags": [
                {
                    "key": "cloud.region",
                    "type": "string",
                    "value": "us-west-2"
                },
                {
                    "key": "http.status_code",
                    "type": "int64",
                    "value": 200
                },
                {
                    "key": "rpc.method",
                    "type": "string",
                    "value": "ListObjectsV2"
                },
                {
                    "key": "rpc.service",
                    "type": "string",
                    "value": "S3"
                },
                {
                    "key": "rpc.system",
                    "type": "string",
                    "value": "aws-api"
                },
                {
                    "key": "server.address",
                    "type": "string",
                    "value": "s3.us-west-2.amazonaws.com"
                }
                    ],
                    "traceID": "4ba03977861af9541cacc87f559d03be",
                }
            ],
            ...
```

As we didn't create this span manually, we can modify the span using two methods. I used span.name to identify the span, because all S3 spans start with "S3." like S3.ListObjectsV2, S3.GetObject, etc.

## Modifying span attributes using SpanProcessor

To modify the span attributes, we can create a custom `SpanProcessor`. This processor will update the span once its created and updates its attributes.

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.botocore import BotocoreInstrumentor

class CustomSpanProcessor(BatchSpanProcessor):
    # Called when span is started
    def on_start(self, span, parent_context=None):
        if span.name.startswith("S3.") and api_params.get("Bucket"):
            span.set_attribute("bucket.name", api_params["Bucket"])

provider = TracerProvider()
otlp_exporter = OTLPSpanExporter()
processor = CustomSpanProcessor(otlp_exporter)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
trace.get_tracer(__name__)
```

Here, we created a custom `CustomSpanProcessor` that inherits from `BatchSpanProcessor`. In the `on_start` method, we check if the span name starts with "S3." and then set the `bucket.name` attribute.

There's also a `on_end` method that can be used to modify the span after it has finished, but in this case, we only need to modify it when it starts.

## Modifying span attributes using request hook

Another way to modify span attributes is by using hooks. There are two types of hooks, request_hook and response_hook. The request hook is called before the request is sent, and the response hook is called after the response is received.

```python
from opentelemetry.instrumentation.botocore import BotocoreInstrumentor


def s3_request_instrument_hook(span, service_name, operation_name, api_params):
    if span and span.is_recording() and span.name.startswith("S3.") and api_params.get("Bucket"):
        span.set_attribute("bucket.name", api_params["Bucket"])

BotocoreInstrumentor().instrument(request_hook=s3_request_instrument_hook)
```

`is_recording()` checks if the span is active, only active and required spans should be exported to reduce computation and network overhead.

Once we've modified the span attributes, we can see that the span now includes the `bucket.name` attribute:

```json
"spans": [
        {
            "operationName": "S3.ListObjectsV2",
            "spanID": "a14024b2975f7873",
            "tags": [
                {
                    "key": "cloud.region",
                    "type": "string",
                    "value": "us-west-2"
                },
                {
                    "key": "http.status_code",
                    "type": "int64",
                    "value": 200
                },
                {
                    "key": "rpc.method",
                    "type": "string",
                    "value": "ListObjectsV2"
                },
                {
                    "key": "rpc.service",
                    "type": "string",
                    "value": "S3"
                },
                {
                    "key": "rpc.system",
                    "type": "string",
                    "value": "aws-api"
                },
                {
                    "key": "server.address",
                    "type": "string",
                    "value": "s3.us-west-2.amazonaws.com"
                },
                {
                    "key": "bucket.name",
                    "type": "string",
                    "value": "my-s3-bucket"
                }
            ],
            ...
            "traceID": "4ba03977861af9541cacc87f559d03be",
        }
    ],
    ...
```

# References
https://opentelemetry-python-kinvolk.readthedocs.io/en/latest/instrumentation/botocore/botocore.html

https://github.com/open-telemetry/opentelemetry-python-contrib/blob/main/instrumentation/opentelemetry-instrumentation-botocore/src/opentelemetry/instrumentation/botocore/__init__.py