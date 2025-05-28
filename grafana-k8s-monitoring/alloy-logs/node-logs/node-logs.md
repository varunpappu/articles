# Inside Grafana K8s Monitoring: Unlocking Node-Level Observability with alloy-logs and nodeLogs

The `alloy-logs` module in Grafana k8s-Monitoring helps you build robust logging pipelines across your Kubernetes cluster. One of its most powerful features is `nodeLogs`, which gives you deep visibility into the nodes themselves—not just the containers running on them.

For anyone managing Kubernetes, node-level logs are essential when debugging stability issues, performance bottlenecks, or networking problems that container logs alone can’t explain.

## Why Node-Level Logs Matter

When a Kubernetes pod crashes, restarting it may fix the issue temporarily. But if the root cause is on the host node—like a misconfigured kubelet or a failing runtime daemon—you need logs from the node itself.

By collecting logs from **journald** (the system log manager used on most Linux nodes), `nodeLogs` allows you to:

- Monitor system services like `kubelet`, `containerd`, and `node-problem-detector`
- Debug node-level failures affecting workloads
- Correlate system behavior with application performance
- Investigate incidents like failed pulls, service restarts, or degraded nodes

## Understanding Journald

`journald` is the logging subsystem in `systemd` that captures structured logs for all services on a Linux node. Unlike plain text log files, journald includes rich metadata—like severity, executable path, and service identifiers—making it powerful for filtering and analysis.

Each system-level service (like `kubelet` or `containerd`) is managed by `systemd`, and logs are grouped by **unit name**. This structure makes it easy to isolate and analyze specific services.

## Log Structure: What Journald Provides

Journald logs include a wealth of metadata, which helps filter and analyze logs more effectively. Key fields include:

| Field                     | Description                                        |
|---------------------------|----------------------------------------------------|
| `__journal__systemd_unit` | The systemd unit generating the log               |
| `MESSAGE`                 | The main log message                              |
| `PRIORITY`                | Severity level (e.g., 3 = Error, 6 = Info)        |
| `UNIT`                    | Alias for the service unit name                   |
| `SYSLOG_IDENTIFIER`       | Process name from syslog (e.g., `kubelet`)        |
| `EXE`                     | Path to the executable emitting the log           |

### Sample Log Entry

Here’s an example of a journald log entry from kubelet:

```text
__journal__systemd_unit=kubelet.service
MESSAGE=Kubelet is now running on node worker-1
PRIORITY=6
UNIT=kubelet.service
SYSLOG_IDENTIFIER=kubelet
EXE=/usr/bin/kubelet
````

You can inspect logs manually on a node using the following command:

```bash
journalctl -u kubelet.service -o verbose
```

## Enabling `nodeLogs`

Now that we’ve covered why node-level logs are essential, let’s dive into how you can enable and configure `nodeLogs` in your Grafana k8s-Monitoring Helm chart. This powerful feature not only collects logs from the system services running on the nodes, but also integrates them seamlessly into your observability pipeline.

The `nodeLogs` configuration in the **alloy-logs** module allows you to enable this functionality with ease. By simply setting the **enabled** flag to `true`, you deploy an alloy-logs DaemonSet that scrapes logs directly from the journald subsystem on each node. These logs are then enriched and filtered based on your configuration and routed through your telemetry pipeline—just like your container logs.

### Basic Configuration

To activate `nodeLogs`, you just need to add the following to your Helm values file:

```yaml
nodeLogs:
  enabled: true
```

This is a straightforward configuration that sets the foundation for a comprehensive node-level logging setup, capturing logs from the host node to provide deeper insights into your Kubernetes cluster's health.

## Key SystemD Units to Monitor

When configuring `nodeLogs`, you'll likely want to focus on logs from the most important system services. The **units** field allows you to specify which services you want to capture logs from. By filtering out unnecessary services, you can keep your logs focused and relevant.

Here are the key systemd units to monitor in a Kubernetes environment:

| Unit Name                       | Description                                                        |
| ------------------------------- | ------------------------------------------------------------------ |
| `kubelet.service`               | The Kubernetes node agent. Critical for scheduling and node health |
| `containerd.service`            | Container runtime. Logs pull/start failures                        |
| `docker.service`                | Alternative container runtime (if you're using Docker instead)     |
| `node-problem-detector.service` | Detects node-level issues like disk pressure and kernel deadlocks  |

### Example Configuration

To monitor these key units, you can set the `units` field as follows:

```yaml
journal:
  units:
    - kubelet.service
    - containerd.service
    - node-problem-detector.service
```

If you leave the `units` field empty, **all** systemd services will be captured, which can flood your pipeline with irrelevant data. It’s best to specify the services that are most important for your use case.

## Configuration Options

The **nodeLogs** section provides a variety of settings to further customize the log collection process. You can define how logs are captured, enriched, and processed.

### Basic Journald Settings

Here are some important settings you can customize in your configuration:

| Key                    | Type   | Default                             | Description                                  |
| ---------------------- | ------ | ----------------------------------- | -------------------------------------------- |
| `journal.path`         | string | `"/var/log/journal"`                | Path to journald logs on the node            |
| `journal.maxAge`       | string | `"8h"`                              | Maximum age of logs to collect               |
| `journal.jobLabel`     | string | `"integrations/kubernetes/journal"` | Label to tag this log stream in your backend |
| `journal.formatAsJson` | bool   | `false`                             | Emit logs as structured JSON                 |
| `journal.units`        | list   | `[]`                                | List of services to collect logs from        |

## Pre-Scrape and Post-Scrape Processing

To ensure your logs are collected in the most efficient way possible, you can leverage **pre-scrape** and **post-scrape** processing options. These allow you to filter logs before they are sent to your backend or transform the log data after it is collected.

### Pre-Scrape Filtering

For example, you can filter logs before they even hit your backend:

```yaml
extraDiscoveryRules: |
  - action: keep
    regex: kubelet.service|containerd.service
    source_labels: [__journal__systemd_unit]
```

### Post-Scrape Processing

You can also transform the collected logs after they are scraped:

```yaml
extraLogProcessingStages: |
  - json:
      expressions:
        message: MESSAGE
        priority: PRIORITY
  - template:
      source: priority
      template: |
        {{ if eq .Value "3" }}error
        {{ else if eq .Value "4" }}warning  
        {{ else if eq .Value "6" }}info
        {{ else }}debug{{ end }}
      target: level
```

## Controlling Labels and Metadata

Managing your log metadata is just as important as collecting the logs themselves. Use the **labelsToKeep** and **structuredMetadata** fields to control which metadata gets passed through your system.

For example:

```yaml
labelsToKeep:
  - instance
  - job
  - unit
  - level

structuredMetadata:
  severity: level
  service: unit
```

## Complete Configuration Example

Here's a complete example of how you can configure `nodeLogs`:

```yaml
nodeLogs:
  enabled: true
  journal:
    path: "/var/log/journal"
    maxAge: "24h"
    units:
      - kubelet.service
      - containerd.service
      - node-problem-detector.service
  extraDiscoveryRules: |
    - action: keep
      regex: kubelet.service|containerd.service|node-problem-detector.service
      source_labels: [__journal__systemd_unit]
  extraLogProcessingStages: |
    - json:
        expressions:
          message: MESSAGE
          priority: PRIORITY
    - template:
        source: priority
        template: |
          {{ if eq .Value "3" }}error
          {{ else if eq .Value "4" }}warning  
          {{ else if eq .Value "6" }}info
          {{ else }}debug{{ end }}
        target: level
  labelsToKeep:
    - instance
    - job
    - unit
    - level
  structuredMetadata:
    severity: level
```

## Wrapping Up

To get the most out of `nodeLogs`, keep these essential practices in mind:

- **Start small:** Focus on `kubelet.service` and `containerd.service` first.
- **Limit retention:** Use 8–24 hours unless specific needs require longer.
- **Monitor volume:** Watch log output to avoid overloading storage backends.
- **Set alerts:** Create notifications for high-severity logs from critical services.

`nodeLogs` bridges the gap between application monitoring and infrastructure observability. By capturing system-level events that container logs miss, it provides the deep visibility needed to maintain stable, secure Kubernetes operations and resolve issues faster than ever before.

You can find the complete `values.yaml` configuration [here](https://github.com/varunpappu/articles/blob/main/grafana-k8s-monitoring/alloy-logs/node-logs/values.yaml).



## References

* [nodeLogs `values.yaml`](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/charts/feature-node-logs/values.yaml)
* [Alloy Logs Documentation](https://grafana.com/docs/k8s-monitoring/latest/features/logs/alloy-logs/)
* [systemd.journal-fields(7)](https://www.freedesktop.org/software/systemd/man/latest/systemd.journal-fields.html)
* [RFC 3164 - Syslog Priority Values](https://tools.ietf.org/html/rfc3164#section-4.1.1)
* [journalctl - Output Formats](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html#-o)
