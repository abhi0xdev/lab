Intro -->
Sure. I'm Abhinandan, I have 4 years of experience working as a DevOps and SRE engineer at Cognizant, on a healthcare-domain platform running on Azure Kubernetes.
I joined as an intern in February 2022, moved to Junior DevOps Engineer within six months, and was promoted to DevOps Engineer in mid-2023 — so my growth has been within the same account, which gave me deep familiarity with the platform.
My work sits between DevOps and SRE. On the reliability side, I own our Prometheus and Grafana observability stack for an AKS environment running around 1,500 pods, I'm on the incident rotation handling P1 and P2 issues, and I do a lot of root-cause analysis using Dynatrace and Azure Monitor. On the delivery side, I support our GitHub Actions and Azure DevOps pipelines, contribute to Terraform infrastructure, and automate operational tasks with Bash and Python.
Outside of work I recently built a full observability platform from scratch called Athena — Prometheus, Loki, Tempo, OpenTelemetry, Grafana — with proper SLO-based alerting, mainly to deepen my hands-on building skills. It's on my GitHub.
I'm Microsoft AZ-400 certified, currently serving notice, and I'm looking for a DevOps or SRE role where I can go deeper on reliability and observability at scale

---

Day-to-Day Activities -->

My day breaks into three buckets.
First, observability — roughly 40% of my time. I start by reviewing Grafana dashboards and overnight alerts to check platform health. If alerts fired overnight, I investigate whether they were real or a tuning problem. I maintain 5-6 dashboards covering latency, error rates, pod health, and saturation, and I tune Prometheus alert rules to keep signal-to-noise high. I also use Dynatrace heavily for trace-level analysis of slow endpoints and error spikes.
Second, incident response — about 30%. When a P1 or P2 comes in, I do initial investigation — pulling logs from App Insights and Log Analytics, checking Kubernetes events, looking at Dynatrace traces. If it's something I can handle at the ops level — a crash loop from a bad probe, an image-pull issue, a node resource problem — I resolve it. If it's an application bug, I partner with the L3 engineering team and hand them a full diagnostic trail so they fix it faster.
Third, automation and infrastructure — about 30%. I write Bash and Python for operational tasks like node disk cleanup and image updates, I support our CI/CD pipelines when builds or releases fail, and I've contributed to our Terraform infrastructure during the migration from CloudFormation.
And I'm on an alternate-weekend on-call rotation as first responder for P1/P2 alerts

---

Project Workflow -->

At Cognizant I work on a healthcare platform running on AKS — around 86 deployments and 1,500 pods across multiple environments, supported by separate platform, infrastructure, dev, and QA teams.
The observability architecture has a few layers. Prometheus collects metrics — both Kubernetes-level like pod health and CPU/memory, and application metrics from the services. Grafana is our visualization layer, where I own dashboards for latency, error rates, and saturation. For deeper APM and distributed tracing we use Dynatrace, and for application logs and telemetry we use Azure Application Insights and Log Analytics, which integrate tightly with our Azure infra.
On alerting, we use Alertmanager with severity-based routing. When I joined, on-call was getting paged for non-actionable alerts — I spent significant effort tuning thresholds, adding inhibition rules, and aligning alerts to real user impact.
On the delivery side, CI/CD runs on GitHub Actions and Azure DevOps. The pipeline design is owned by senior engineers; I support the operational side — troubleshooting failures, image promotion, rollbacks. Infrastructure is moving from CloudFormation to Terraform, and I've contributed on the variable and environment-config side.
My deepest ownership is the observability layer and the operational reliability of the platform.

---

Athena workflow -->

Athena is an observability platform I built from scratch. It instruments three microservices in different languages — Python, Node, and Go — each emitting metrics, structured logs, and OpenTelemetry traces.
The data flow has three pillars. Metrics: each service exposes a /metrics endpoint, Prometheus discovers them via ServiceMonitor CRDs and scrapes every 15 seconds. Logs: Promtail runs as a DaemonSet, tails container stdout, adds Kubernetes labels, and ships to Loki. Traces: services export OTLP to an OpenTelemetry Collector that forwards to Tempo. Everything's unified in Grafana, so I can jump from a metric anomaly to a trace to the exact log line.
For alerting, I defined an SLO — 99.5% availability over 30 days — and implemented multi-window, multi-burn-rate alerts following the Google SRE workbook, routed through Alertmanager.
The most valuable part was deliberately breaking it — memory leaks, latency injection, dependency failures, error spikes, probe misconfigurations — and documenting the full root-cause investigation for each.

---

Story 1 — OOMKilled / memory issue -->

We had a backend API getting OOMKilled every few hours — pods would run fine, then get killed and restart. Error rate spiked on each restart. I checked Grafana and saw memory climbing on a steady slope regardless of traffic, which pointed to a leak, not load. I confirmed OOMKilled from the pod's last state — exit code 137. Then I correlated with App Insights logs and found one endpoint being hit repeatedly during the growth window. I handed the engineering team the memory trend, the trace data, and the suspect endpoint — they found an unbounded in-memory cache and fixed it within a day. Memory flattened out completely. What I took away was that handing off a full diagnostic trail, not just 'it's OOMing,' cut their investigation time in half.

---

Story 2 — Noisy alert cleanup -->

When I moved into the DevOps role, on-call was getting paged 8-10 times a weekend, and most of it wasn't actionable. I exported 90 days of alert history and categorized which alerts actually led to action. About 12 rules caused 70% of the noise. For each, I looked at the underlying Prometheus rule — some had thresholds on raw counts instead of rates, some were missing a 'for' clause so they fired on instantaneous spikes, some were on metrics that didn't impact users. I fixed thresholds, added 'for' durations, added inhibition rules, and removed non-actionable ones. Weekend pages dropped to 2-3, almost all actionable. The principle I internalized: every alert should require a human action — if it doesn't, it's a dashboard metric, not an alert

---

Story 3 — CrashLoopBackOff from probe -->

During a release, a service started CrashLoopBackOff right after deploy. I was on-call. I checked the Kubernetes events and saw repeated liveness probe failures. App Insights showed the service was starting but never reaching ready. The release had increased the DB connection pool size, so startup was now taking ~25 seconds, but the liveness probe had a 10-second initial delay — so the kubelet was killing the pod before it finished starting. I recommended increasing initialDelaySeconds and adding a startupProbe. They redeployed and it came up clean. The lesson — CrashLoopBackOff is a symptom with many causes, and probe timings need to keep up when startup logic changes.

---

PromQL queries (know these cold) -->

# Request rate (the R in RED method)
```
sum(rate(http_requests_total{app="checkout-api"}[5m])) by (status)
```
# Error rate as a ratio (the E in RED)
```
sum(rate(http_requests_total{app="checkout-api",status=~"5.."}[5m]))
/
sum(rate(http_requests_total{app="checkout-api"}[5m]))
```
# p99 latency (the D in RED) — histogram_quantile over a bucket
```
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{app="checkout-api"}[5m])) by (le)
)
```
# Memory usage vs limit — catches leaks before OOMKill
```
container_memory_working_set_bytes{pod=~"checkout-api.*"}
/ on(pod) kube_pod_container_resource_limits{resource="memory"}
```
# Pod restart rate — catches crash loops
```
increase(kube_pod_container_status_restarts_total{namespace="production"}[1h])
```
# Is the target up? (the value of the pull model)
```
up{app="checkout-api"}
```
---

Alerting rule YAML (burn-rate) -->
```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: checkout-api-slo
  namespace: observability
  labels:
    release: kube-prom-stack   # must match Helm release or Prometheus ignores it
spec:
  groups:
  - name: checkout-api-slo
    rules:
    # Recording rule: the SLI
    - record: sli:checkout_api_availability:ratio_rate5m
      expr: |
        sum(rate(http_requests_total{app="checkout-api",status!~"5.."}[5m]))
        / sum(rate(http_requests_total{app="checkout-api"}[5m]))

    # Fast-burn alert: pages when burning budget at 14.4x
    - alert: CheckoutAPI_ErrorBudget_FastBurn
      expr: |
        (1 - sli:checkout_api_availability:ratio_rate5m) > (14.4 * 0.005)
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "checkout-api burning error budget fast"
        description: "At this rate the 30-day budget exhausts in ~2 days"
```
---
Bash script (the disk cleanup — explain the safety) -->
```
#!/bin/bash
set -euo pipefail   # fail fast: exit on error, undefined var, or pipe failure

THRESHOLD=75
usage=$(df -h /var/lib | awk 'NR==2 {print $5}' | sed 's/%//')

if (( usage < THRESHOLD )); then
    echo "Disk at ${usage}%, below threshold. Nothing to do."
    exit 0
fi

echo "Disk at ${usage}%, cleaning up..."
```
# AKS uses containerd, so crictl not docker
```
crictl images --quiet | while read -r img; do
    if ! crictl ps -a --quiet --image "$img" | grep -q .; then
        crictl rmi "$img" >/dev/null 2>&1 || true
    fi
done
```
# Remove old compressed logs, truncate huge active ones (preserve file handles)
```
find /var/log -name "*.gz" -mtime +14 -delete 2>/dev/null || true
find /var/log -name "*.log" -size +500M -exec truncate -s 100M {} \; 2>/dev/null || true

final=$(df -h /var/lib | awk 'NR==2 {print $5}' | sed 's/%//')
echo "Cleanup done. Disk now at ${final}%"
```
---

Python script (the bit to explain: why Python over Bash) -->
```
#!/usr/bin/env python3
"""Summarize error patterns from exported App Insights logs."""
import json, re, sys
from collections import Counter

def normalize(msg: str) -> str:
    """Group similar errors by stripping IDs, GUIDs, timestamps."""
    msg = re.sub(r"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}", "<guid>", msg)
    msg = re.sub(r"\d{4}-\d{2}-\d{2}T[\d:.]+Z?", "<timestamp>", msg)
    msg = re.sub(r"\b\d{5,}\b", "<id>", msg)
    return msg.strip()[:200]

def main():
    with open(sys.argv[1]) as f:
        logs = json.load(f)

    errors = Counter()
    for entry in logs:
        if entry.get("severityLevel", "").lower() in ("error", "critical"):
            errors[normalize(entry.get("message", ""))] += 1

    total = sum(errors.values())
    print(f"Total errors: {total} | Unique patterns: {len(errors)}\n")
    for sig, count in errors.most_common(10):
        print(f"{count:>5} ({count/total*100:4.1f}%)  {sig}")

if __name__ == "__main__":
    main()
```
---

 Kubernetes YAML (Deployment with probes — explain the probe difference) -->
```
 apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: checkout-api
  template:
    metadata:
      labels:
        app: checkout-api
    spec:
      containers:
      - name: checkout-api
        image: myregistry.azurecr.io/checkout-api:v1.2.3
        ports:
        - containerPort: 8080
        resources:
          requests:          # scheduler uses this to place the pod
            cpu: 100m
            memory: 128Mi
          limits:            # exceed memory limit = OOMKilled
            cpu: 500m
            memory: 256Mi
        readinessProbe:      # gates traffic — fail = removed from Service endpoints
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:       # gates restart — fail = container killed & restarted
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```
  ---
