# Inside Grafana K8s Monitoring: Unlocking Pod-Level Observability with alloy-logs and podLogs

The `alloy-logs` module in Grafana k8s-Monitoring serves as the foundation for building comprehensive logging pipelines across your Kubernetes cluster. At the heart of this system lies `podLogs`, which provides deep visibility into your application workloads—capturing every log message from every container across your cluster.

For anyone managing Kubernetes workloads, pod-level logs are the primary source of truth for understanding application behavior, debugging failures, and monitoring performance in real-time. This comprehensive guide will walk you through everything you need to know about implementing robust pod-level observability at scale.

***

## Why Pod-Level Logs Matter

When a Kubernetes application starts exhibiting unexpected behavior, the first place you look is the logs. But collecting logs at scale across hundreds or thousands of pods requires a robust, efficient system that can handle high volume while preserving essential context.

By collecting logs directly from pod containers, `podLogs` allows you to:

* **Monitor application behavior and performance** across all workloads with real-time insights
* **Debug container failures** including startup issues, runtime errors, and crash loops
* **Correlate application logs** with infrastructure metrics and Kubernetes events for holistic troubleshooting
* **Investigate incidents** with full context about what each service was doing during critical moments
* **Maintain audit trails** for compliance and security analysis with complete log history

The key advantage is having **unified visibility** across your entire cluster without the complexity of managing separate logging agents for each application.

***

## Understanding Kubernetes Log Collection

Kubernetes stores container logs in a structured directory hierarchy on each node, typically under `/var/log/pods`. This standardized approach creates a predictable path structure that logging agents can leverage for efficient collection.

### Log Storage Architecture

```bash
/var/log/pods/
├── namespace1_pod1_uid1/
│   ├── container1/
│   │   └── 0.log
│   └── container2/
│       └── 0.log
└── namespace2_pod2_uid2/
    └── app/
        ├── 0.log
        └── 1.log
```

Each log file contains timestamped entries with container output, rotated automatically by the kubelet to prevent disk space exhaustion. The challenge lies in efficiently collecting these logs from across the cluster while preserving the rich Kubernetes metadata that makes them actionable for debugging and monitoring.

### Key Collection Challenges

* **Scale:** Handling thousands of pods generating high-volume logs
* **Metadata enrichment:** Adding Kubernetes context (namespace, pod name, labels, annotations)
* **Performance:** Minimizing resource overhead on nodes
* **Reliability:** Ensuring no log loss during pod restarts or node failures
* **Security:** Filtering sensitive information before shipping logs

***

## Log Structure: What Pod Logs Provide

Pod logs come enriched with extensive Kubernetes metadata, making them incredibly powerful for filtering, routing, and analysis. The `podLogs` collector automatically extracts and attaches this context to every log line.

### Core Metadata Fields

| Field                     | Description                                           | Example Value                          |
| ------------------------- | ----------------------------------------------------- | -------------------------------------- |
| `namespace`               | Kubernetes namespace containing the pod               | `production`                           |
| `pod`                     | Pod name generating the log                           | `web-app-7d4b9c8f6-xyz42`             |
| `container`               | Container name within the pod                         | `app`                                  |
| `node_name`               | Node where the pod is running                         | `worker-node-1`                        |
| `cluster`                 | Cluster identifier (if configured)                    | `production-us-east-1`                 |
| `job`                     | Job label for the log stream                          | `integrations/kubernetes/pod-logs`     |
| `__meta_kubernetes_pod_*` | Pod labels and annotations as discoverable metadata   | `app.kubernetes.io/name=web-service`   |
| `filename`                | Source log file path                                  | `/var/log/pods/.../app/0.log`          |

### Sample Enriched Log Entry

```json
{
  "timestamp": "2024-01-15T14:30:25.123456789Z",
  "namespace": "production",
  "pod": "web-app-7d4b9c8f6-xyz42",
  "container": "app",
  "node_name": "worker-node-1",
  "cluster": "production-us-east-1",
  "job": "integrations/kubernetes/pod-logs",
  "app_kubernetes_io_name": "web-service",
  "app_kubernetes_io_version": "v2.1.0",
  "deployment_environment": "production",
  "message": "Successfully processed user request ID: abc123 in 45ms"
}
```

### Manual Log Inspection

You can inspect logs manually on a node for debugging or validation:

```bash
# View logs for a specific pod
tail -f /var/log/pods/production_web-app-7d4b9c8f6-xyz42_*/app/*.log

# Check log rotation files
ls -la /var/log/pods/production_web-app-7d4b9c8f6-xyz42_*/app/
```

This metadata-rich approach enables powerful querying patterns in Grafana, such as:
- Filter by application: `{app_kubernetes_io_name="web-service"}`
- Debug specific deployments: `{pod=~"web-app-.*", namespace="production"}`
- Monitor node-specific issues: `{node_name="worker-node-1"}`

***

## Enabling `podLogs`

To enable pod-level log collection in your Grafana k8s-Monitoring Helm chart, add the following minimal configuration to your values file:

```yaml
podLogs:
  enabled: true
```

This simple configuration deploys an `alloy-logs` DaemonSet that:
- Runs on every node in your cluster
- Automatically discovers all pods and containers
- Enriches logs with Kubernetes metadata
- Routes logs through your configured observability pipeline
- Handles log rotation and cleanup

***

## Configuration Options

The **podLogs** configuration provides multiple core settings that define how logs are collected, processed, and enriched.

### Core Settings

| Key                        | Type   | Default                              | Description                                               |
| -------------------------- | ------ | ------------------------------------ | --------------------------------------------------------- |
| `enabled`                  | bool   | `false`                              | Enable pod log collection                                 |
| `gatherMethod`             | string | `"volumes"`                          | Method for collecting pod logs (filesystem, API, etc.)    |
| `namespaces`               | object | `{}`                                 | Namespace inclusion/exclusion rules                       |
| `extraDiscoveryRules`      | list   | `[]`                                 | Advanced filtering based on pod labels/annotations        |
| `extraLogProcessingStages` | list   | `[]`                                 | Custom parsing, enrichment, sampling, or dropping of logs |
| `secretFilter.enabled`     | bool   | `false`                              | Enable automatic secret masking                           |
| `labelsToKeep`             | list   | `[]`                                 | Preserve only specific labels to reduce cardinality       |
| `structuredMetadata`       | map    | `{}`                                 | Enrich logs with derived fields from Kubernetes metadata  |
| `jobLabel`                 | string | `"integrations/kubernetes/pod-logs"` | Label to tag this log stream                              |

---

### Pre-Scrape Processing

These settings control **how logs are discovered and collected before ingestion**.

#### Gather Method

The `gatherMethod` field determines how logs are collected from your cluster. Each method has different characteristics suited for different environments:

| Method                         | Description                                                      | Status       | Best Used When                                                                                          |
| ------------------------------ | ---------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------- |
| `volumes`                      | Direct mount of `/var/log/pods` (default, most performant)       | Stable       | Standard Kubernetes clusters with node access. Best for production workloads requiring high throughput. |
| `filelog`                      | File-based collection with advanced parsing capabilities         | Experimental | Useful if you need multiline parsing or custom log formats.                                             |
| `kubernetesApi`                | Collects logs via Kubernetes API (higher latency, more overhead) | Stable       | When node filesystem access isn’t available.                                                            |
| `OpenShiftClusterLogForwarder` | Uses OpenShift’s native log forwarding                           | Experimental | For OpenShift clusters leveraging native integrations.                                                  |

**Example:**

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
      - "production"
      - "staging"
      - "monitoring"
    exclude:
      - "kube-system"
      - "kube-public"
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

---

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

---

### Controlling Labels and Metadata

These settings manage which labels and metadata are kept on logs to **avoid excessive cardinality** while preserving useful context.

* **`labelsToKeep`** – Explicitly list which labels to retain.
* **`structuredMetadata`** – Add Kubernetes metadata as structured fields for analysis.

**Example:**

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

---

## Complete Configuration Example

```yaml
podLogs:
  enabled: true
  gatherMethod: volumes

  namespaces:
    include:
      - "production"
      - "staging"
    exclude:
      - "kube-system"
      - "kube-public"

  extraDiscoveryRules: |
    - action: keep
      regex: "true"
      source_labels: [__meta_kubernetes_pod_annotation_logs_enabled]
    - action: drop
      regex: "test.*"
      source_labels: [__meta_kubernetes_pod_name]

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

## Advanced Use Case: Custom Log Paths with Extra Configuration

Beyond standard pod logs, enterprise environments often require collecting application-specific logs from custom file paths. This includes audit logs written to persistent volumes, sidecar containers with shared directories, or legacy applications with established file-based logging patterns.

The `alloy-logs` module's **extraConfig** feature provides the flexibility to handle these complex scenarios through custom collection pipelines.

### Enterprise Logging Scenarios

**Common custom log collection requirements:**

- **Audit logs** written to persistent volumes or CSI storage for compliance
- **Application-specific logs** stored in custom directories with special naming conventions  
- **Sidecar containers** writing logs to shared volumes for centralized collection
- **Legacy applications** with established file-based logging that can't be easily containerized
- **Security logs** requiring special handling, parsing, or retention policies
- **Multi-tenant applications** with tenant-specific log paths and processing requirements


### Real-World Example: HashiCorp Vault Audit Logs

Let's examine a production use case where Vault Enterprise writes audit logs to CSI-mounted volumes. This configuration demonstrates the full power of custom log collection:

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


## Wrapping Up

To maximize the value of `podLogs`, follow these best practices:

- **Start selective:** Begin with critical namespaces and expand gradually
- **Monitor cardinality:** Watch label combinations to avoid high-cardinality issues  
- **Use structured logs:** JSON-formatted logs work best with processing stages
- **Implement sampling:** Control volume from high-throughput applications
- **Secure sensitive data:** Always enable secret filtering for production workloads

`podLogs` provides the comprehensive application visibility that forms the backbone of effective Kubernetes observability. By capturing every log message with rich Kubernetes context, it enables rapid debugging, thorough monitoring, and deep insights into how your workloads behave across the entire cluster lifecycle.

You can find the complete `values.yaml` configuration [here](https://github.com/varunpappu/articles/blob/main/grafana-k8s-monitoring/alloy-logs/pod-logs/values.yaml).

## References

* [podLogs `values.yaml`](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/charts/feature-pod-logs/values.yaml)
* [Alloy Logs Documentation](https://grafana.com/docs/k8s-monitoring/latest/features/logs/alloy-logs/)
* [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
* [Grafana Alloy Processing Stages](https://grafana.com/docs/alloy/latest/reference/components/loki.process/)
* [Promtail Configuration](https://grafana.com/docs/loki/latest/clients/promtail/configuration/)