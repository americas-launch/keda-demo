# KEDA Demo — 4 Scaling Scenarios

Demonstrates KEDA autoscaling on OpenShift across Kafka, PostgreSQL, HTTP (CPU), and Prometheus triggers.

**Cluster:** rosa-rwfjw (OCP 4.20)
**Namespace:** keda-demo
**KEDA:** Red Hat Custom Metrics Autoscaler v2.19.0 (openshift-keda)

---

## Prerequisites

Everything is already deployed. To verify:

```bash
oc get pods -n keda-demo
oc get scaledobject -n keda-demo
```

---

## Scenario 1: Kafka — Scale on Consumer Lag

Consumer pods scale from 0 when messages pile up in the `keda-demo-orders` topic.

```bash
# Start watching pods
oc get pods -n keda-demo -w &

# Produce 100 messages
oc delete job kafka-producer -n keda-demo --ignore-not-found
oc apply -f 01-kafka/producer-job.yaml

# Watch consumer pods appear and process messages
# After messages drain, consumers scale back to 0
```

**Trigger:** `kafka` — lagThreshold: 5 messages/partition
**Scale range:** 0 → 3

---

## Scenario 2: Database — Scale on Pending Rows

Worker pods scale from 0 when unprocessed rows appear in Postgres.

```bash
# Start watching pods
oc get pods -n keda-demo -w &

# Insert 50 pending rows
oc delete job db-seed -n keda-demo --ignore-not-found
oc apply -f 02-database/seed-job.yaml

# Watch worker pods appear and process rows
# After all rows are done, workers scale back to 0
```

**Trigger:** `postgresql` — query: `SELECT COUNT(*) FROM jobs WHERE status = 'pending'`, threshold: 5
**Scale range:** 0 → 5

---

## Scenario 3: HTTP — Scale on CPU Utilization

HTTP app pods scale up when CPU usage exceeds 50% due to traffic.

```bash
# Start watching pods
oc get pods -n keda-demo -w &

# Blast HTTP traffic
oc delete job http-load-generator -n keda-demo --ignore-not-found
oc apply -f 03-http/load-generator-job.yaml

# Watch http-app pods scale from 1 → N
# After load stops, pods scale back to 1
```

**Trigger:** `cpu` — Utilization threshold: 50%
**Scale range:** 1 → 8

---

## Scenario 4: Prometheus — Scale on Custom Metric

App pods scale when the `http_requests_total` rate exceeds threshold, queried from OpenShift Prometheus.

```bash
# Start watching pods
oc get pods -n keda-demo -w &

# Generate traffic to increment the metric
oc delete job prom-load-generator -n keda-demo --ignore-not-found
oc apply -f 04-prometheus/load-generator-job.yaml

# Watch prom-app pods scale from 1 → N
# After load stops and rate drops, pods scale back to 1
```

**Trigger:** `prometheus` — query: `sum(rate(http_requests_total{namespace="keda-demo"}[1m]))`, threshold: 10
**Scale range:** 1 → 6

---

## Useful Commands

```bash
# Check all ScaledObjects
oc get scaledobject -n keda-demo

# Describe a specific scaler
oc describe scaledobject kafka-consumer-scaler -n keda-demo

# Watch HPA created by KEDA
oc get hpa -n keda-demo

# Check KEDA operator logs
oc logs -n openshift-keda deploy/keda-operator --tail=50

# Reset DB scenario (clear processed rows)
oc exec deploy/postgres -n keda-demo -- psql -U kedauser -d kedadb -c "DELETE FROM jobs;"
```

---

## Cleanup

```bash
oc delete namespace keda-demo
oc delete kafkatopic keda-demo-orders -n lightwell-infra
```
