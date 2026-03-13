---
hide:
    - toc
---

# Why HPA Lags Behind Traffic Spikes by Design

> **TL;DR:** Kubernetes HPA has a minimum reaction time of ~90 seconds under ideal conditions. This isn't a bug or a misconfiguration — it's a fundamental property of a pull-based, backward-looking metrics pipeline. CPU is also the wrong signal for reactive autoscaling. Here's what's actually happening, and what to do about it.

---

## Table of Contents

1. [The Problem That Surprises Everyone](#1-the-problem-that-surprises-everyone)
2. [The Full Reaction Pipeline](#2-the-full-reaction-pipeline)
3. [The Smoothing Window Is Intentional](#3-the-smoothing-window-is-intentional)
4. [Why You Can't Tune Your Way Out of This](#4-why-you-cant-tune-your-way-out-of-this)
5. [CPU Is the Wrong Signal Entirely](#5-cpu-is-the-wrong-signal-entirely)
6. [KEDA: Different Causality, Not Just Faster Autoscaling](#6-keda-different-causality-not-just-faster-autoscaling)
7. [What Actually Works in Production](#7-what-actually-works-in-production)
8. [HPA vs. KEDA: Choosing the Right Tool](#8-hpa-vs-keda-choosing-the-right-tool)
9. [Configuration Reference](#9-configuration-reference)
10. [Summary](#10-summary)

---

## 1. The Problem That Surprises Everyone

You've set up HPA. You've tuned `targetCPUUtilizationPercentage`. You've stress-tested it in staging. Everything looks good.

Then production gets a sudden traffic spike — a viral post, a flash sale, a batch job triggering downstream APIs — and your pods get hammered. You watch the CPU metrics climb. You wait for pods to spin up. New pods finally appear... just as the spike subsides. Your users absorbed all the latency. The autoscaler created capacity for the calm that followed.

This scenario plays out constantly across engineering teams. And the frustrating part is that there's nothing obviously wrong. HPA was configured correctly. Kubernetes was behaving exactly as designed. The problem is architectural, not operational.

Understanding *why* this happens is the first step to building systems that actually handle spikes well.

---

## 2. The Full Reaction Pipeline

Before a single new pod can serve traffic, five sequential things have to happen. None of them are parallelised in the default setup. Each adds mandatory latency.

### Step 1: cAdvisor Scrapes Metrics (~15 seconds)

cAdvisor (Container Advisor) runs as part of the kubelet on every node. It collects resource usage metrics from all containers on that node — CPU, memory, network I/O, and disk. By default, it scrapes every **15 seconds**.

When a traffic spike hits at `t=0`, the earliest cAdvisor can observe elevated CPU is `t=15`. The data simply doesn't exist yet.

### Step 2: Metrics-Server Polls Kubelet (~60 seconds)

Metrics-server is a cluster-wide aggregator. It polls the kubelet (which surfaces cAdvisor data via the Summary API) to collect resource metrics across all nodes. Its default polling interval is **60 seconds**.

This is the biggest single contributor to HPA lag. Even if cAdvisor scrapes at `t=15`, metrics-server may not pick that data up until its next poll cycle — which could be nearly a minute away.

> **Note:** You can reduce this with `--metric-resolution=15s` on the metrics-server deployment. But you cannot reduce it to zero — it's still a polling loop.

### Step 3: HPA Controller Syncs (~15 seconds)

The HPA controller runs in the controller manager and periodically reconciles the desired replica count against the current metrics. Its sync period defaults to **15 seconds**, controlled by `--horizontal-pod-autoscaler-sync-period`.

At each sync, HPA queries the metrics API, calculates the desired replica count using its algorithm, applies stabilization windows, and (if warranted) issues a scale-up request to the Deployment or ReplicaSet.

### Step 4: Pod Scheduling, Image Pull, Startup, and Readiness (~30+ seconds)

Once HPA issues a scale-up, the new pods don't immediately serve traffic. They go through:

- **Scheduling:** The scheduler selects a node based on resource availability, affinity rules, taints/tolerations, and topology spread constraints. This is usually fast but depends on cluster state.
- **Image pull:** If the image isn't cached on the target node, it must be pulled from the registry. On a cold node with a 1GB image, this alone can take 30–60 seconds.
- **Container startup:** The runtime starts the container, environment variables are injected, init containers run.
- **Readiness probe:** The pod won't receive traffic until its readiness probe passes. Depending on your probe configuration — `initialDelaySeconds`, `periodSeconds`, `failureThreshold` — this can add meaningful latency.

**The math:**

```
15s (cAdvisor) + 60s (metrics-server) + 15s (HPA sync) + 30s (pod startup) = ~120s worst case
                                                                               ~90s best case
```

And that's under ideal conditions: warm image cache, available node capacity, no scheduling delays, fast readiness probes.

### The Implication

A traffic spike that peaks at `t=0` and dissipates by `t=2:00` may be **entirely over** before your autoscaler finishes reacting to it. The new pods come online just in time to serve the trickle of requests that follows the wave. Meanwhile, your existing pods were overloaded for the full duration of the spike.

---

## 3. The Smoothing Window Is Intentional

On top of the pipeline latency, HPA applies a stabilization window to prevent flapping.

### Scale-Up Window: 0 seconds (default)

HPA will scale up immediately when it detects sustained high utilization — no stabilization delay. This sounds reactive, but "sustained" is the key word. HPA averages metrics across multiple samples. A single elevated reading doesn't trigger a scale-out.

### Scale-Down Window: 300 seconds (5 minutes, default)

HPA waits 5 minutes of sustained low utilization before scaling down. This prevents an oscillation loop where pods are constantly being added and removed in response to noisy metrics.

### The Averaging Algorithm

HPA doesn't react to instantaneous metrics. It uses a ratio-based formula:

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))
```

And `currentMetricValue` is averaged across all pods in the target. A brief spike on one pod, averaged with stable metrics on other pods, may not cross the threshold that triggers a scale-out.

This smoothing is **correct behavior**. Without it, HPA would thrash on noisy CPU data, spinning pods up and down every minute. The averaging is protecting you from flap storms.

The uncomfortable truth is that both things are simultaneously true: **the smoothing is designed correctly, and it makes HPA unsuitable for spike handling**. These aren't in conflict. They're a consequence of the same design decision.

---

## 4. Why You Can't Tune Your Way Out of This

Engineers who discover this problem usually try to tune their way around it. These are the common approaches and why they have hard limits:

### Reduce metrics-server poll interval

```yaml
# metrics-server deployment args
- --metric-resolution=15s
```

This helps. You can cut the metrics-server delay from 60s down to 15s. But you've still got cAdvisor scraping at 15s, HPA syncing at 15s, and pod startup taking 30s+. Your floor drops from ~120s to ~75s. Real spikes are often shorter than that.

### Reduce HPA sync period

```yaml
# kube-controller-manager flag
--horizontal-pod-autoscaler-sync-period=5s
```

Again, helpful at the margins. But HPA can only act on data that exists. Syncing more frequently against stale metrics doesn't help.

### Reduce pod startup time

This is actually worth doing independently of HPA:

- Pre-pull images on nodes with DaemonSets or node pre-provisioning
- Optimize your readiness probe: lower `initialDelaySeconds`, ensure the probe is fast
- Use distroless or minimal base images to reduce pull size
- Consider startup probes to separate initialization from readiness

But even with a 10-second pod startup, you're still bounded by the metrics pipeline above it.

### The Fundamental Ceiling

You are working within a pull-based, backward-looking pipeline. Every component in the chain is reactive — it observes what already happened and reports it. No amount of tuning changes this architecture. There is a floor on how fast HPA can ever react, and for sudden traffic spikes, that floor is simply too high.

---

## 5. CPU Is the Wrong Signal Entirely

Here's the deeper problem: even if you could eliminate all pipeline latency, CPU would still be the wrong thing to measure.

### CPU Is a Lagging Indicator

When a traffic spike hits, the sequence of events is:

```
1. Requests arrive faster than pods can process them
2. Requests queue at the load balancer or in-pod queue
3. CPU starts climbing as pods work harder to drain the queue
4. Metrics-server observes elevated CPU
5. HPA scales out
6. New pods help drain the queue
```

By the time CPU is elevated, **users are already waiting**. The damage is already happening. You're not predicting load — you're confirming it after the fact.

### CPU Is a Noisy Signal

CPU utilization fluctuates constantly with GC pauses, connection setup overhead, background tasks, and JIT compilation (in JVM-based services). HPA's smoothing exists precisely because CPU is too noisy to act on directly. But the smoothing further delays reaction.

### CPU Tells You About Compute, Not About Demand

A Kafka consumer pod can have very low CPU while messages accumulate by the millions in its topic partition. A batch processor can be nearly idle while thousands of jobs queue. CPU measures how hard your code is working — not how much work is waiting to be done.

For stateless HTTP services with compute-intensive handlers, CPU is a reasonable proxy. For queue consumers, stream processors, or event-driven services, it's close to meaningless as a scaling signal.

---

## 6. KEDA: Different Causality, Not Just Faster Autoscaling

KEDA (Kubernetes Event-Driven Autoscaler) is often described as "HPA but faster." That framing misses the point.

KEDA's key insight isn't speed — it's **different causality**. Instead of measuring the consequence of load (CPU), it measures the *source* of load.

### Kafka Consumer Lag

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: order-processor
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: order-processors
      topic: orders
      lagThreshold: "100"     # Scale when lag exceeds 100 messages
      activationLagThreshold: "10"
```

When your Kafka consumer lag starts growing — messages accumulating faster than they're being processed — KEDA triggers a scale-out. This happens **before** CPU climbs. You're reacting to the cause, not the consequence.

If your lag threshold is 100 messages and your consumer processes 50 messages/second/pod, you're scaling before you're even 2 seconds behind. Compare that to waiting for CPU to climb over 5–10 seconds and then waiting 90 more seconds for the pipeline.

### SQS Queue Depth

```yaml
triggers:
- type: aws-sqs-queue
  metadata:
    queueURL: https://sqs.us-east-1.amazonaws.com/123/orders
    queueLength: "50"          # Scale when queue exceeds 50 messages
    awsRegion: us-east-1
```

Queue depth is a direct measure of pending work. When it grows, you need more workers — regardless of whether your existing workers are CPU-bound.

### Prometheus Custom Metrics

```yaml
triggers:
- type: prometheus
  metadata:
    serverAddress: http://prometheus:9090
    metricName: http_requests_pending
    threshold: "100"
    query: sum(http_requests_pending{service="api"})
```

This lets you scale on any metric your application exposes. Active connections, request queue depth, p99 latency, error rate — anything that meaningfully represents demand rather than resource consumption.

### KEDA + HPA: They're Complementary

KEDA actually creates an HPA object under the hood (you can see it with `kubectl get hpa`). It manages the HPA, injecting external metrics. So KEDA doesn't replace the scheduler/startup pipeline — pods still take 30+ seconds to be ready. What KEDA eliminates is the delay between "load is arriving" and "HPA is aware of it."

---

## 7. What Actually Works in Production

Based on the constraints above, here's a practical framework for different types of load.

### For Predictable Load: Cron-Based Scaling

If your traffic spikes at predictable times — 9am when the office opens, 6pm during peak consumer hours, Monday after a weekend — scale **before** the load arrives.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-cron-scaler
spec:
  scaleTargetRef:
    name: api-deployment
  cooldownPeriod: 300
  triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: "50 8 * * 1-5"    # Scale up at 8:50am weekdays
      end: "0 20 * * 1-5"      # Scale down at 8pm weekdays
      desiredReplicas: "20"
```

This approach has zero reaction time. You're not reacting to load — you're anticipating it. Cron-based scaling outperforms every reactive approach for known traffic patterns because it sidesteps the entire latency problem.

Pair this with your baseline HPA to handle unexpected load on top of the pre-scaled floor.

### For Queue and Stream Consumers: KEDA Lag Scaler

Running HPA on CPU for a Kafka consumer, SQS worker, or RabbitMQ subscriber is a category error. The signal that matters is queue depth and consumer lag — not how hard the consumer is working at this moment.

```yaml
# Kafka consumer with lag-based scaling
triggers:
- type: kafka
  metadata:
    bootstrapServers: kafka-broker:9092
    consumerGroup: payment-processors
    topic: payment-events
    lagThreshold: "50"
    offsetResetPolicy: latest

# SQS worker with queue depth scaling  
- type: aws-sqs-queue
  metadata:
    queueURL: https://sqs.us-east-1.amazonaws.com/account/payment-queue
    queueLength: "25"
    awsRegion: us-east-1
    scaleOnInFlight: "true"   # Include in-flight messages in the count
```

### For Stateless HTTP APIs: Combination Approach

For traditional HTTP services, the best production setup combines multiple layers:

**Layer 1: Higher minimum replicas during known traffic windows**

```yaml
# HPA with traffic-aware minimum
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 10      # Never drop below 10 during business hours
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

Scaling down is cheap. Running 10 pods when you only need 3 costs a small amount. Getting paged at 2am because 3 pods got hammered by an unexpected spike is much more expensive.

**Layer 2: KEDA for leading indicators**

If you have request queue depth metrics from your load balancer or service mesh, use them:

```yaml
triggers:
- type: prometheus
  metadata:
    serverAddress: http://prometheus:9090
    metricName: nginx_active_connections
    threshold: "200"
    query: |
      sum(nginx_ingress_controller_nginx_process_connections{
        state="active",
        ingress=~"api-.*"
      })
```

**Layer 3: Standard CPU-based HPA as the backstop**

Even with leading indicators, keep CPU-based HPA as a fallback. It handles the cases your other signals miss.

### Tune Your Stabilization Windows

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up immediately (no smoothing)
      policies:
      - type: Percent
        value: 100                        # Can double replica count in one cycle
        periodSeconds: 15
      - type: Pods
        value: 10                         # Or add 10 pods at a time
        periodSeconds: 15
      selectPolicy: Max                   # Use whichever is larger
    scaleDown:
      stabilizationWindowSeconds: 300     # Wait 5 minutes before scaling down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60                 # Remove at most 10% of pods per minute
```

The aggressive scale-up policy (`selectPolicy: Max`) is important for spikes — you want to add capacity as fast as possible when you do detect the need. The conservative scale-down prevents thrashing.

---

## 8. HPA vs. KEDA: Choosing the Right Tool

| Scenario | Recommended Approach | Why |
|---|---|---|
| Gradual, sustained CPU load | HPA with CPU target | Works exactly as designed |
| Kafka / RabbitMQ consumer | KEDA lag scaler | Queue depth is the real signal |
| SQS / SNS worker | KEDA queue depth | Work waiting ≠ CPU busy |
| Predictable daily traffic patterns | KEDA cron triggers | Zero reaction time |
| Flash traffic spikes | Higher `minReplicas` + cron pre-scale | Can't react fast enough reactively |
| HTTP API with Prometheus metrics | KEDA Prometheus + HPA backstop | Richer signals than CPU alone |
| Stateful workloads | Manual or custom controller | HPA/KEDA aren't designed for this |
| GPU workloads | Custom metrics on queue/batch depth | GPU utilization lags even more than CPU |

The rule of thumb: **HPA is the right tool for gradual, sustained load. It was never designed for spikes.** Knowing that distinction before your next incident is the whole point.

---

## 9. Configuration Reference

### Optimized metrics-server deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: metrics-server
        args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s          # Reduce from default 60s
        - --requestheader-client-ca-file=/etc/kubernetes/front-proxy-ca.crt
```

### HPA with custom behavior

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: production-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: production-api
  minReplicas: 5
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 5
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      selectPolicy: Min
```

### KEDA ScaledObject with multiple triggers

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: event-worker
  pollingInterval: 10         # Check triggers every 10 seconds
  cooldownPeriod: 60          # Wait 60s before scaling to zero
  minReplicaCount: 2          # Never fully scale to zero in production
  maxReplicaCount: 50
  advanced:
    restoreToOriginalReplicaCount: false
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0
        scaleDown:
          stabilizationWindowSeconds: 300
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: event-workers
      topic: events
      lagThreshold: "100"
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: queue_depth
      threshold: "50"
      query: sum(queue_depth{job="event-worker"})
```

### Pod startup optimization for faster readiness

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: api
        image: myapp:latest
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          failureThreshold: 30
          periodSeconds: 2          # Check every 2s during startup
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 0    # Start checking immediately after startup passes
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          failureThreshold: 3
      terminationGracePeriodSeconds: 60
```

---

## 10. Summary

The HPA lag problem has three root causes, each requiring a different solution:

**Root cause 1: Pipeline latency is structural, not configurable away.**
The cAdvisor → metrics-server → HPA → pod startup chain has a minimum latency of 60–90 seconds under best-case conditions. This is the cost of a pull-based, reactive metrics architecture. You can optimize each step, but you cannot eliminate the chain.

**Root cause 2: Smoothing is correct but means HPA reacts to trends, not spikes.**
The stabilization window and metric averaging that prevent flapping also mean HPA is fundamentally a trend-follower. A spike that lasts 2 minutes doesn't look like a trend — it looks like noise. That's by design.

**Root cause 3: CPU is a lagging indicator of demand.**
By the time CPU rises, users are already queueing. For anything other than compute-bound HTTP services, CPU measures the wrong thing entirely.

The practical implications:

- For **predictable load**: use cron-based pre-scaling to eliminate reaction time entirely
- For **queue/stream consumers**: use KEDA lag scalers, not CPU-based HPA
- For **unexpected spikes on HTTP APIs**: higher `minReplicas` is the cheapest and most reliable buffer
- For **gradual sustained load**: HPA with CPU is well-suited and works correctly
- For **everything production-critical**: combine layers — cron scaling + leading-indicator KEDA triggers + CPU HPA as a backstop

HPA is an excellent tool. It's just not a spike-handling tool. Knowing the difference before you need it is the whole point.
