# KEDA Autoscaling on OpenShift — Live Demo Kit

Event-driven autoscaling with [KEDA](https://keda.sh) across 5 real-world trigger types. Each scenario includes Kubernetes manifests and a live interactive dashboard (Vue 3 + Chart.js) to visualize scaling behavior in real time.

## What This Demonstrates

KEDA extends Kubernetes HPA with 60+ event sources and **scale-to-zero** — pods spin up only when work arrives. This repo contains 5 production-representative scenarios:

| # | Scenario | Trigger | Scale Range | Dashboard |
|---|----------|---------|-------------|-----------|
| 01 | **Kafka** — Consumer lag | `kafka` — lagThreshold per partition | 0 → 10 | Yes |
| 02 | **Database** — Pending rows | `postgresql` — SQL query result | 0 → 5 | Yes |
| 03 | **HTTP** — Request rate | `prometheus` — req/s from ServiceMonitor | 1 → 20 | Yes |
| 04 | **Prometheus** — Custom metric | `prometheus` — arbitrary PromQL | 1 → 6 | No |
| 05 | **Cron** — Scheduled scaling | `cron` — time-based rules | varies | No |

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  KEDA (Custom Metrics Autoscaler Operator)           │
│  Polls trigger sources → creates/updates HPA         │
└──────────┬──────────────────────────────┬────────────┘
           │                              │
     ScaledObject                   ScaledObject
     (Kafka lag)                    (SQL query)
           │                              │
    ┌──────▼──────┐               ┌───────▼──────┐
    │kafka-consumer│               │  db-worker   │
    │ Deployment   │               │  Deployment  │
    │ replicas: 0  │               │  replicas: 0 │
    └─────────────┘               └──────────────┘
```

KEDA manages the full lifecycle: **0 → N → 0**. It creates an HPA under the hood, but adds scale-to-zero semantics and external trigger support that native HPA doesn't provide.

## Prerequisites

- OpenShift 4.14+ (tested on 4.20)
- Red Hat Custom Metrics Autoscaler operator installed (provides KEDA v2.19.0)
- Strimzi/AMQ Streams operator (for Kafka scenario)
- PostgreSQL (deployed via `00-common/`)
- `oc` CLI authenticated to the cluster

## Quick Start

```bash
# Create namespace
oc new-project keda-demo

# Deploy shared infrastructure (Postgres)
oc apply -f 00-common/

# Deploy any scenario
oc apply -f 01-kafka/      # Kafka consumer lag scaling
oc apply -f 02-database/   # Database queue scaling
oc apply -f 03-http/       # HTTP request rate scaling
oc apply -f 04-prometheus/ # Custom Prometheus metric scaling
```

## Scenario Details

### 01 — Kafka: Scale on Consumer Group Lag

**Problem:** Messages pile up in a Kafka topic. Consumers need to scale out to keep up, and scale to zero when idle.

**How it works:**
- Producer sends messages to `keda-demo-orders` (10 partitions)
- KEDA monitors consumer group lag via Kafka's consumer group protocol
- When lag exceeds 50 messages/partition, consumers scale up (max 10 — one per partition)
- When lag clears, consumers scale back to zero after 15s cooldown

**Dashboard:** Interactive UI with produce buttons (+25/+50/+100/+500/+1000), per-partition lag breakdown, time-to-completion timer with run history, and live consumer pod sidebar.

```yaml
# Key ScaledObject config
triggers:
  - type: kafka
    metadata:
      lagThreshold: "50"
      topic: keda-demo-orders
      consumerGroup: keda-demo-consumer-group
```

### 02 — Database: Scale on Pending Queue Rows

**Problem:** Jobs accumulate in a Postgres table. Workers should process them and disappear when the queue is empty.

**How it works:**
- Dashboard seeds pending rows into a `jobs` table
- KEDA runs `SELECT COUNT(*) FROM jobs WHERE status = 'pending'` every 10s
- Workers scale 0 → 5 based on pending count (threshold: 5 rows per worker)
- Workers mark rows as `done`, then scale back to zero

**Dashboard:** Job queue stats (pending/processing/done), completion gauge, seed buttons, KEDA toggle.

### 03 — HTTP: Scale on Request Rate

**Problem:** An HTTP service gets overloaded. Need to scale replicas based on actual request throughput, not just CPU.

**How it works:**
- App exposes Prometheus metrics via `/metrics` endpoint
- ServiceMonitor scrapes into OpenShift monitoring stack
- KEDA queries `sum(rate(http_requests_total[1m]))` from Prometheus
- Pods scale 1 → 20 when request rate exceeds threshold

**Dashboard:** Load generator slider (0-50 threads), success rate gauge, error rate bar, req/s chart with threshold line.

### 04 — Prometheus: Scale on Any Custom Metric

**Problem:** You have a custom business metric (queue depth, batch size, GPU utilization) exposed to Prometheus. Scale on it.

**How it works:**
- App increments `http_requests_total` counter on each `/work` call
- KEDA queries any PromQL expression against the cluster's Thanos/Prometheus
- Uses bearer token auth via TriggerAuthentication

## Key KEDA Concepts Shown

| Concept | Where |
|---------|-------|
| **Scale to zero** | Kafka consumers, DB workers start at 0 replicas |
| **ScaledObject** | Every scenario — declarative scaling config |
| **TriggerAuthentication** | Prometheus (bearer token), PostgreSQL (secret ref) |
| **pollingInterval** | How often KEDA checks the trigger source (10s default) |
| **cooldownPeriod** | Delay before scaling down after metric drops (15-30s) |
| **Multiple trigger types** | Kafka, PostgreSQL, Prometheus, Cron across scenarios |

## File Structure

```
00-common/           # Shared infra (Postgres)
01-kafka/            # Kafka scaling + dashboard
  app-configmap.yaml #   Dashboard app (Python + Vue 3 + Chart.js)
  app-deployment.yaml#   Dashboard deployment + RBAC
  app-service.yaml   #   Service + Route
  consumer-deployment.yaml  # Kafka consumer (scale target)
  scaled-object.yaml #   KEDA ScaledObject
02-database/         # DB scaling + dashboard
03-http/             # HTTP scaling + dashboard
04-prometheus/       # Prometheus scaling
05-cron/             # Cron-based scaling
```

## Useful Commands

```bash
# Watch all ScaledObjects
oc get scaledobject -n keda-demo -w

# Check HPA created by KEDA
oc get hpa -n keda-demo

# Describe scaling decisions
oc describe scaledobject kafka-consumer-scaler -n keda-demo

# KEDA operator logs
oc logs -n openshift-keda deploy/keda-operator --tail=50

# Watch pods scale in real time
oc get pods -n keda-demo -w
```

## Cleanup

```bash
oc delete -f 01-kafka/ -f 02-database/ -f 03-http/ -f 04-prometheus/ -f 00-common/
oc delete kafkatopic keda-demo-orders -n lightwell-infra
oc delete project keda-demo
```

## References

- [KEDA Docs](https://keda.sh/docs/latest/)
- [KEDA Scalers](https://keda.sh/docs/latest/scalers/) — 60+ supported triggers
- [OpenShift Custom Metrics Autoscaler](https://docs.openshift.com/container-platform/latest/nodes/cma/nodes-cma-autoscaling-custom.html)
- [KEDA GitHub](https://github.com/kedacore/keda)
