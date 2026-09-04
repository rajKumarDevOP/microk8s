# Kubernetes Observability Stack on MicroK8s

## Project Overview

This project implements a complete Kubernetes observability stack on a
**single-node MicroK8s cluster** running on Ubuntu/EC2.

The stack provides:

-   Kubernetes and node metrics using Prometheus
-   Linux host metrics using Node Exporter
-   Kubernetes object/state metrics using kube-state-metrics
-   Kubernetes/container logs using Promtail + Loki
-   Distributed tracing infrastructure using Tempo
-   Visualization and dashboards using Grafana
-   External access to Grafana through Traefik Ingress
-   Grafana hosted under a URL subpath: `/kubegrafana`
-   Coexistence with an existing host-level Prometheus already using TCP
    port `9100`

The implementation was deliberately configured so that the existing host
Prometheus remained untouched while the Kubernetes Node Exporter
continued to use its standard port `9100` inside the Pod network.

------------------------------------------------------------------------

## 1. Architecture

``` text
                         Users / Browser
                               |
                               | https://<domain>/kubegrafana/
                               v
                         +-------------+
                         |   Traefik   |
                         |  Ingress    |
                         +-------------+
                               |
                               | /kubegrafana
                               v
                         +-------------+
                         |   Grafana   |
                         |  Service    |
                         +-------------+
                               |
             +-----------------+------------------+
             |                 |                  |
             v                 v                  v
        Prometheus          Loki               Tempo
        Metrics             Logs               Traces
             ^                 ^                  ^
             |                 |                  |
       +-----+-----+           |            OpenTelemetry /
       |           |           |              app agents
       |           |           |
       |      kube-state-   Promtail
       |      metrics          |
       |           |           |
       +-----+-----+           |
             |                 |
             v                 v
       Kubernetes          Pod stdout/stderr
       API / Kubelet
             |
             v
       Node Exporter
             |
             v
       Linux / EC2 host
```

### Data flow

#### Metrics

``` text
Linux node
    |
    v
Node Exporter ----+
                  |
Kubernetes ------>| Prometheus ---> Grafana
kube-state-metrics+
Kubelet/cAdvisor -+
```

#### Logs

``` text
Application / Pod stdout & stderr
                |
                v
             Promtail
                |
                v
               Loki
                |
                v
             Grafana
```

#### Traces

``` text
Instrumented application
          |
          v
 OpenTelemetry / OTLP
          |
          v
        Tempo
          |
          v
       Grafana
```

> Important: deploying Tempo provides the trace backend, but
> applications still need OpenTelemetry instrumentation/export
> configuration before application traces appear.

------------------------------------------------------------------------

# 2. Environment

## Kubernetes platform

-   Kubernetes distribution: **MicroK8s**
-   Cluster topology: **Single node**
-   Operating system: **Ubuntu**
-   Infrastructure: **AWS EC2**
-   Container runtime: MicroK8s/containerd
-   Ingress controller: **Traefik**
-   Package manager: **Helm 3 via MicroK8s**

The cluster is currently single-node and therefore does not provide high
availability.

------------------------------------------------------------------------

# 3. MicroK8s Addons and Components

The MicroK8s environment has the following relevant components enabled:

``` text
dns
cert-manager
community
helm
helm3
metrics-server
traefik
observability
```

The `observability` addon was used to bootstrap the observability stack.

Important distinction:

-   MicroK8s addon = installation mechanism
-   Helm = package/release management mechanism
-   Kubernetes resources = the actual Deployments, StatefulSets,
    DaemonSets, Services, ConfigMaps, CRDs, etc.

The observability stack is managed through Helm releases in the
`observability` namespace.

------------------------------------------------------------------------

# 4. Observability Namespace

All observability components were deployed into:

``` text
observability
```

Check:

``` bash
kubectl get all -n observability
```

Useful resource checks:

``` bash
kubectl get pods -n observability
kubectl get svc -n observability
kubectl get deploy -n observability
kubectl get statefulset -n observability
kubectl get daemonset -n observability
```

------------------------------------------------------------------------

# 5. Helm Releases

The main Helm releases are:

``` text
kube-prom-stack
loki
tempo
```

Check them with:

``` bash
microk8s helm3 list -n observability
```

The Prometheus/Grafana stack uses:

``` text
Chart: kube-prometheus-stack
Chart version: 77.6.2
Application version: v0.85.0
```

The Loki installation uses:

``` text
Chart: loki-stack
Chart version: 2.10.3
```

The Tempo installation uses:

``` text
Chart: tempo
Chart version: 1.24.1
```

> Note: the `loki-stack` chart used by the MicroK8s observability addon
> is deprecated. This is a future improvement area; the current project
> documents the deployed stack as implemented.

------------------------------------------------------------------------

# 6. Components Deployed

## 6.1 Prometheus

Prometheus is responsible for collecting and storing time-series
metrics.

It collects metrics from sources including:

-   Node Exporter
-   kube-state-metrics
-   kubelet/cAdvisor
-   Kubernetes monitoring endpoints
-   ServiceMonitors/PodMonitors configured by the stack

Typical metrics include:

``` text
CPU
Memory
Filesystem
Network
Pod status
Container restarts
Deployment replicas
Kubernetes object state
```

Prometheus is deployed and managed as part of `kube-prometheus-stack`.

------------------------------------------------------------------------

## 6.2 Grafana

Grafana provides:

-   Metrics visualization
-   Kubernetes dashboards
-   Node dashboards
-   Log exploration
-   Trace exploration
-   Alert visualization

Grafana is exposed externally through Traefik under:

``` text
https://<domain>/kubegrafana/
```

The Grafana Kubernetes Service remains internal:

``` text
Service type: ClusterIP
Service port: 80
```

Check:

``` bash
kubectl get svc -n observability kube-prom-stack-grafana
```

------------------------------------------------------------------------

## 6.3 Node Exporter

Node Exporter exposes Linux host metrics such as:

``` text
CPU
Memory
Disk
Filesystem
Network
Load
```

The Node Exporter image currently used by the stack is:

``` text
quay.io/prometheus/node-exporter:v1.9.1
```

Normally Node Exporter listens on:

``` text
9100
```

### Important production issue encountered

The EC2 host already had a host-level Prometheus process listening on:

``` text
0.0.0.0:9100
```

The original Kubernetes Node Exporter configuration used:

``` yaml
hostNetwork: true
```

and therefore attempted to bind to the host's network namespace:

``` text
0.0.0.0:9100
```

This caused:

``` text
bind: address already in use
```

and the Node Exporter Pod entered:

``` text
CrashLoopBackOff
```

### Root cause

With:

``` yaml
hostNetwork: true
```

the Pod shares the host network namespace.

Therefore:

``` text
Host Prometheus
    |
    +---- :9100

Node Exporter
    |
    +---- :9100
```

caused a port conflict.

### Solution

The Helm values were changed to:

``` yaml
prometheus-node-exporter:
  hostNetwork: false
```

This allows Node Exporter to continue listening on:

``` text
Pod-IP:9100
```

without occupying:

``` text
Host-IP:9100
```

The existing host Prometheus therefore remains untouched.

### Final networking model

``` text
EC2 Host
|
+-- Host Prometheus
|      |
|      +-- 0.0.0.0:9100
|
+-- MicroK8s
       |
       +-- Node Exporter Pod
              |
              +-- Pod-IP:9100
```

The `containerPort: 9100` declaration is not a host port mapping. With
`hostNetwork: false`, the Pod's port is isolated from the host network.

------------------------------------------------------------------------

# 7. kube-state-metrics

`kube-state-metrics` exposes Kubernetes object/state information to
Prometheus.

Examples include:

``` text
Pod state
Deployment replicas
Container restart counts
DaemonSet state
StatefulSet state
Node state
Kubernetes object metadata
```

Example PromQL:

``` promql
kube_pod_info
```

Pod restart example:

``` promql
sum by (namespace, pod, container) (
  kube_pod_container_status_restarts_total
)
```

------------------------------------------------------------------------

# 8. Kubelet / Container Metrics

Container-level resource metrics are collected through
Kubernetes/Kubelet monitoring.

Useful metrics include:

``` text
container_cpu_usage_seconds_total
container_memory_working_set_bytes
container_network_receive_bytes_total
container_network_transmit_bytes_total
```

Example CPU query:

``` promql
sum by (namespace, pod) (
  rate(container_cpu_usage_seconds_total{
    container!="",
    container!="POD"
  }[5m])
)
```

Example memory query:

``` promql
sum by (namespace, pod) (
  container_memory_working_set_bytes{
    container!="",
    container!="POD"
  }
)
```

This is an important distinction:

> Node Exporter provides host/node metrics. Container CPU and memory
> metrics are obtained from Kubernetes/Kubelet/cAdvisor monitoring.

------------------------------------------------------------------------

# 9. Loki

Loki is the log aggregation backend.

It stores and indexes log labels while keeping the log content available
for querying.

The deployed Loki service is available internally at:

``` text
http://loki.observability.svc.cluster.local:3100
```

Check:

``` bash
kubectl get svc -n observability loki
```

------------------------------------------------------------------------

# 10. Promtail

Promtail is deployed as a DaemonSet.

Its purpose is to run on the Kubernetes node and collect container logs.

Data flow:

``` text
Container stdout/stderr
          |
          v
       Promtail
          |
          v
         Loki
```

Check:

``` bash
kubectl get daemonset -n observability
```

and:

``` bash
kubectl get pods -n observability
```

Promtail is currently running on the single MicroK8s node.

> Note: Promtail/Loki stack is the implementation deployed by the
> current MicroK8s observability addon. The `loki-stack` chart is
> deprecated, so migration to a current Loki deployment model is a
> future improvement.

------------------------------------------------------------------------

# 11. Tempo

Tempo provides the distributed tracing backend.

The deployed Tempo service is available inside the cluster.

Check:

``` bash
kubectl get svc -n observability tempo
```

Tempo is integrated with Grafana as a data source.

Important:

``` text
Tempo deployed
       !=
Application traces automatically available
```

Applications must be instrumented and configured to export traces,
commonly through OpenTelemetry.

------------------------------------------------------------------------

# 12. Grafana Data Sources

Grafana was configured with Loki and Tempo in addition to the Prometheus
data source.

The Helm values contain:

``` yaml
grafana:
  additionalDataSources:
    - name: loki
      type: loki
      url: http://loki.observability.svc.cluster.local:3100

    - name: tempo
      type: tempo
      url: http://tempo.observability.svc.cluster.local:3100
```

Therefore Grafana can query:

``` text
Prometheus -> Metrics
Loki       -> Logs
Tempo      -> Traces
```

------------------------------------------------------------------------

# 13. Grafana Subpath Configuration

Grafana was configured to run under:

``` text
/kubegrafana
```

rather than at the root of the domain.

The Grafana Helm configuration is:

``` yaml
grafana:
  grafana.ini:
    server:
      root_url: "%(protocol)s://%(domain)s/kubegrafana/"
      serve_from_sub_path: true
```

### Why this is required

Without `serve_from_sub_path: true`, Grafana may generate links and
static asset URLs relative to `/` instead of `/kubegrafana`.

The desired external URL is:

``` text
https://<domain>/kubegrafana/
```

------------------------------------------------------------------------

# 14. Traefik Ingress

Traefik is the Kubernetes Ingress controller.

Verify:

``` bash
kubectl get ingressclass
```

Expected:

``` text
NAME      CONTROLLER
traefik   traefik.io/ingress-controller
```

Grafana is exposed using:

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: observability
spec:
  ingressClassName: traefik
  rules:
    - http:
        paths:
          - path: /kubegrafana
            pathType: Prefix
            backend:
              service:
                name: kube-prom-stack-grafana
                port:
                  number: 80
```

The request path is intentionally preserved:

``` text
Browser
   |
   | /kubegrafana
   v
Traefik
   |
   | /kubegrafana
   v
Grafana Service
```

A StripPrefix middleware is not required because Grafana itself is
configured with:

``` yaml
serve_from_sub_path: true
```

------------------------------------------------------------------------

# 15. Complete Helm Values

The final custom values used for the observability configuration are:

``` yaml
grafana:
  additionalDataSources:
    - name: loki
      type: loki
      url: http://loki.observability.svc.cluster.local:3100

    - name: tempo
      type: tempo
      url: http://tempo.observability.svc.cluster.local:3100

  grafana.ini:
    server:
      root_url: "%(protocol)s://%(domain)s/kubegrafana/"
      serve_from_sub_path: true

kubeControllerManager:
  endpoints:
    - 172.31.8.60

kubeScheduler:
  endpoints:
    - 172.31.8.60

prometheus-node-exporter:
  hostNetwork: false
```

The controller-manager and scheduler endpoint configuration is specific
to the current single-node MicroK8s environment.

------------------------------------------------------------------------

# 16. Helm Upgrade Method

The existing release was upgraded using:

``` bash
microk8s helm3 upgrade kube-prom-stack \
  prometheus-community/kube-prometheus-stack \
  --version 77.6.2 \
  -n observability \
  --reuse-values \
  -f observability-values.yaml
```

### Command breakdown

``` text
microk8s
```

Uses the MicroK8s CLI.

``` text
helm3
```

Uses Helm 3 bundled with MicroK8s.

``` text
upgrade
```

Updates an existing Helm release.

``` text
kube-prom-stack
```

Existing Helm release name.

``` text
prometheus-community/kube-prometheus-stack
```

Helm repository + chart name.

``` text
--version 77.6.2
```

Pins the chart to the currently deployed version.

``` text
-n observability
```

Targets the `observability` namespace.

``` text
--reuse-values
```

Preserves values already stored in the existing Helm release.

``` text
-f observability-values.yaml
```

Applies the project's custom values file.

------------------------------------------------------------------------

# 17. Why `--reuse-values` Was Important

The existing Helm release already contained custom configuration such
as:

``` yaml
grafana:
  additionalDataSources:
    ...
```

and:

``` yaml
kubeControllerManager:
  endpoints:
    - 172.31.8.60
```

Using:

``` bash
--reuse-values
```

ensures those existing release values are retained while the new
configuration is applied.

Conceptually:

``` text
Existing Helm release values
              +
New values file
              |
              v
       Helm value merge
              |
              v
      Rendered manifests
              |
              v
        Kubernetes
```

------------------------------------------------------------------------

# 18. Validating Helm Changes Before Applying

Before the Node Exporter change was applied, the chart was rendered
with:

``` bash
microk8s helm3 template kube-prom-stack \
  prometheus-community/kube-prometheus-stack \
  --version 77.6.2 \
  -n observability \
  -f node-exporter-values.yaml
```

The rendered output was checked for:

``` yaml
hostNetwork: false
```

This validated the intended configuration before upgrading the live
release.

This is a useful operational pattern:

``` text
Edit values
    |
    v
helm template
    |
    v
Inspect rendered YAML
    |
    v
helm upgrade
    |
    v
Verify Kubernetes resources
```

------------------------------------------------------------------------

# 19. Verification Commands

## Check Helm releases

``` bash
microk8s helm3 list -n observability
```

## Check all resources

``` bash
kubectl get all -n observability
```

## Check Pods

``` bash
kubectl get pods -n observability
```

## Check Deployments

``` bash
kubectl get deployment -n observability
```

## Check StatefulSets

``` bash
kubectl get statefulset -n observability
```

## Check DaemonSets

``` bash
kubectl get daemonset -n observability
```

## Check services

``` bash
kubectl get svc -n observability
```

------------------------------------------------------------------------

# 20. Verify Node Exporter

``` bash
kubectl get pods -n observability \
  -l app.kubernetes.io/name=prometheus-node-exporter
```

Expected:

``` text
1/1   Running
```

Verify the network setting:

``` bash
kubectl get ds -n observability \
  kube-prom-stack-prometheus-node-exporter \
  -o jsonpath='{.spec.template.spec.hostNetwork}{"\n"}'
```

Expected:

``` text
false
```

Check logs:

``` bash
kubectl logs -n observability \
  -l app.kubernetes.io/name=prometheus-node-exporter
```

The previous:

``` text
bind: address already in use
```

error should no longer occur.

------------------------------------------------------------------------

# 21. Verify Prometheus

Check the Prometheus Pod:

``` bash
kubectl get pods -n observability \
  -l app.kubernetes.io/name=prometheus
```

Check the service:

``` bash
kubectl get svc -n observability | grep prometheus
```

A basic PromQL test:

``` promql
up
```

Container memory:

``` promql
container_memory_working_set_bytes
```

Container CPU:

``` promql
rate(container_cpu_usage_seconds_total[5m])
```

Pod information:

``` promql
kube_pod_info
```

------------------------------------------------------------------------

# 22. Verify Loki

Check:

``` bash
kubectl get pods -n observability | grep -E 'loki|promtail'
```

Both should be running.

In Grafana:

``` text
Explore
   |
   v
Loki
```

Start with:

``` logql
{}
```

Then narrow by Kubernetes labels if available:

``` logql
{namespace="observability"}
```

For Grafana logs, for example:

``` logql
{namespace="observability", pod=~".*grafana.*"}
```

The exact available labels should be confirmed from Grafana's Loki label
browser rather than assumed.

------------------------------------------------------------------------

# 23. Grafana Kubernetes Dashboards

The kube-prometheus-stack installation automatically provided Kubernetes
dashboards.

Important dashboards include:

``` text
Kubernetes / Compute Resources / Namespace (Workloads)
Kubernetes / Compute Resources / Node (Pods)
Kubernetes / Compute Resources / Pod
Kubernetes / Compute Resources / Workload

Kubernetes / Networking / Cluster
Kubernetes / Networking / Namespace (Pods)
Kubernetes / Networking / Namespace (Workload)
Kubernetes / Networking / Pod
Kubernetes / Networking / Workload

Kubernetes / Kubelet
Kubernetes / Controller Manager
Kubernetes / Scheduler
Kubernetes / Proxy
Kubernetes / Persistent Volumes

Node Exporter / Nodes
Node Exporter / USE Method / Cluster
Node Exporter / USE Method / Node

Prometheus / Overview
```

### Recommended dashboards

For Pod resource monitoring:

``` text
Kubernetes / Compute Resources / Pod
```

For workloads:

``` text
Kubernetes / Compute Resources / Workload
```

For namespace-level capacity:

``` text
Kubernetes / Compute Resources / Namespace (Workloads)
```

For Linux host monitoring:

``` text
Node Exporter / Nodes
```

For logs:

``` text
Grafana -> Explore -> Loki
```

For traces:

``` text
Grafana -> Explore -> Tempo
```

------------------------------------------------------------------------

# 24. Observability Responsibilities

  ------------------------------------------------------------------------
  Component               Responsibility          Data
  ----------------------- ----------------------- ------------------------
  Prometheus              Metrics                 Time-series metrics
                          collection/storage      

  Node Exporter           Linux host metrics      CPU, RAM, disk, network

  kube-state-metrics      Kubernetes object state Pods, Deployments,
                                                  Nodes, etc.

  Kubelet/cAdvisor        Container resource      Container
                          metrics                 CPU/memory/network

  Grafana                 Visualization           Metrics, logs, traces

  Promtail                Log collection          Container logs

  Loki                    Log storage/query       Kubernetes/application
                                                  logs

  Tempo                   Trace storage/query     Distributed traces

  Traefik                 Ingress                 External Grafana access
  ------------------------------------------------------------------------

------------------------------------------------------------------------

# 25. Troubleshooting Approach Used

The Node Exporter incident demonstrates the troubleshooting workflow
used in this project.

### Step 1 --- Identify the unhealthy Pod

``` bash
kubectl get pods -n observability
```

Node Exporter was:

``` text
CrashLoopBackOff
```

### Step 2 --- Inspect logs

``` bash
kubectl logs -n observability <node-exporter-pod>
```

The root cause was:

``` text
listen tcp 0.0.0.0:9100: bind: address already in use
```

### Step 3 --- Check host port ownership

``` bash
sudo lsof -i :9100
```

This confirmed an existing host Prometheus was already using port
`9100`.

### Step 4 --- Inspect generated Kubernetes configuration

``` bash
kubectl get ds -n observability \
  kube-prom-stack-prometheus-node-exporter \
  -o yaml
```

The original configuration contained:

``` yaml
hostNetwork: true
```

### Step 5 --- Inspect Helm values

``` bash
microk8s helm3 get values kube-prom-stack -n observability
```

The computed values were also inspected:

``` bash
microk8s helm3 get values kube-prom-stack \
  -n observability \
  -a > /tmp/kube-prom-values.yaml
```

### Step 6 --- Compare with chart defaults

The upstream chart default supports:

``` yaml
prometheus-node-exporter:
  hostNetwork: false
```

### Step 7 --- Render before applying

``` bash
microk8s helm3 template ...
```

The rendered manifest confirmed:

``` yaml
hostNetwork: false
```

### Step 8 --- Upgrade

``` bash
microk8s helm3 upgrade ...
```

### Step 9 --- Verify

``` bash
kubectl get pods -n observability
```

Node Exporter returned to:

``` text
1/1 Running
```

This is a reusable Kubernetes troubleshooting pattern:

``` text
Symptom
  |
  v
Pod status
  |
  v
Pod logs
  |
  v
Underlying resource/config
  |
  v
Host/network/resource verification
  |
  v
Helm values
  |
  v
Render
  |
  v
Apply
  |
  v
Verify
```

------------------------------------------------------------------------

# 26. Useful Kubernetes/Helm Commands

## Helm values

User-supplied values:

``` bash
microk8s helm3 get values kube-prom-stack \
  -n observability
```

All computed values:

``` bash
microk8s helm3 get values kube-prom-stack \
  -n observability \
  -a
```

Release history:

``` bash
microk8s helm3 history kube-prom-stack \
  -n observability
```

Release status:

``` bash
microk8s helm3 status kube-prom-stack \
  -n observability
```

## Inspect generated resources

``` bash
kubectl describe pod -n observability <pod-name>
```

``` bash
kubectl describe ds -n observability \
  kube-prom-stack-prometheus-node-exporter
```

``` bash
kubectl get ds -n observability \
  kube-prom-stack-prometheus-node-exporter \
  -o yaml
```

------------------------------------------------------------------------

# 27. Current Project State

The current implementation successfully provides:

-   [x] MicroK8s observability namespace
-   [x] Prometheus
-   [x] Grafana
-   [x] Alertmanager
-   [x] Node Exporter
-   [x] kube-state-metrics
-   [x] Loki
-   [x] Promtail
-   [x] Tempo
-   [x] Grafana Loki data source
-   [x] Grafana Tempo data source
-   [x] Kubernetes dashboards
-   [x] Node Exporter dashboards
-   [x] Traefik Ingress
-   [x] Grafana served under `/kubegrafana`
-   [x] Existing host Prometheus on port `9100` preserved
-   [x] Node Exporter/host Prometheus port conflict resolved

------------------------------------------------------------------------

# 28. Future Improvements

The following are logical next phases for turning this into a more
production-ready observability platform.

## Application metrics

Add application-level metrics such as:

``` text
HTTP request rate
HTTP latency
HTTP error rate
JVM metrics
Database connection pool
Business/application metrics
```

Expose them through Prometheus-compatible endpoints and configure
`ServiceMonitor` or `PodMonitor` resources.

## Application logs

Ensure application containers consistently write useful logs to:

``` text
stdout/stderr
```

and add appropriate Loki labels such as:

``` text
namespace
pod
container
application
environment
```

## Distributed tracing

Instrument applications using OpenTelemetry and export traces to Tempo.

## Alerting

Configure Prometheus/Alertmanager rules for:

``` text
High CPU
High memory
Pod CrashLoopBackOff
High restart count
Node disk pressure
Filesystem exhaustion
Pod availability
Application error rate
```

## TLS

Use the existing cert-manager integration to provide HTTPS certificates
for the Traefik Grafana Ingress.

## High availability

The current MicroK8s cluster is single-node.

A production HA implementation would require multiple Kubernetes nodes
and appropriate storage/networking architecture.

## Loki modernization

The currently deployed `loki-stack` chart is deprecated. A future
iteration should evaluate migration to a current Loki deployment
architecture.

------------------------------------------------------------------------

# 29. Project Demonstration Checklist

A project demonstration can be performed in this order:

### Kubernetes infrastructure

``` bash
microk8s status
kubectl get nodes
```

### Observability components

``` bash
kubectl get pods -n observability
```

### Helm releases

``` bash
microk8s helm3 list -n observability
```

### Node metrics

Open:

``` text
Grafana
 -> Dashboards
 -> Node Exporter / Nodes
```

### Kubernetes Pod metrics

Open:

``` text
Grafana
 -> Dashboards
 -> Kubernetes / Compute Resources / Pod
```

### Logs

Open:

``` text
Grafana
 -> Explore
 -> Loki
```

Run:

``` logql
{}
```

### Traces

Open:

``` text
Grafana
 -> Explore
 -> Tempo
```

### Grafana external access

Open:

``` text
https://<domain>/kubegrafana/
```

------------------------------------------------------------------------

# 30. Key Engineering Lessons

### 1. Container ports are not automatically host ports

``` yaml
containerPort: 9100
```

does not mean the application occupies host port `9100`.

The host conflict occurred because:

``` yaml
hostNetwork: true
```

made the Pod share the host network namespace.

### 2. Diagnose the actual error before changing ports

The correct solution was not to arbitrarily change Node Exporter from
`9100` to another port.

The better solution was to remove the unnecessary host network sharing:

``` yaml
hostNetwork: false
```

while keeping the standard Node Exporter port.

### 3. Helm values are the source of configuration

For Helm-managed Kubernetes applications, prefer changing the Helm
values rather than manually editing generated Deployments/DaemonSets.

### 4. Render before upgrading

Use:

``` bash
helm template
```

to inspect what Kubernetes manifests Helm will generate.

### 5. Preserve existing configuration

For an existing release, understand the difference between:

``` text
--reuse-values
```

and replacing values entirely.

### 6. Metrics, logs and traces are separate pipelines

``` text
Metrics -> Prometheus
Logs   -> Loki
Traces -> Tempo
```

Grafana is the visualization/query layer that brings them together.

------------------------------------------------------------------------

# 31. Final Architecture Summary

``` text
                    AWS EC2 / Ubuntu
                    Single MicroK8s Node
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
    Node Exporter     Kubelet/cAdvisor   kube-state-metrics
          |                |                |
          +----------------+----------------+
                           |
                           v
                       Prometheus
                           |
                           |
                           v
                       Grafana
                           ^
                           |
              +------------+------------+
              |                         |
              |                         |
             Loki                     Tempo
              ^                         ^
              |                         |
          Promtail                OpenTelemetry
              ^                         ^
              |                         |
        Pod stdout/stderr       Instrumented Apps


External access:

Browser
   |
   | https://<domain>/kubegrafana/
   v
Traefik Ingress
   |
   v
Grafana Service
   |
   v
Grafana Pod
```

The resulting platform provides a foundation for **Kubernetes
infrastructure monitoring, container monitoring, centralized logging,
visualization, and distributed tracing** while preserving the existing
host-level Prometheus deployment.
