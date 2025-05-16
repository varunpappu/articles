# **Routing Telemetry in Grafana K8s Monitoring: A Deep Dive into Destinations**

In the [first article](https://nsvarun14.medium.com/inside-grafana-kubernetes-monitoring-meet-the-alloy-modules-powering-your-observability-adbad8a06bf7) of this series, I introduced the Alloy modules that power the Grafana Kubernetes Monitoring ecosystem. These modules collect and provide telemetry data from your Kubernetes cluster, but the journey doesn't end there. Once your metrics, logs, traces, and profiles are captured, they need to be routed to the right destinations for further analysis and storage.

In this article, we'll explore the concept of **Destinations** in Grafana Kubernetes Monitoring—where telemetry data goes after being collected. We’ll dive into how you can configure different destinations, including Prometheus, Loki, and OTLP-compatible backends, to efficiently route your data.

---

## **Default Telemetry Routing in Grafana Kubernetes Monitoring**

Out-of-the-box, the Grafana Kubernetes Monitoring Helm chart configures your system to route telemetry data to three primary destinations, each specialized for different types of telemetry:

- **Prometheus**: For metrics
- **Loki**: For logs
- **OTLP (OpenTelemetry Protocol)**: For traces, metrics, and logs

These default destinations are configured in the `destinations` block within your Helm chart values. Below is an example of the default configuration:

```yaml
destinations:
  - name: metricsService
    type: prometheus
    url: http://prometheus-grafana-cloud-endpoint/api/prom/push

  - name: logsService
    type: loki
    url: http://loki-grafana-cloud-endpoint/loki/api/v1/push

  - name: otlpService
    type: otlp
    protocol: http
    metrics:
      enabled: true
    logs:
      enabled: true
    traces:
      enabled: true
    url: https://otel-grafana-cloud-endpoint/otlp
```

## **Understanding OTLP Service Configuration**

The **Alloy agents** handle the routing of telemetry data—metrics, logs, and traces—to destinations like Prometheus, Loki, and OTLP-compatible backends. These agents intelligently evaluate each destination’s capabilities and route telemetry accordingly, preserving native formats and minimizing unnecessary conversions.

At a high level, the routing logic follows these rules:

* **Partial OTLP Support**: When an OTLP destination supports only a subset of telemetry types (e.g., traces only), metrics and logs are routed to Prometheus and Loki, respectively. Any OTLP-formatted metrics or logs are translated into the appropriate format for these backends.

* **Full OTLP Support**: When an OTLP destination supports all telemetry types (metrics, logs, and traces), OTLP-native telemetry is routed directly to the OTLP backend. Prometheus and Loki will only receive data in their native formats, while OTLP-native telemetry bypasses them entirely.

### **Configuration Examples**

#### Example 1: OTLP Destination Supports Only Traces

```yaml
destinations:
  - type: prometheus
  - type: loki
  - type: otlp   # only supports traces
```

**In this configuration:**

* **Metrics**: All metrics are sent to Prometheus. If they are originally OTLP-formatted, they are translated into Prometheus format.
* **Logs**: All logs are sent to Loki, with OTLP logs translated as needed.
* **Traces**: Traces are sent directly to the OTLP destination.

This setup ensures compatibility by converting OTLP telemetry to match the supported formats of Prometheus and Loki when necessary.

#### Example 2: OTLP Destination Supports Metrics, Logs, and Traces

```yaml
destinations:
  - type: prometheus
  - type: loki
  - type: otlp   # supports metrics, logs, and traces
```

**In this configuration:**

* **Metrics**: Prometheus-native metrics go to Prometheus. OTLP-formatted metrics are routed directly to the OTLP destination without translation.
* **Logs**: Loki-native logs go to Loki. OTLP-formatted logs are routed directly to the OTLP destination, bypassing Loki.
* **Traces**: All traces are sent directly to the OTLP destination.

This setup ensures OTLP-native telemetry stays within the OTLP ecosystem, maintaining data fidelity and avoiding unnecessary format conversions.


> **Note:**
> Currently, only **OTLP over HTTP** is supported—**gRPC is not supported** at this time.

## **Authentication for Telemetry Destinations**

To ensure that only authorized services can push telemetry data to your destinations, authentication is often required. The Helm chart offers several authentication methods, each suited for different needs:

| Authentication Type | Description |
| ------------------- | --------------------------------------------------- |
| `none` | No authentication required. Default option. |
| `basic` | Basic authentication using a username and password. |
| `bearerToken` | Token-based authentication. |
| `oauth2` | OAuth 2.0 support (not covered in this example). |

### **1. Basic Authentication (Inline Configuration)**

For destinations requiring basic authentication, you can provide the credentials directly within your Helm chart values:

```yaml
destinations:
  - name: logsService
    type: loki
    url: http://loki-grafana-cloud-endpoint/loki/api/v1/push
    auth:
      type: basic
      username: "my-username"
      password: "my-password"
```

### **2. Bearer Token Authentication (Inline Configuration)**

For services requiring bearer token authentication, you can directly specify the token in the configuration:

```yaml
destinations:
  - name: logsService
    type: loki
    url: http://loki-grafana-cloud-endpoint/loki/api/v1/push
    auth:
      type: bearerToken
      bearerToken: "my-token"
```

Alternatively, for better security, store the token in a Kubernetes Secret:

```yaml
secret:
  embed: true
```

### **3. Managing Authentication with Kubernetes Secrets**

For greater security, you can store sensitive credentials (such as usernames and passwords) in Kubernetes Secrets and reference them within your configuration. This approach avoids exposing sensitive data in plain text:

```yaml
destinations:
  - name: otlpService
    type: otlp
    protocol: grpc
    metrics:
      enabled: false
    logs:
      enabled: false
    traces:
      enabled: true
    url: https://otel-grafana-cloud-endpoint/otlp
    auth:
      type: basic
      usernameKey: otlpUsername
      passwordKey: agentToken
    secret:
      create: false
      name: k8s-monitoring
      namespace: observability
```

---

## **Advanced Configuration: Additional Fields for Destinations**

Beyond authentication, Grafana Kubernetes Monitoring provides advanced configuration options within the `destinations` block. These fields offer deeper customization, especially in multi-cluster or dynamically configured environments.

| Field | Description | Supported Destinations |
| ------------------ | ----------------------------------------------------------------------- | ---------------------- |
| `extraHeaders` | Add extra HTTP headers to telemetry data. | Prometheus, Loki, OTLP |
| `extraHeadersFrom` | Dynamically reference HTTP headers from external sources or variables. | Prometheus, Loki, OTLP |
| `clusterLabels` | Add labels indicating the cluster name—useful for multi-cluster setups. | Prometheus, Loki, OTLP |
| `extraLabels` | Add extra labels to metrics before sending (not available for OTLP). | Prometheus, Loki |
| `extraLabelsFrom` | Dynamically add extra labels (not available for OTLP). | Prometheus, Loki |

### **Example: Configuring a Prometheus Destination with Extra Fields**

In a dynamic or multi-cluster environment, you can enrich your telemetry data with extra labels, headers, and cluster-specific information. Below is an example configuration for a **Prometheus** destination that utilizes these fields:

```yaml
destinations:
  - name: prometheusService
    type: prometheus
    url: http://prometheus.example.com/api/prom/push
    extraHeaders:
      Authorization: Bearer my-token
    extraHeadersFrom:
      X-Request-ID: {{ .Values.requestId }}
    clusterLabels:
      cluster: my-cluster
    extraLabels:
      env: production
    extraLabelsFrom:
      region: {{ .Values.region }}
      platform: env("PLATFORM")
```

This configuration dynamically injects values like `requestId`, `region`, and `platform`, ensuring that your telemetry data is enriched with contextual information.

---

## **Conclusion**

Understanding how telemetry data is routed within Grafana Kubernetes Monitoring is essential for building a reliable and secure observability pipeline. In this article, we took a deep dive into how Destinations are configured and used to send metrics, logs, and traces to services like Prometheus, Loki, and OTLP-compatible backends.

We also looked at different authentication mechanisms—basic auth, bearer tokens, and Kubernetes Secrets—to ensure secure data transmission. Finally, we explored advanced configuration options like extra headers, cluster labels, and dynamic labels, which allow you to tailor your telemetry pipeline for multi-cluster and production-grade environments.

You can find the complete values.yaml configuration [here](https://github.com/varunpappu/articles/blob/main/grafana-k8s-monitoring/destinations/values.yaml).

In the upcoming articles, we’ll begin exploring the individual Alloy modules—starting with alloy-logs, the component responsible for collecting logs from Kubernetes pods and nodes.

---

## **Additional Resources**

For more details on configuring specific destinations, refer to the following documentation:

* **[Loki Destination Documentation](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/docs/destinations/loki.md)**
* **[OTLP Destination Documentation](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/docs/destinations/otlp.md)**
* **[Prometheus Destination Documentation](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/docs/destinations/prometheus.md)**
