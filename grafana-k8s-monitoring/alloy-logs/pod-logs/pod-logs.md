# Inside Grafana K8s Monitoring: Unlocking Pod-Level Observability with alloy-logs and podLogs

Modern Kubernetes clusters generate an *ocean of logs*. Every pod and container continuously emits signals about your applications’ internal state. Hidden in this firehose of logs lies truth about performance, errors, and user impact—but only if you can collect and analyze it effectively.

In the Grafana Kubernetes Monitoring stack, **`alloy-logs`** powers cluster-wide log collection pipelines. At its core is **`podLogs`**, the component that captures raw Kubernetes container logs and enriches them with valuable Kubernetes metadata.

Together, they transform simple standard output streams into a structured, queryable, cluster-wide source of truth.

***

## Why Pod-Level Logs Matter

When a Kubernetes application starts exhibiting unexpected behavior, the first place you look is the logs. A failing rollout, an init container that won’t start, or a pod stuck in CrashLoopBackoff—these are issues that metrics alone often can’t fully explain.

Pod-level logs provide:

- **Real-time insight into application behavior** across your entire cluster workload  
- **Powerful debugging capability** for startup failures, runtime exceptions, and crash loops  
- **Correlated troubleshooting** by combining logs with metrics and Kubernetes events  
- **Incident investigation context** to understand what each service was doing at critical moments  
- **Audit trails and compliance records** through long-term log retention  

The key advantage is having **unified visibility** across your entire cluster without the complexity of managing separate logging agents for each application.

***

## From Raw Logs to Enriched Observability

By default, Kubernetes stores container logs on each node under `/var/log/pods/`, with directories organized by namespace and pod UID. These are rotating files managed by the kubelet:

```bash
/var/log/pods/
├── namespace1_pod1_uid1/
│   ├── container1/0.log
│   └── container2/0.log
└── namespace2_pod2_uid2/
    └── app/0.log
```

Tailing these files can be useful in a pinch but is insufficient for cluster-wide search, filtering by app version, or troubleshooting thousands of pods simultaneously.

This is where `podLogs` steps in:

- It **discovers logs cluster-wide**—no sidecar containers or per-app configuration required  
- It **enriches each log line** with Kubernetes metadata like namespace, pod name, container, labels, annotations, node, and cluster info  
- It **streams logs into alloy-logs pipelines**, which then route logs to Grafana Loki, Grafana Cloud, or other backends  
- It handles cluster scale and churn gracefully, so pods restarting, rolling updates, or node failures won’t interrupt observability  

***

## What PodLogs Actually Add

`podLogs` doesn’t just capture the *what* (the log message). It provides the crucial *where* and *who*—making logs far more useful.

For example, instead of just:

```
Successfully processed user request ID: abc123 in 45ms
```

You get a structured, enriched log entry:

```json
{
  "timestamp": "2024-01-15T14:30:25.123456789Z",
  "namespace": "production",
  "pod": "web-app-7d4b9c8f6-xyz42",
  "container": "app",
  "node_name": "worker-node-1",
  "app_kubernetes_io_name": "web-service",
  "app_kubernetes_io_version": "v2.1.0",
  "message": "Successfully processed user request ID: abc123 in 45ms"
}
```

This extra context lets you:

- Filter logs by application: `{app_kubernetes_io_name="web-service"}`  
- Focus on a specific deployment environment: `{namespace="production", pod=~"web-app-.*"}`  
- Investigate node-specific issues: `{node_name="worker-node-1"}`  

It’s the difference between searching blindly in a haystack and being handed a neatly labeled toolbox.

***

## Enabling `podLogs`

With Grafana’s Kubernetes Monitoring Helm chart, enabling pod-level log collection requires only a minimal configuration addition to your `values.yaml`:

**Example:**

```yaml
podLogs:
  enabled: true
```

This deploys an `alloy-logs` DaemonSet that:

- Runs on every node in your cluster  
- Automatically discovers all pods and containers  
- Enriches logs with Kubernetes metadata  
- Routes logs through your observability pipeline  
- Handles log rotation and cleanup gracefully  

***

## Configuration Options

The `podLogs` feature offers flexible configuration to control how logs are collected, processed, and enriched.

### Core Settings

| Key                        | Type   | Default                              | Description                                               |
| -------------------------- | ------ | ------------------------------------ | --------------------------------------------------------- |
| `enabled`                  | bool   | `false`                              | Enable pod log collection                                 |
| `gatherMethod`             | string | `"volumes"`                          | Method for collecting logs (filesystem, API, etc.)       |
| `namespaces`               | object | `{}`                                 | Include/exclude namespace rules                           |
| `extraDiscoveryRules`      | list   | `[]`                                 | Advanced filtering based on pod labels/annotations       |
| `extraLogProcessingStages` | list   | `[]`                                 | Post-processing such as parsing, enrichment, sampling    |
| `secretFilter.enabled`     | bool   | `false`                              | Automatically mask secrets                                |
| `labelsToKeep`             | list   | `[]`                                 | Control which labels to retain to reduce cardinality     |
| `structuredMetadata`       | map    | `{}`                                 | Add Kubernetes metadata fields to logs                    |
| `jobLabel`                 | string | `"integrations/kubernetes/pod-logs"` | Label to tag the log stream                               |

***

### Pre-Scrape Processing

These settings control **how logs are discovered and collected before ingestion**.

#### Gather Method

The `gatherMethod` field determines how logs are collected from your cluster. Each method has different characteristics suited for different environments:
| Method                         | Description                                                      | Status       | Best Use Case                                               |
| ------------------------------ | ---------------------------------------------------------------- | ------------ | ----------------------------------------------------------- |
| `volumes`                      | Direct mount of `/var/log/pods` (default, most performant)       | Stable       | Standard Kubernetes with node access; high throughput       |
| `filelog`                      | File-based collection with advanced parsing                      | Experimental | For multiline logs or custom formats                        |
| `kubernetesApi`                | Collects logs via the Kubernetes API                             | Stable       | No node filesystem access; higher latency and overhead     |
| `OpenShiftClusterLogForwarder` | Uses OpenShift native log forwarding                             | Experimental | For OpenShift clusters leveraging native integrations       |

Example:

```yaml
podLogs:
  enabled: true
  gatherMethod: volumes
```

#### Namespace Filtering

The `namespaces` setting lets you define **inclusion/exclusion rules** to control which namespaces are scraped for logs. This is crucial to avoid collecting noisy system logs or unnecessary application logs.

**Example:**

```yaml
podLogs:
  namespaces:
    include:
      - production
      - staging
      - monitoring
    exclude:
      - kube-system
      - kube-public
```

#### Extra Discovery Rules

You can further refine log discovery with **extraDiscoveryRules**, using metadata such as pod annotations or names.

**Example:**

```yaml
podLogs:
  extraDiscoveryRules: |
    - action: keep
      regex: "true"
      source_labels: [__meta_kubernetes_pod_annotation_logs_enabled]
    - action: drop
      regex: "test.*"
      source_labels: [__meta_kubernetes_pod_name]
```

*In the above example, logs are scraped only from pods that have the annotation `logs_enabled: true`.*

**Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  annotations:
    logs_enabled: "true"
spec:
  containers:
    - name: app
      image: my-app:latest
```

***

### Post-Scrape Processing

These settings define what happens **after logs are collected**, such as parsing, enrichment, or security filtering.

* **`extraLogProcessingStages`** – Add log parsing, field extraction, sampling, or filtering.
* **`secretFilter`** – Automatically detect and mask sensitive values before logs are shipped downstream.

**Example:**

```yaml
podLogs:
  extraLogProcessingStages: |
    - json:
        expressions:
          level: level
          message: msg
          timestamp: ts
    - labels:
        level:
    - match:
        selector: '{level="debug"}'
        stages:
          - sampling:
              rate: 0.1

  secretFilter:
    enabled: true
    includeGeneric: true
    partialMask: 4
    replacement: "[REDACTED]"
```

***

### Controlling Labels and Metadata

These settings manage which labels and metadata are kept on logs to **avoid excessive cardinality** while preserving useful context.

* **`labelsToKeep`** – Explicitly list which labels to retain.
* **`structuredMetadata`** – Add Kubernetes metadata as structured fields for analysis.

```yaml
podLogs:
  labelsToKeep:
    - app.kubernetes.io/name
    - app.kubernetes.io/instance
    - deployment.environment
    - level

  structuredMetadata:
    k8s.pod.name: k8s.pod.name
    k8s.pod.uid: k8s.pod.uid
    k8s.pod.ip: k8s.pod.ip
```

***

## Complete Configuration Example

You can find the complete `values.yaml` configuration [here](https://github.com/varunpappu/articles/blob/main/grafana-k8s-monitoring/alloy-logs/pod-logs/values.yaml).

***

## Advanced Use Case: Custom Log Paths

Beyond standard pod logs, many enterprise environments require collecting application-specific logs from custom paths such as audit logs stored on persistent volumes, logs from sidecar containers, or legacy applications.

The `alloy-logs` module supports this with its **extraConfig** feature, allowing you to define custom collection pipelines.

### Real-World Example: HashiCorp Vault Audit Logs

Let's examine a use case where Vault Enterprise writes audit logs to CSI-mounted volumes. This configuration demonstrates the full power of custom log collection:

```yaml
alloy-logs:
  enabled: true
  controller:
    volumes:
      extra:
        - name: host-fs
          hostPath:
            path: /var/lib/kubelet/pods
            type: DirectoryOrCreate
  alloy:
    storagePath: /var/lib/alloy
    mounts:
      extra:
        - name: host-fs
          mountPath: /hostfs/var/lib/kubelet/pods
  # Custom log collection pipeline
  extraConfig: |
    declare "audit_logs" {
      argument "audit_logs_destinations" {
        comment = "Must be a list of log destinations where collected logs should be forwarded to"
      }

      // Step 1: Discover Vault pods in target namespace
      discovery.kubernetes "vault_pods" {
        role = "pod"
        namespaces {
          names = ["vault-enterprise"]
        }
      }

      // Step 2: Enrich pod discovery with custom labels and metadata
      discovery.relabel "filtered_vault_pods" {
        targets = discovery.kubernetes.vault_pods.targets
                
        rule {
          target_label = "job"
          replacement  = "vault-enterprise/hcvault"
          action       = "replace"
        }
                
        // Map pod labels and annotations for pipeline access
        rule {
          action = "labelmap"
          regex = "__meta_kubernetes_pod_label_(.+)"
        }

        rule {
          action = "labelmap"
          regex = "__meta_kubernetes_pod_annotation_(.+)"
        }
      }

      // Step 3: Construct file paths using pod UID
      discovery.relabel "vault_paths" {
        targets = discovery.relabel.filtered_vault_pods.output

        rule {
          source_labels = ["__meta_kubernetes_pod_uid"]
          target_label  = "__path__"
          // Complex glob pattern for CSI-mounted audit logs
          replacement   = "/hostfs/var/lib/kubelet/pods/$1/volumes/kubernetes.io~csi/**/mount/*audit*.log"
        }
      }

      // Step 4: Discover actual log files matching the patterns
      local.file_match "vault_audit" {
        path_targets = discovery.relabel.vault_paths.output      
      }

      // Step 5: Collect logs from discovered files
      loki.source.file "vault_logs" {
        targets    = local.file_match.vault_audit.targets
        forward_to = [loki.process.vault_logs.receiver]
      }

      // Step 6: Process collected logs
      loki.process "vault_logs" {
        
        // Rich structured metadata for enhanced searchability
        stage.structured_metadata {
          values = {
            "k8s_pod_name" = "k8s_pod_name",
            "k8s_pod_uid" = "k8s_pod_uid",
          }
        }

        // Curated label set for optimal performance
        stage.label_keep {
          values = [
            "app_kubernetes_io_name",
          ]
        }

        forward_to = argument.audit_logs_destinations.value
      }
    }

    // Step 7: Instantiate the audit log collector
    audit_logs "feature" {
      audit_logs_destinations = [
        loki.write.grafana_cloud_logs.receiver,
      ]
    }
```
### Step-wise Summary

- **Mount host paths** → Grants Alloy access to pod log directories (`/var/lib/kubelet/pods`) via `/hostfs`.  
- **Discover Vault pods** → Finds all pods in the `vault-enterprise` namespace.  
- **Enrich metadata** → Adds pod labels and annotations as log labels; sets a consistent job label.  
- **Build file paths** → Constructs CSI-mounted Vault audit log file paths using Pod UIDs.  
- **Match log files** → Expands glob patterns and finds the actual log files on disk.  
- **Collect logs** → Reads audit logs from the discovered files.  
- **Process logs** → Adds structured metadata (pod name, UID) and keeps only curated labels for performance.  
- **Forward logs** → Sends processed logs to Grafana Cloud Loki for centralized storage and analysis.  


***

## Wrapping Up

Pod-level logs are more than debugging artifacts—they’re the **narrative of your workloads**, written in real time. But without enrichment and scalability, this narrative is fragmented and hard to follow.  

Grafana’s `podLogs`, powered by `alloy-logs`, transforms raw logs into a **cluster-wide, structured source of truth**. With minimal configuration, you gain:  

- Unified visibility across every namespace and node  
- Fast, correlated troubleshooting with metrics and events  
- Structured, compliant logs for auditing and retention  

In short: you move from drowning in logs to **understanding your cluster’s story with clarity and confidence.**  

## References

* [podLogs `values.yaml`](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/charts/feature-pod-logs/values.yaml)
* [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
* [Grafana Alloy Processing Stages](https://grafana.com/docs/alloy/latest/reference/components/loki.process/)
* [Promtail Configuration](https://grafana.com/docs/loki/latest/clients/promtail/configuration/)