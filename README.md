# MicroK8s Kubernetes Cluster – Complete Setup, Ingress, Metrics Server & Headlamp Runbook

> **Purpose:** Complete operational documentation of the MicroK8s setup performed on the Ubuntu server, including cluster initialization, kubectl/Helm configuration, Metrics Server, Ingress/Traefik, host-port considerations, Headlamp deployment, custom `/headlamp` configuration, troubleshooting, and final validation.
>
> **Document status:** Based on the working setup and troubleshooting performed during this implementation.
>
> **Kubernetes version observed:** v1.35.6  
> **MicroK8s node:** `ip-172-31-8-60`  
> **Node IP:** `172.31.8.60`  
> **Headlamp version:** v0.44.0  
> **Headlamp Helm chart:** 0.44.0  
> **Namespace used for Headlamp:** `kube-system`  
> **Ingress hostname:** `uat.svkm.ac.in`  
> **Headlamp external path:** `/headlamp`

---

# 1. Final Architecture

The final setup is approximately:

```text
                         Client / Browser
                               |
                               | HTTPS
                               v
                   https://uat.svkm.ac.in/headlamp
                               |
                               v
                         +-----------+
                         |  Traefik  |
                         | Ingress   |
                         +-----+-----+
                               |
                               | HTTP
                               v
                    Service: my-headlamp
                    Namespace: kube-system
                    ClusterIP
                    Port: 80
                               |
                               | targetPort: http
                               v
                    Headlamp Pod :4466
                    Base URL: /headlamp
                               |
                               | In-cluster Kubernetes API
                               v
                     Kubernetes API Server
                         10.152.183.1:443
```

MicroK8s components:

```text
MicroK8s
├── Kubernetes API Server
├── containerd
├── Calico/CNI
├── DNS / CoreDNS
├── Metrics Server
├── cert-manager
├── Helm
├── Helm 3
└── Ingress / Traefik
```

---

# 2. Server Environment

The MicroK8s cluster was running on:

```text
Hostname:  ip-172-31-8-60
IP:       172.31.8.60
OS:       Ubuntu
Node:     ip-172-31-8-60
K8s:      v1.35.6
```

Check the environment:

```bash
hostname
hostname -I
cat /etc/os-release
uname -a
```

Check Kubernetes:

```bash
microk8s kubectl version
microk8s kubectl get nodes -o wide
```

---

# 3. MicroK8s Installation

MicroK8s was used as the Kubernetes distribution.

Install Snap if required:

```bash
sudo apt update
sudo apt install -y snapd
```

Install MicroK8s:

```bash
sudo snap install microk8s --classic
```

Add the current user to the MicroK8s group:

```bash
sudo usermod -a -G microk8s $USER
```

Refresh the shell/session after the group change:

```bash
newgrp microk8s
```

Create the kubeconfig directory if needed:

```bash
mkdir -p ~/.kube
chmod 700 ~/.kube
```

Check MicroK8s:

```bash
microk8s status
```

---

# 4. Verify MicroK8s

Basic status:

```bash
microk8s status
```

Expected state:

```text
microk8s is running
```

Check node:

```bash
microk8s kubectl get nodes
```

Observed working result:

```text
NAME             STATUS   ROLES    AGE   VERSION
ip-172-31-8-60   Ready    <none>   ...   v1.35.6
```

Detailed node information:

```bash
microk8s kubectl get nodes -o wide
```

Check all resources:

```bash
microk8s kubectl get all -A
```

---

# 5. MicroK8s Addons

The working MicroK8s environment had these enabled:

```text
cert-manager
dns
ha-cluster
helm
helm3
metrics-server
```

Check:

```bash
microk8s status
```

The important addons were:

```text
cert-manager
dns
helm
helm3
metrics-server
```

---

# 6. DNS Addon

Enable DNS:

```bash
microk8s enable dns
```

Verify:

```bash
microk8s status
```

Check CoreDNS:

```bash
microk8s kubectl get pods -n kube-system | grep -i dns
```

Check DNS service:

```bash
microk8s kubectl get svc -n kube-system | grep -i dns
```

Test DNS from a pod:

```bash
microk8s kubectl run dns-test \
  --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -- nslookup kubernetes.default
```

Expected result should resolve the Kubernetes service.

---

# 7. Metrics Server

Metrics Server was enabled using the MicroK8s addon.

Enable:

```bash
microk8s enable metrics-server
```

Verify:

```bash
microk8s status
```

Check the Metrics Server pod:

```bash
kubectl get pods -n kube-system | grep metrics
```

Check the service:

```bash
kubectl get svc -n kube-system | grep metrics
```

Check API registration:

```bash
kubectl get apiservice | grep metrics
```

Expected:

```text
v1beta1.metrics.k8s.io
```

Test node metrics:

```bash
kubectl top nodes
```

Test pod metrics:

```bash
kubectl top pods -A
```

If `kubectl top` initially reports that metrics are unavailable, inspect:

```bash
kubectl get pods -n kube-system | grep metrics
kubectl logs -n kube-system <metrics-server-pod>
kubectl describe apiservice v1beta1.metrics.k8s.io
```

---

# 8. Helm Installation

Helm was installed using Snap.

Initial command:

```bash
sudo snap install helm
```

Snap displayed:

```text
This revision of snap "helm" was published using classic confinement...
```

This is expected for the Helm Snap package.

Install with:

```bash
sudo snap install helm --classic
```

Verify:

```bash
which helm
helm version
```

Typically:

```text
/snap/bin/helm
```

---

# 9. kubectl vs MicroK8s kubectl

A major troubleshooting issue occurred here.

This worked:

```bash
microk8s kubectl get nodes
```

Result:

```text
NAME             STATUS   ROLES    AGE   VERSION
ip-172-31-8-60   Ready    <none>   ...   v1.35.6
```

But normal:

```bash
kubectl get nodes
```

returned:

```text
Error from server (NotFound): the server could not find the requested resource
```

The kubeconfig investigation showed:

```bash
echo "$KUBECONFIG"
```

returned empty.

Then:

```bash
kubectl config view --minify
```

returned:

```text
error: current-context must exist in order to minify
```

while:

```bash
microk8s kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}'
```

returned:

```text
https://127.0.0.1:16443
```

## Correct kubeconfig setup

Generate the MicroK8s kubeconfig:

```bash
mkdir -p ~/.kube
microk8s config > ~/.kube/config
chmod 600 ~/.kube/config
```

Then verify:

```bash
kubectl config current-context
```

Expected:

```text
microk8s
```

Check:

```bash
kubectl config get-contexts
```

Expected:

```text
CURRENT   NAME       CLUSTER            AUTHINFO
*         microk8s   microk8s-cluster   admin
```

Then:

```bash
kubectl get nodes
```

Now normal `kubectl` and MicroK8s kubectl use the same cluster.

---

# 10. Important kubectl Namespace Lesson

Kubernetes commands default to the `default` namespace unless `-n` is supplied.

For example:

```bash
kubectl logs pod/my-headlamp-xxxxx
```

looked in:

```text
namespace: default
```

and produced:

```text
pods "my-headlamp-xxxxx" not found in namespace "default"
```

The correct command was:

```bash
kubectl logs -n kube-system pod/my-headlamp-xxxxx
```

General rule:

```bash
kubectl get pods -n <namespace>
kubectl logs -n <namespace> <pod>
kubectl describe pod -n <namespace> <pod>
```

---

# 11. Ingress / Traefik

The environment used Traefik for ingress routing.

The intended routing was:

```text
https://uat.svkm.ac.in/headlamp
```

to:

```text
my-headlamp.kube-system.svc:80
```

The important concept is:

```text
External HTTPS
      |
      v
Traefik
      |
      | HTTP
      v
Headlamp Service :80
      |
      | targetPort
      v
Headlamp :4466
```

---

# 12. Headlamp Service

The final Headlamp Service was:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headlamp
  namespace: kube-system
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
  type: ClusterIP
```

The container listens on:

```text
4466/TCP
```

Therefore:

```text
Service port 80
       |
       v
Container port 4466
```

Check:

```bash
kubectl get svc my-headlamp -n kube-system
```

Detailed:

```bash
kubectl get svc my-headlamp -n kube-system -o yaml
```

Check endpoints:

```bash
kubectl get endpoints my-headlamp -n kube-system
```

Or:

```bash
kubectl get endpointslice -n kube-system \
  -l kubernetes.io/service-name=my-headlamp
```

---

# 13. Headlamp Installation

The old Kubernetes Dashboard Helm release was first removed:

```bash
helm uninstall kubernetes-dashboard -n kubernetes-dashboard
```

Headlamp repository:

```bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
```

Install:

```bash
helm install my-headlamp headlamp/headlamp \
  --namespace kube-system
```

Verify:

```bash
helm list -n kube-system
```

Expected release:

```text
my-headlamp
```

---

# 14. Headlamp Initial Configuration

The Headlamp container was configured to run in-cluster:

```text
-in-cluster
-in-cluster-context-name=main
```

Container:

```text
Image:
ghcr.io/headlamp-k8s/headlamp:v0.44.0

Port:
4466
```

The application uses the Kubernetes ServiceAccount:

```text
my-headlamp
```

Check:

```bash
kubectl get serviceaccount my-headlamp -n kube-system
```

Generate login token:

```bash
kubectl create token my-headlamp -n kube-system
```

Use the generated token in the Headlamp login page.

---

# 15. Headlamp Base URL Configuration

The requirement was to expose Headlamp at:

```text
https://uat.svkm.ac.in/headlamp
```

rather than using a separate root hostname.

Therefore Headlamp was configured with:

```text
Base URL:
/headlamp
```

The corresponding application argument was:

```text
-base-url=/headlamp
```

The final Helm configuration uses:

```yaml
config:
  inCluster: true
  inClusterContextName: "main"
  baseURL: "/headlamp"
  sessionTTL: 86400
```

---

# 16. Headlamp Ingress

The original Dashboard Ingress used Traefik annotations such as:

```yaml
traefik.ingress.kubernetes.io/service.serversscheme: https
```

That was appropriate for the Dashboard backend because the Dashboard service was HTTPS on port 443.

Headlamp is different.

Headlamp's Service is:

```text
my-headlamp:80
```

and its backend is HTTP.

Therefore Headlamp should use:

```yaml
traefik.ingress.kubernetes.io/service.serversscheme: http
```

Example Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: headlamp
  namespace: kube-system
  annotations:
    traefik.ingress.kubernetes.io/service.serversscheme: http
spec:
  ingressClassName: traefik
  rules:
    - host: uat.svkm.ac.in
      http:
        paths:
          - path: /headlamp
            pathType: Prefix
            backend:
              service:
                name: my-headlamp
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f headlamp-ingress.yaml
```

Verify:

```bash
kubectl get ingress -n kube-system
```

Describe:

```bash
kubectl describe ingress headlamp -n kube-system
```

---

# 17. Why Dashboard HTTPS Configuration Should NOT Be Reused

Dashboard:

```text
Backend:
HTTPS :443
```

Headlamp:

```text
Backend:
HTTP :80
```

Therefore:

### Dashboard

```yaml
traefik.ingress.kubernetes.io/service.serversscheme: https
```

### Headlamp

```yaml
traefik.ingress.kubernetes.io/service.serversscheme: http
```

Do not blindly copy the Dashboard transport configuration:

```text
dashboard-transport@kubernetescrd
```

to Headlamp.

---

# 18. Host Port / Port Binding Concept

A Kubernetes Service of type `ClusterIP` does not bind a host port.

Headlamp:

```text
Pod: 4466
Service: 80
Type: ClusterIP
```

means:

```text
Node host port
      |
      X
      |
Kubernetes Service
      |
      v
Pod :4466
```

There is no requirement to bind host port 4466.

This is preferable when using Traefik.

## Avoid unnecessary hostPort

Do not configure:

```yaml
ports:
  - containerPort: 4466
    hostPort: 4466
```

unless there is a specific requirement.

With Ingress, the correct model is:

```text
Internet
   |
Traefik host port
   |
ClusterIP Service
   |
Pod
```

rather than:

```text
Internet
   |
Node hostPort
   |
Headlamp Pod
```

---

# 19. Host Port Unbinding

If a pod was previously configured with a `hostPort`, changing the Deployment and removing the `hostPort` field is the correct way to unbind it.

Check:

```bash
kubectl get deployment my-headlamp -n kube-system -o yaml
```

Look under:

```yaml
containers:
- ports:
```

You should have:

```yaml
- containerPort: 4466
  name: http
  protocol: TCP
```

and NOT:

```yaml
hostPort: 4466
```

After changing the Deployment, Kubernetes creates a new ReplicaSet/pod without the host port.

Check:

```bash
kubectl get pods -n kube-system -o wide
```

---

# 20. Major Headlamp CrashLoopBackOff Troubleshooting

A major issue occurred after configuring:

```text
-base-url=/headlamp
```

The pod entered:

```text
CrashLoopBackOff
```

Observed:

```text
READY   STATUS
0/1     CrashLoopBackOff
```

Initial investigation:

```bash
kubectl logs -n kube-system my-headlamp-xxxxx --previous
```

The application itself started successfully:

```text
Listen address: :4466
Base URL: /headlamp
Use In Cluster: true
```

But Kubernetes events revealed the real cause:

```text
Liveness probe failed: HTTP probe failed with statuscode: 404
```

and:

```text
Container headlamp failed liveness probe, will be restarted
```

This was the root cause.

---

# 21. Understanding the CrashLoop

The sequence was:

```text
Headlamp starts
       |
       v
Listen on :4466
       |
       v
Base URL = /headlamp
       |
       v
Kubernetes liveness probe
       |
       v
HTTP 404
       |
       v
Kubernetes considers container unhealthy
       |
       v
Container killed
       |
       v
Container restarted
       |
       v
CrashLoopBackOff
```

Important lesson:

> A container can be completely functional while Kubernetes keeps restarting it because the health probe is incorrectly configured.

---

# 22. Temporary Diagnostic Patch

To prove the probes were causing the restart, the probes were temporarily removed:

```bash
kubectl patch deployment my-headlamp -n kube-system --type=json -p='[
  {"op":"remove","path":"/spec/template/spec/containers/0/livenessProbe"},
  {"op":"remove","path":"/spec/template/spec/containers/0/readinessProbe"}
]'
```

After this patch:

```text
Headlamp login page appeared.
```

This proved:

```text
Headlamp application: working
Ingress: working
Service: working
Probe configuration: problem
```

### Important

This was only a troubleshooting/diagnostic step.

Do not rely on a manual patch as the permanent configuration when the Deployment is Helm-managed.

---

# 23. Permanent Helm Fix

The Headlamp Helm chart values were inspected:

```bash
helm show values headlamp/headlamp
```

The relevant configuration was:

```yaml
config:
  inCluster: true
  inClusterContextName: "main"
  baseURL: ""

probes:
  scheme: HTTP
  livenessProbe:
    initialDelaySeconds: 0
    periodSeconds: 10
    timeoutSeconds: 1
    successThreshold: 1
    failureThreshold: 3

  readinessProbe:
    initialDelaySeconds: 0
    periodSeconds: 10
    timeoutSeconds: 1
    successThreshold: 1
    failureThreshold: 3
```

The permanent Helm configuration was applied using a values file:

```bash
helm upgrade my-headlamp headlamp/headlamp \
  -n kube-system \
  -f headlamp-values.yaml
```

Result:

```text
Release "my-headlamp" has been upgraded.
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
```

---

# 24. Final Probe Configuration

After the permanent Helm upgrade, the Deployment generated the correct probe paths:

```yaml
livenessProbe:
  httpGet:
    path: /headlamp/
    port: http
    scheme: HTTP

readinessProbe:
  httpGet:
    path: /headlamp/
    port: http
    scheme: HTTP
```

The critical detail is the trailing slash:

```text
/headlamp/
```

rather than:

```text
/headlamp
```

This solved the previous:

```text
404
```

probe failure.

---

# 25. Final Successful Headlamp Pod

After the Helm upgrade:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=headlamp -o wide
```

Result:

```text
NAME                         READY   STATUS    RESTARTS   AGE
my-headlamp-f787d768-gzqrv   1/1     Running   0          ...
```

Rollout:

```bash
kubectl rollout status deployment/my-headlamp -n kube-system
```

Result:

```text
deployment "my-headlamp" successfully rolled out
```

This confirms the permanent configuration works.

---

# 26. ReplicaSet Troubleshooting

During the failed rollout, multiple ReplicaSets existed:

```text
my-headlamp-759d4766bd
my-headlamp-7df7cd9bf8
my-headlamp-bb88f899
```

The old ReplicaSet had:

```text
1 desired
1 ready
```

The new ReplicaSet had:

```text
1 desired
0 ready
```

This indicated a rolling update where the new version could not become healthy.

Check ReplicaSets:

```bash
kubectl get rs -n kube-system | grep headlamp
```

Check rollout:

```bash
kubectl rollout status deployment/my-headlamp -n kube-system
```

Rollback if necessary:

```bash
kubectl rollout undo deployment/my-headlamp -n kube-system
```

Check revision history:

```bash
kubectl rollout history deployment/my-headlamp -n kube-system
```

---

# 27. Headlamp Logs

Current logs:

```bash
kubectl logs -n kube-system \
  deployment/my-headlamp \
  --tail=200
```

Specific pod:

```bash
kubectl logs -n kube-system <HEADLAMP_POD> --tail=200
```

Follow logs:

```bash
kubectl logs -n kube-system <HEADLAMP_POD> -f
```

Previous crashed container:

```bash
kubectl logs -n kube-system <HEADLAMP_POD> \
  --previous \
  --tail=300
```

The `--previous` option is particularly important for CrashLoopBackOff.

---

# 28. Kubernetes Events

Recent events:

```bash
kubectl get events -n kube-system \
  --sort-by='.lastTimestamp'
```

Last 30:

```bash
kubectl get events -n kube-system \
  --sort-by='.lastTimestamp' | tail -30
```

Headlamp-specific:

```bash
kubectl get events -n kube-system \
  --sort-by='.lastTimestamp' | grep -i headlamp
```

Useful event types:

```text
Scheduled
Pulled
Created
Started
Killing
Unhealthy
BackOff
SuccessfulCreate
SuccessfulDelete
ScalingReplicaSet
```

---

# 29. Pod Description

For complete troubleshooting:

```bash
kubectl describe pod -n kube-system <HEADLAMP_POD>
```

Look at:

```text
Conditions
Containers
State
Last State
Restart Count
Liveness
Readiness
Events
```

---

# 30. Check Container Termination

Useful command:

```bash
kubectl get pod <POD> -n kube-system \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

Check restart count:

```bash
kubectl get pod <POD> -n kube-system \
  -o jsonpath='{.status.containerStatuses[0].restartCount}'
echo
```

Typical reasons:

```text
Error
OOMKilled
Completed
```

---

# 31. Service Troubleshooting

Check service:

```bash
kubectl get svc my-headlamp -n kube-system
```

Detailed:

```bash
kubectl describe svc my-headlamp -n kube-system
```

Check endpoints:

```bash
kubectl get endpoints my-headlamp -n kube-system
```

Check EndpointSlices:

```bash
kubectl get endpointslice -n kube-system \
  -l kubernetes.io/service-name=my-headlamp
```

If there are no endpoints, check:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=headlamp
```

and compare Service selectors:

```bash
kubectl get svc my-headlamp -n kube-system -o yaml
```

with pod labels.

---

# 32. Test Headlamp Without Ingress

Use port forwarding:

```bash
kubectl port-forward \
  -n kube-system \
  svc/my-headlamp \
  8080:80
```

Then:

```text
http://127.0.0.1:8080/headlamp/
```

This bypasses Traefik.

If this works:

```text
Pod
  ↓
Service
  ↓
Headlamp
```

is working.

If external access fails, focus on:

```text
Ingress
Traefik
DNS
TLS
Firewall/Security Group
```

---

# 33. Ingress Troubleshooting

Check:

```bash
kubectl get ingress -A
```

Check Headlamp:

```bash
kubectl get ingress headlamp -n kube-system
```

Detailed:

```bash
kubectl describe ingress headlamp -n kube-system
```

Full configuration:

```bash
kubectl get ingress headlamp \
  -n kube-system \
  -o yaml
```

Check IngressClass:

```bash
kubectl get ingressclass
```

Confirm:

```text
traefik
```

is available if using:

```yaml
ingressClassName: traefik
```

---

# 34. Traefik Troubleshooting

Find Traefik:

```bash
kubectl get pods -A | grep -i traefik
```

Check services:

```bash
kubectl get svc -A | grep -i traefik
```

Logs:

```bash
kubectl logs -n <TRAEFIK_NAMESPACE> <TRAEFIK_POD> --tail=200
```

Follow:

```bash
kubectl logs -n <TRAEFIK_NAMESPACE> <TRAEFIK_POD> -f
```

When troubleshooting:

```text
Browser
   |
   v
Traefik
   |
   v
Ingress
   |
   v
Service
   |
   v
Pod
```

Determine the first layer where the request fails.

---

# 35. DNS Troubleshooting

Check hostname resolution:

```bash
nslookup uat.svkm.ac.in
```

or:

```bash
dig uat.svkm.ac.in
```

Check:

```bash
getent hosts uat.svkm.ac.in
```

Compare:

```bash
dig uat.svkm.ac.in
dig +short uat.svkm.ac.in
```

If DNS is private/internal, verify that the client is using the correct DNS resolver and that the DNS record points to the expected Traefik endpoint.

---

# 36. HTTP Troubleshooting

Test externally:

```bash
curl -I https://uat.svkm.ac.in/headlamp/
```

Detailed:

```bash
curl -vk https://uat.svkm.ac.in/headlamp/
```

This helps determine:

```text
DNS
TLS
HTTP status
redirect
server
```

Test HTTP if configured:

```bash
curl -v http://uat.svkm.ac.in/headlamp/
```

---

# 37. TLS Troubleshooting

Check certificate:

```bash
openssl s_client \
  -connect uat.svkm.ac.in:443 \
  -servername uat.svkm.ac.in
```

Look for:

```text
Certificate chain
Verify return code
TLS version
Cipher
```

For Traefik TLS problems, inspect:

```bash
kubectl describe ingress headlamp -n kube-system
```

and relevant TLS Secret:

```bash
kubectl get secret -n <namespace>
```

---

# 38. Headlamp Authentication

Generate a ServiceAccount token:

```bash
kubectl create token my-headlamp -n kube-system
```

The token is then supplied to the Headlamp login page.

Check ServiceAccount:

```bash
kubectl get serviceaccount my-headlamp -n kube-system
```

Check permissions:

```bash
kubectl get clusterrolebinding | grep my-headlamp
```

Check ClusterRoles:

```bash
kubectl get clusterrole | grep -i headlamp
```

For authorization failures, inspect:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:kube-system:my-headlamp
```

Example:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:kube-system:my-headlamp
```

---

# 39. Headlamp In-Cluster Kubernetes API

Headlamp logs showed:

```text
Use In Cluster: true
```

and:

```text
clusterURL: https://10.152.183.1:443
context: main
```

This means Headlamp is using the Kubernetes in-cluster configuration rather than requiring a kubeconfig file mounted into the pod.

This is the preferred approach for this deployment.

The messages:

```text
error loading kubeconfig files
open : no such file or directory
```

and:

```text
open /home/headlamp/.config/Headlamp/kubeconfigs/config:
no such file or directory
```

were related to external/dynamic kubeconfig loading and were not the cause of the CrashLoopBackOff.

The actual cause was the HTTP health probe returning 404.

---

# 40. Helm Management

Check repositories:

```bash
helm repo list
```

Update repository:

```bash
helm repo update
```

Check release:

```bash
helm list -n kube-system
```

Get values:

```bash
helm get values my-headlamp -n kube-system
```

Get all values:

```bash
helm get values my-headlamp -n kube-system --all
```

Get manifest:

```bash
helm get manifest my-headlamp -n kube-system
```

Check chart values:

```bash
helm show values headlamp/headlamp
```

Check release status:

```bash
helm status my-headlamp -n kube-system
```

Check history:

```bash
helm history my-headlamp -n kube-system
```

---

# 41. Helm Upgrade

Use a values file for persistent custom configuration:

```bash
helm upgrade my-headlamp headlamp/headlamp \
  -n kube-system \
  -f headlamp-values.yaml
```

Verify:

```bash
helm status my-headlamp -n kube-system
```

Then:

```bash
kubectl rollout status deployment/my-headlamp -n kube-system
```

Never rely on:

```bash
kubectl edit deployment ...
```

for Helm-managed configuration unless it is only a temporary diagnostic change.

---

# 42. Helm Rollback

If an upgrade causes a failure:

```bash
helm history my-headlamp -n kube-system
```

Rollback:

```bash
helm rollback my-headlamp <REVISION> -n kube-system
```

Verify:

```bash
kubectl rollout status deployment/my-headlamp -n kube-system
```

Then:

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=headlamp
```

---

# 43. Headlamp Uninstallation

Remove the Helm release:

```bash
helm uninstall my-headlamp -n kube-system
```

Verify:

```bash
kubectl get pods -n kube-system | grep -i headlamp
kubectl get svc -n kube-system | grep -i headlamp
kubectl get ingress -n kube-system | grep -i headlamp
```

If an Ingress was separately created:

```bash
kubectl delete ingress headlamp -n kube-system
```

Remove the repository if desired:

```bash
helm repo remove headlamp
```

---

# 44. Kubernetes Dashboard Removal

The previous Dashboard release was removed with:

```bash
helm uninstall kubernetes-dashboard \
  -n kubernetes-dashboard
```

Verify:

```bash
helm list -A
```

and:

```bash
kubectl get pods -A | grep -i dashboard
```

If Dashboard resources were installed separately using YAML, they must be removed using the corresponding manifest or documented installation method.

---

# 45. General Kubernetes Troubleshooting Workflow

When something is not working, use this order:

```text
1. Node
   |
   v
2. Pod
   |
   v
3. Container logs
   |
   v
4. Service
   |
   v
5. Endpoints
   |
   v
6. Ingress
   |
   v
7. Ingress Controller
   |
   v
8. DNS
   |
   v
9. TLS
   |
   v
10. External firewall / Security Group
```

Commands:

```bash
kubectl get nodes
kubectl get pods -A
kubectl describe pod -n <namespace> <pod>
kubectl logs -n <namespace> <pod>
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl get ingress -A
kubectl describe ingress -n <namespace> <ingress>
kubectl get ingressclass
kubectl get events -A --sort-by='.lastTimestamp'
```

---

# 46. CrashLoopBackOff Workflow

When:

```text
CrashLoopBackOff
```

do:

```bash
kubectl get pod -n <namespace> <pod>
```

Then:

```bash
kubectl describe pod -n <namespace> <pod>
```

Then:

```bash
kubectl logs -n <namespace> <pod>
```

Then:

```bash
kubectl logs -n <namespace> <pod> --previous
```

Then:

```bash
kubectl get events -n <namespace> \
  --sort-by='.lastTimestamp'
```

Check termination:

```bash
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

Common causes:

```text
Application crash
Bad environment variable
Bad configuration
Image problem
Permission problem
OOMKilled
Liveness probe failure
Readiness probe failure
Missing volume
Missing Secret
Missing ConfigMap
```

---

# 47. Probe Troubleshooting

When a pod keeps restarting:

```bash
kubectl describe pod -n <namespace> <pod>
```

Look for:

```text
Liveness probe failed
Readiness probe failed
```

Check the actual configured probe:

```bash
kubectl get deployment <deployment> -n <namespace> -o yaml
```

Test the endpoint manually if possible:

```bash
kubectl exec -n <namespace> <pod> -- \
  curl -i http://127.0.0.1:<port>/<path>
```

If the container restarts too quickly and `exec` returns:

```text
container not found
```

use:

```bash
kubectl logs --previous
```

or temporarily remove the probe for diagnosis.

Then restore the correct probe through the actual configuration management system, such as Helm.

---

# 48. Important Lesson From the Headlamp Incident

The Headlamp incident demonstrates an important Kubernetes principle:

```text
Application health != Kubernetes probe health
```

Headlamp was able to start and serve the login page, but Kubernetes considered it unhealthy because the configured HTTP probe received:

```text
404
```

Therefore:

```text
Application works
        +
Probe wrong
        =
CrashLoopBackOff
```

Always test the exact health endpoint used by Kubernetes.

---

# 49. Useful Resource Commands

All namespaces:

```bash
kubectl get pods -A
kubectl get svc -A
kubectl get deploy -A
kubectl get ingress -A
kubectl get events -A
```

Current namespace:

```bash
kubectl get all
```

Specific namespace:

```bash
kubectl get all -n kube-system
```

Wide output:

```bash
kubectl get pods -A -o wide
```

Labels:

```bash
kubectl get pods -n kube-system --show-labels
```

YAML:

```bash
kubectl get deployment my-headlamp -n kube-system -o yaml
```

JSON:

```bash
kubectl get pod <pod> -n <namespace> -o json
```

---

# 50. Final Validation Checklist

Run the following after changes.

## MicroK8s

```bash
microk8s status
```

Expected:

```text
microk8s is running
```

## Nodes

```bash
kubectl get nodes -o wide
```

Expected:

```text
Ready
```

## DNS

```bash
kubectl get pods -n kube-system | grep -i dns
```

## Metrics

```bash
kubectl top nodes
kubectl top pods -A
```

## Helm

```bash
helm version
helm list -A
```

## Headlamp

```bash
kubectl get pods -n kube-system \
  -l app.kubernetes.io/name=headlamp
```

Expected:

```text
1/1 Running
```

## Deployment

```bash
kubectl rollout status deployment/my-headlamp -n kube-system
```

Expected:

```text
successfully rolled out
```

## Service

```bash
kubectl get svc my-headlamp -n kube-system
```

Expected:

```text
80/TCP
```

## Endpoints

```bash
kubectl get endpoints my-headlamp -n kube-system
```

Expected:

```text
<POD_IP>:4466
```

## Ingress

```bash
kubectl get ingress headlamp -n kube-system
```

## Events

```bash
kubectl get events -n kube-system \
  --sort-by='.lastTimestamp' | grep -i headlamp
```

There should be no continuing:

```text
Liveness probe failed
Readiness probe failed
Back-off restarting failed container
```

## External access

```text
https://uat.svkm.ac.in/headlamp/
```

---

# 51. Useful One-Line Diagnostics

### Check Headlamp

```bash
kubectl get pods,svc,ingress -n kube-system | grep -i headlamp
```

### Check Headlamp logs

```bash
kubectl logs -n kube-system deployment/my-headlamp --tail=100
```

### Check recent Headlamp events

```bash
kubectl get events -n kube-system --sort-by='.lastTimestamp' | grep -i headlamp | tail -30
```

### Check Traefik

```bash
kubectl get pods -A | grep -i traefik
```

### Check Metrics Server

```bash
kubectl get pods -n kube-system | grep metrics
```

### Check cluster

```bash
kubectl get nodes && kubectl get pods -A
```

### Check Helm

```bash
helm list -A
```

---

# 52. Final Working Configuration Summary

```text
Kubernetes Distribution:
MicroK8s

Kubernetes:
v1.35.6

Node:
ip-172-31-8-60

Node IP:
172.31.8.60

MicroK8s API:
https://127.0.0.1:16443

Cluster Service API:
https://10.152.183.1:443

Enabled Addons:
- cert-manager
- dns
- helm
- helm3
- metrics-server

Headlamp:
v0.44.0

Helm release:
my-headlamp

Headlamp namespace:
kube-system

Headlamp Service:
my-headlamp

Service port:
80

Container port:
4466

Service type:
ClusterIP

Headlamp base URL:
/headlamp

External URL:
https://uat.svkm.ac.in/headlamp/

Ingress:
Traefik

Ingress class:
traefik

Backend protocol:
HTTP

Authentication:
Kubernetes ServiceAccount token

Token command:
kubectl create token my-headlamp -n kube-system

Probe path:
/headlamp/

Deployment status:
Successfully rolled out

Pod status:
1/1 Running
```

---

# 53. Key Lessons Learned

## 1. Always specify the namespace

Incorrect:

```bash
kubectl logs pod/my-headlamp
```

Correct:

```bash
kubectl logs -n kube-system pod/my-headlamp
```

## 2. `microk8s kubectl` and `kubectl` can use different configuration

If:

```bash
microk8s kubectl get nodes
```

works but:

```bash
kubectl get nodes
```

doesn't, check kubeconfig/context.

Generate:

```bash
microk8s config > ~/.kube/config
```

## 3. Helm Snap classic confinement is expected

Use:

```bash
sudo snap install helm --classic
```

## 4. Don't expose application pods with hostPort unnecessarily

Prefer:

```text
Ingress
  ↓
ClusterIP
  ↓
Pod
```

over:

```text
hostPort
  ↓
Pod
```

## 5. Don't manually edit Helm-managed resources permanently

Temporary:

```bash
kubectl patch
```

is useful for diagnosis.

Permanent:

```bash
helm upgrade ... -f values.yaml
```

## 6. Probe failures can cause CrashLoopBackOff

Always inspect:

```bash
kubectl get events
```

A pod can be healthy from an application perspective while Kubernetes keeps restarting it.

## 7. For path-based reverse proxying, configure the application's base URL

For:

```text
https://uat.svkm.ac.in/headlamp
```

Headlamp needs:

```text
baseURL=/headlamp
```

and the health probe must use a valid path:

```text
/headlamp/
```

## 8. Test each networking layer separately

```text
Pod
 ↓
Service
 ↓
Ingress
 ↓
Traefik
 ↓
DNS
 ↓
TLS
 ↓
Client
```

This avoids guessing and makes troubleshooting much faster.

---

# 54. Final Operational Command Set

For daily administration:

```bash
# Cluster
kubectl get nodes
kubectl get pods -A

# MicroK8s
microk8s status

# Metrics
kubectl top nodes
kubectl top pods -A

# Helm
helm list -A

# Headlamp
kubectl get pods -n kube-system -l app.kubernetes.io/name=headlamp
kubectl logs -n kube-system deployment/my-headlamp --tail=100

# Service
kubectl get svc my-headlamp -n kube-system
kubectl get endpoints my-headlamp -n kube-system

# Ingress
kubectl get ingress -n kube-system
kubectl describe ingress headlamp -n kube-system

# Events
kubectl get events -n kube-system --sort-by='.lastTimestamp'

# Rollout
kubectl rollout status deployment/my-headlamp -n kube-system

# Authentication
kubectl create token my-headlamp -n kube-system
```

---

# 55. Final State

The MicroK8s environment is successfully running with:

```text
                    MicroK8s
                       |
        +--------------+--------------+
        |              |              |
       DNS         Metrics Server   Helm
        |              |              |
        +--------------+--------------+
                       |
                    Traefik
                       |
                 Ingress /headlamp
                       |
                my-headlamp:80
                       |
                  Headlamp:4466
                       |
              Kubernetes API Server
```

Headlamp is exposed through the existing Traefik ingress at:

```text
https://uat.svkm.ac.in/headlamp/
```

The Headlamp deployment is Helm-managed, uses in-cluster Kubernetes authentication, uses `/headlamp` as its base URL, and has working liveness/readiness probes at `/headlamp/`.

The previous CrashLoopBackOff was caused by an incorrect health-probe path and was permanently resolved through the Helm-managed configuration.
