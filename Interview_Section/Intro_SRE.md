Intro -->
Sure. I'm Abhinandan, I've been at Cognizant about 4 years now, working as a DevOps and SRE engineer on a healthcare platform that runs on Azure Kubernetes.
Most of my work is reliability — I look after our Prometheus and Grafana stack for an AKS environment with around 1,500 pods, I'm on the P1/P2 on-call rotation, and I do a lot of root-cause work with Dynatrace and Azure Monitor. I also support our CI/CD pipelines and automate the repetitive ops stuff with Bash and Python.
Outside work I built my own observability platform from scratch — Prometheus, Loki, Tempo, OpenTelemetry — just to get hands-on building one end to end. It's on my GitHub.
I'm AZ-400 certified and looking for an SRE or DevOps role where I can go deeper on reliability at scale

---

Day-to-Day Activities -->

Honestly it splits into about three things. The biggest chunk — maybe 40% — is observability. I usually start the day going through the Grafana dashboards and whatever alerts fired overnight, figuring out if something was actually wrong or if it's just an alert that needs tuning. I look after five or six dashboards — latency, error rates, pod health, that kind of thing — and I spend a fair bit of time tuning alert rules so on-call isn't drowning in noise. I lean on Dynatrace a lot too, especially for tracing down slow endpoints.
Then there's incident response, roughly 30%. When a P1 or P2 comes in, I do the first pass — pull logs from App Insights and Log Analytics, check the Kubernetes events, look at the traces. If it's something on the ops side, like a crash loop from a bad probe or an image-pull issue, I'll just fix it. If it turns out to be an actual application bug, I hand it to the L3 engineering team — but I make sure I give them the full picture so they're not starting from scratch.
The rest is automation and infra. I write Bash and Python for the repetitive stuff — disk cleanup on nodes, image updates — and I help out when our CI/CD pipelines break. I also pitched in on our Terraform migration off CloudFormation. And I'm on an alternate-weekend on-call rotation as first responder

---

Project Workflow -->

So the platform I work on is a healthcare app running on AKS — somewhere around 86 deployments and 1,500 pods across our environments, with separate teams for platform, infra, dev, and QA.
On the observability side, Prometheus handles the metrics — both the Kubernetes-level stuff like pod health and CPU, and the application metrics coming from the services themselves. Grafana sits on top for visualization, and that's where I own the dashboards. For the deeper tracing and APM we use Dynatrace, and for app logs we're on Azure App Insights and Log Analytics since they tie into our Azure setup nicely. Alerting goes through Alertmanager with severity-based routing — and that's an area I put real work into, because when I joined, on-call was getting paged for stuff that didn't matter. I spent a lot of time tuning thresholds and adding inhibition rules so the alerts actually mean something now.
On the delivery side, the pipelines run on GitHub Actions and Azure DevOps. The senior folks own the pipeline design — I handle the operational side, so troubleshooting failures, image promotion, rollbacks. And like I mentioned, I've chipped in on the Terraform migration on the config side.
But if you ask where I'm strongest — it's really the observability layer and keeping the platform reliable day to day

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
