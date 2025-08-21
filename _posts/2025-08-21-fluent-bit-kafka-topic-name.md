---
layout: post
title:  "How to Use Kubernetes Metadata for Dynamic Kafka Topics in Fluent Bit"
date:   2025-08-21 18:00:00 +0300
categories: posts
---

Managing logs at scale can be painful, especially when all of them end up in the same Kafka topic. What if you could route logs into **pod-specific topics** (or any custom topic) directly from Fluent Bit? This makes filtering and querying much easier, especially in multi-tenant or large Kubernetes environments. In this post, we’ll explore how to set up **dynamic Kafka topics in Fluent Bit using Kubernetes metadata**.

## The problem
Fluent Bit’s Kafka output plugin supports dynamic topic assignment using `Dynamic_topic on` and `Topic_key`. However, the documentation is vague about using **nested fields**, which is the case for kubernetes metadata fields, as the topic key.

**Example:**
```json
{
   "timestamp":"2025-05-13T19:27:31.954980134Z",
   "log":"Server is listening on port 5000",
   "kubernetes":{
      "namespace_name":"default",
      "pod_id":"de70b4a7-2803-485b-9015-3d17791969ae",
      "labels":{
         "app":"dy-7d8eeb3d4fb6da81d88da56280de7d42",
         "pod-template-hash":"5744dfc46c"
      },
      "container_name":"dy-7d8eeb3d4fb6da81d88da56280de7d42",
      "annotations":{
         "kubectl.kubernetes.io/restartedAt":"2025-05-13T22:27:29+03:00"
      },
      "container_image":"docker.io/library/dy-7d8eeb3d4fb6da81d88da56280de7d42:latest",
      "container_hash":"sha256:d110c044a3e9fed6c3b3c9222a1b4d5cf4bc5f128a1d5f47980abee7ff98f745",
      "host":"kind-control-plane",
      "docker_id":"da27432083a32dada6ccac359d72233a4e93d98d1a139e6b2d4bee7d167c49cc",
      "pod_name":"dy-7d8eeb3d4fb6da81d88da56280de7d42-5744dfc46c-dgsjf"
   }
}
```

At first glance, it looks like we should be able to use `` $kubernetes['pod_name'] `` as Topic_key, since the docs show this syntax in [ filter examples ](https://docs.fluentbit.io/manual/data-pipeline/filters/grep#nested-fields-example). But in practice, this doesn’t work for outputs — only for filters.
<br> <br>
**❌ Configuration that doesn't work:**

```yml
    [OUTPUT]
        Name          kafka
        Match         *
        Brokers       kafka:9093
        Topics        kube
        Topic_key     $kubernetes['pod_name']    # This syntax doesn't work
        Dynamic_topic on
```

## The solution
After some trial and error, I discovered why the config in the docs doesn’t work:

**Fluent Bit’s Kafka output plugin only supports top-level keys in Topic_key**.
<br>
<br>
That means if your field is nested (like `kubernetes.pod_name`), Fluent Bit won’t resolve it directly.

> The trick is to **promote the nested field to the top level** before it reaches the output plugin.

We can do this using a **Lua filter**.

### Step 1: Original log (nested field inside kubernetes)
```json
{
  "timestamp":"2025-05-13T19:27:31.954980134Z",
  "log":"Server is listening on port 5000",
  "kubernetes": {
    "namespace_name":"default",
    "pod_name":"dy-7d8eeb3d4fb6da81d88da56280de7d42-5744dfc46c-dgsjf"
  }
}
```

### Step 2: Lua filter to promote nested field
```lua
function promote_pod_name(tag, timestamp, record)
    local k8s = record["kubernetes"]
    if k8s and k8s["pod_name"] then
        record["topic_name"] = k8s["pod_name"]
    else
        record["topic_name"] = "default-topic" -- fallback
    end
    return 1, timestamp, record
end
```
Quick Note:
- 1 means "keep the record"
- `timestamp` is passed through filter unchanged
- `record` is the updated log with _topic_name_ at top level

### Step 3: Log after Lua filter

```json
{
  "timestamp":"2025-05-13T19:27:31.954980134Z",
  "log":"Server is listening on port 5000",
  "topic_name":"dy-7d8eeb3d4fb6da81d88da56280de7d42-5744dfc46c-dgsjf",
  "kubernetes": { ... }
}
```

Now Fluent Bit can safely use **topic_name** as the Topic_key.

### Step 4: Fluent Bit config

```yml
[FILTER]
    Name     lua
    Match    kube.*
    script   /fluent-bit/scripts/promote_pod_name.lua
    call     promote_pod_name

[OUTPUT]
    Name          kafka
    Match         *
    Brokers       kafka:9093
    Topics        kube
    Topic_key     topic_name
    Dynamic_topic on
    Format        json

promote_pod_name.lua: |
  function promote_pod_name(tag, timestamp, record)
      local k8s = record["kubernetes"]
      if k8s and k8s["pod_name"] then
          record["topic_name"] = k8s["pod_name"]
      else
          record["topic_name"] = "default-topic" -- fallback
      end
      return 1, timestamp, record
  end

```

Here's what a typical **ConfigMap configuration for Fluent Bit** for processing logs coming from kubernetes pods would look like:
```yml

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        multiline.parser  docker, cri
        Tag               kube.*
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On

    [FILTER]
        Name              kubernetes
        Match             kube.*
        Merge_Log         On
        Keep_Log          Off
        Kube_Tag_Prefix   kube.var.log.containers.
        Kube_URL          https://kubernetes.default.svc:443
        Kube_CA_File      /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File   /var/run/secrets/kubernetes.io/serviceaccount/token
        K8S-Logging.Exclude  On

    [FILTER]
        Name     lua
        Match    kube.*
        script   /fluent-bit/scripts/promote_pod_name.lua
        call     promote_pod_name

    [OUTPUT]
        Name          kafka
        Match         *
        Brokers       kafka:9093
        Topics        kube
        Topic_key     topic_name
        Dynamic_topic on
        Format        json

  promote_pod_name.lua: |
    function promote_pod_name(tag, timestamp, record)
        local k8s = record["kubernetes"]
        if k8s and k8s["pod_name"] then
            record["topic_name"] = k8s["pod_name"]
        else
            record["topic_name"] = "default-topic" -- fallback
        end
        return 1, timestamp, record
    end
```

This approach isn’t limited to `pod_name` — you can promote **any nested Kubernetes field or label** to a top-level field and use it as your Kafka topic key. This is useful for:
-	Splitting logs by namespace
-	Creating per-application Kafka topics
-	Enforcing multi-tenant log isolation
 
If you try this in your environment or run into edge cases, I’d love to hear from you! Feel free to connect with me on LinkedIn or drop me a message.
