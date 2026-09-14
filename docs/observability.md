# Observability

Outline only.
Each section says what it will cover and what state that part is in, so the gaps are visible rather than missing.

Sections marked **not built** describe something that does not exist yet.
They stay in this document on purpose: what we do not have is as relevant as what we do, particularly where MOC 1.0 had it.

## Where observability runs

The fleet layer is hub-central, the UI layer runs on both the hub and the workload clusters, because a dashboard queries whatever Thanos is local to its own cluster. **TBD**

## Metrics

### Collected in-cluster

User workload monitoring: what enabling it turns on, and where its configuration lives. **TBD**

### Reaching the hub

ACM MultiClusterObservability. **Not built**, blocked on object storage, tracked in [CCI-MOC/MOC-issues#467](https://github.com/CCI-MOC/MOC-issues/issues/467).
Described here so the gap is on the record. **TBD**

### The metric allowlist

MOC 1.0 filtered which metrics reached the hub through an `observability-metrics-custom-allowlist` ConfigMap.
There is no equivalent here, because the stack that reads it is not deployed. **TBD**

### Control plane metrics on hosted clusters

Why `kube_pod_resource_request` is not available to a hosted cluster, and what it would take.
Details in [CCI-MOC/MOC-issues#481](https://github.com/CCI-MOC/MOC-issues/issues/481). **TBD**

## Dashboards

Two tools, and which one a person sees depends on what they are allowed to see.

### Perses, through the Cluster Observability Operator

What is deployed, on which clusters, and how its datasource is wired.
The operator creates that datasource itself, which decides what a dashboard can show. **TBD**

### Grafana, one per project

A project sees its own metrics and not the cluster's.
Why this exists next to Perses rather than instead of it. **TBD**

### Who can see what

The RBAC that gates dashboard access. **TBD**

## Logs

**Not built.** There is no logging stack: no Loki, no log forwarder, no collectors.
MOC 1.0 centralised logs from five clusters into a LokiStack, with audit logs kept for 90 days and application and infrastructure logs for 30.
The backend needs object storage, the same blocker as MCO above.

This section exists to record that delta. **TBD**

## Traces

**Not built**, and not planned yet either.
No tracing stack exists here, no Tempo, no OpenTelemetry collector, and MOC 1.0 did not have one to compare against.

The [COO component list](https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html-single/about_red_hat_openshift_cluster_observability_operator/index#coo-about_cluster_observability_operator_overview) marks everything except Prometheus as optional, tracing included, so this is something we leave out rather than something we are missing.
Worth a section anyway, because the question comes up as soon as somebody debugs a request across services, and the answer today is simply no. **TBD**

## Correlating signals

Red Hat documents this as the [troubleshooting panel](https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html-single/ui_plugins_for_red_hat_openshift_cluster_observability_operator/index#coo-troubleshooting-ui-plugin-about_troubleshooting-ui-plugin) in the Cluster Observability Operator, powered by Korrel8r, GA for OpenShift 4.19 and later.
It is a navigation graph in the console between metrics, logs, alerts, netflows and resources, not a label join inside a dashboard.

Logs and traces in that graph need a LokiStack and a TempoStack respectively, and we have neither.
The panel itself does not, so it is installable here today, with two of its edges leading nowhere.
Metrics, alerts and resources would still correlate, so this is not all-or-nothing.

Perses does not correlate, it resolves datasources and draws dashboards. **TBD**

## Alerting

Prometheus evaluates rules, Alertmanager routes them, and dashboards only display.
Per-project separation for alerts uses the same mechanism as metrics rather than a second one.
Partly built. **TBD**

## Telemetry to Red Hat

What leaves a cluster, on what schedule, and how to turn it off.
Relevant for the NIST and HIPAA clusters, where the question is less about the data than about being able to answer what is sent. **TBD**

## How we use ACM

Design decisions rather than mechanics.
Whether policy auto-remediation is enabled, how ACM policy is configured if at all, and where ACM is deliberately not used for observability.

Owner: **TBD/@schwesig**, carried over from MOC 1.0.