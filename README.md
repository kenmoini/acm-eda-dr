# OpenShift Cluster DR with ACM and EDA

The built-in DR and application/cluster placement is very opinionated - sometimes you just want to "run a job" or "toggle something" to initiate a DR plan.  This can do just that.

## Prereqs

- 3 OpenShift cluster, an ACM Hub, a Primary, and Secondary site - technically you could use 2, leverage the ACM Hub as a secondary site, but really just get 3.
- A repo with some ArgoCD apps - maybe like this one
- Optionally a GKE cluster maybe if you wanna do weird things with that too
- ?????

## Concepts

### ArgoCD Application Cluster Label Targeting:

- App Grouping: `mario.apps.acme.org: deploy`
- App Cluster Placement: `target.mario.apps.acme.org: primary` #primary/secondary
- Active App Placement: `placement.mario.apps.acme.org: active` #(this toggles in the AAP job when a node goes down)

## Hub Setup

- Install the following Operators:
  - ACM
  - AAP 2.5 (cluster-scoped)
  - OpenShift GitOps
  - Cluster Observability Operator

- Deploy a MultiClusterHub CR in Basic availabilityMode:

```yaml
---
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
spec:
  availabilityConfig: Basic
```

- Deploy a basic AAP 2.5 Plaform:

```yaml
---
apiVersion: aap.ansible.com/v1alpha1
kind: AnsibleAutomationPlatform
metadata:
  name: aap
  namespace: aap
spec:
  api:
    log_level: INFO
    replicas: 1
  database:
    postgres_data_volume_init: false
  ingress_type: Route
  no_log: true
  redis_mode: standalone
  route_tls_termination_mechanism: Edge
  service_type: ClusterIP
  lightspeed:
    disabled: true
  hub:
    disabled: true
```

- [Import Primary and Secondary clusters into RHACM](https://github.com/kenmoini/rhacm-importer/blob/main/openshift-rhacm-importer.yaml)
- Apply a few labels to the clusters:
  - Primary Cluster:
    - `target.mario.apps.acme.org=primary`
    - `placement.mario.apps.acme.org=active`
  - Secondary Cluster:
    - `target.mario.apps.acme.org=secondary`
  - Both Clusters:
    - `mario.apps.acme.org=deploy`

- Enable ACM + GitOps integration: `oc apply -f rhacm-config/gitops-integration.yml`

> If you can't see the clusters in ArgoCD make sure you have admin access mapped and afterwards restart the dex pod

```yaml
---
kind: Group
apiVersion: user.openshift.io/v1
metadata:
  name: gitops-admins
users:
  - admin
  - kemo
---
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
spec:
  rbac:
    policy: |-
      g, system:cluster-admins, role:admin
      g, gitops-admins, role:admin
```

- Enable ACM O11y:

```bash
oc apply -f rhacm-config/mco/init.yml
oc apply -f rhacm-config/mco/job.yml

# Wait about 30s

oc apply -f rhacm-config/mco/mco.yml

# wait about 2min

oc apply -f rhacm-config/mco/thanos-rules.yml
oc apply -f rhacm-config/cco/uiplugin.yml
```

- Create some Placements: `oc apply -Rf rhacm-config/placements/`
- Create some PlacementBindings: `oc apply -Rf rhacm-config/placement-bindings/`
- Create some policies: `oc apply -Rf rhacm-config/policies/`

### ACM AlertManagerConfig

```yaml
"global":
  "resolve_timeout": "5m"
"receivers":
- "name": "null"
- name: "eda-policy-violation-processor"
  webhook_configs:
    - url: 'http://policy-violation-processor.aap.svc:8000/endpoint'
"route":
  "group_by":
  - "namespace"
  "group_interval": "5m"
  "group_wait": "30s"
  "receiver": "null"
  "repeat_interval": "12h"
  "routes":
  - "match":
      "alertname": "Watchdog"
    "receiver": "null"
  - receiver: "eda-policy-violation-processor"
    matchers:
      - "alertname = EDAViolatedPolicyReport"
      - "processor = eda"
```

---

> Don't look below here, scratch pad until I know what to do with things

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: observability-metrics-custom-allowlist
  namespace: open-cluster-management-observability
data:
  metrics_list.yaml: |
    names:
      - kube_persistentvolumeclaim_status_phase
  uwl_metrics_list.yaml: |
    names:
      - kube_persistentvolumeclaim_status_phase


```

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-stuck
  namespace: openshift-monitoring
  # If the label is not present, the alerting rule is deployed to Thanos Ruler
  # labels:
  #  openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus
spec:
  groups:
  - name: PodStuck
    rules:
    - alert: PodStuckContainerCreating
      annotations:
        summary: Pod stuck at ContainerCreating in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ContainerCreating
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ContainerCreating
      expr: kube_pod_container_status_waiting_reason{reason="ContainerCreating"}  == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: PodStuckImagePullBackOff
      annotations:
        summary: Cannot pull images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ImagePullBackOff
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ImagePullBackOff
      expr: kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: PodStuckInitImagePullBackOff
      annotations:
        summary: Cannot pull init images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at Init:ImagePullBackOff
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at Init:ImagePullBackOff
      expr: kube_pod_init_container_status_waiting_reason{reason="ImagePullBackOff"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: PodStuckErrImagePull
      annotations:
        summary: Cannot pull images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
      expr: kube_pod_container_status_waiting_reason{reason="ErrImagePull"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: PodStuckInitErrImagePull
      annotations:
        summary: Cannot pull images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
      expr: kube_pod_init_container_status_waiting_reason{reason="ErrImagePull"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: PodStuckCrashLoopBackOff
      annotations:
        summary: CrashLoopBackOff in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CrashLoopBackOff
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CrashLoopBackOff
      expr: kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1 
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: PodStuckCreateContainerError
      annotations:
        summary: Pod cannot created in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CreateContainerError
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CreateContainerError
      expr: kube_pod_container_status_waiting_reason{reason="CreateContainerError"} == 1 
      for: 2m
      labels:
        severity: critical
        processor: eda
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: leaf-pod-stuck
  namespace: openshift-monitoring
  # If the label is not present, the alerting rule is deployed to Thanos Ruler
  labels:
    openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus
spec:
  groups:
  - name: LeafPodStuck
    rules:
    - alert: LeafPodStuckContainerCreating
      annotations:
        summary: Pod stuck at ContainerCreating in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ContainerCreating
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ContainerCreating
      expr: kube_pod_container_status_waiting_reason{reason="ContainerCreating"}  == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: LeafPodStuckImagePullBackOff
      annotations:
        summary: Cannot pull images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ImagePullBackOff
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ImagePullBackOff
      expr: kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: LeafPodStuckInitImagePullBackOff
      annotations:
        summary: Cannot pull init images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at Init:ImagePullBackOff
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at Init:ImagePullBackOff
      expr: kube_pod_init_container_status_waiting_reason{reason="ImagePullBackOff"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: LeafPodStuckErrImagePull
      annotations:
        summary: Cannot pull images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
      expr: kube_pod_container_status_waiting_reason{reason="ErrImagePull"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: LeafPodStuckInitErrImagePull
      annotations:
        summary: Cannot pull images in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at ErrImagePull
      expr: kube_pod_init_container_status_waiting_reason{reason="ErrImagePull"} == 1
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: LeafPodStuckCrashLoopBackOff
      annotations:
        summary: CrashLoopBackOff in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CrashLoopBackOff
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CrashLoopBackOff
      expr: kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1 
      for: 2m
      labels:
        severity: critical
        processor: eda
    - alert: LeafPodStuckCreateContainerError
      annotations:
        summary: Pod cannot created in project {{ $labels.namespace }}
        message: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CreateContainerError
        description: Pod  {{ $labels.pod }}  in project {{ $labels.namespace }} project stuck at CreateContainerError
      expr: kube_pod_container_status_waiting_reason{reason="CreateContainerError"} == 1 
      for: 2m
      labels:
        severity: critical
        processor: eda
```