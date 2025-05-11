# **Routing Telemetry in Grafana Kubernetes Monitoring: A Deep Dive into Destinations**

In the [first article](https://nsvarun14.medium.com/inside-grafana-kubernetes-monitoring-meet-the-alloy-modules-powering-your-observability-adbad8a06bf7) of this series, I introduced the Alloy modules that power the Grafana Kubernetes Monitoring ecosystem. These modules collect and provide telemetry data from your Kubernetes cluster, but the journey doesn't end there. Once your metrics, logs, traces, and profiles are captured, they need to be routed to the right destinations for further analysis and storage.

In this article, we'll explore the concept of **Destinations** in Grafana Kubernetes Monitoring—where telemetry data goes after being collected. We’ll dive into how you can configure different destinations, including Prometheus, Loki, and OTLP-compatible backends, to efficiently route your data. I’ll also show you how to use the OTLP destination to streamline your telemetry pipeline, especially when integrating with services like Grafana Cloud.

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
````

### **Understanding OTLP Service Configuration**

When configuring the OTLP destination, be mindful that if you connect to a Tempo endpoint, it will only process traces and will ignore logs and metrics. To ensure all telemetry data is processed correctly, use an **OTEL endpoint**. Requests sent to OTEL will automatically be forwarded to Tempo and then properly routed to Grafana Cloud for further handling.

---

## **Authentication for Telemetry Destinations**

To ensure that only authorized services can push telemetry data to your destinations, authentication is often required. The Helm chart offers several authentication methods, each suited for different needs:

| Authentication Type | Description                                         |
| ------------------- | --------------------------------------------------- |
| `none`              | No authentication required. Default option.         |
| `basic`             | Basic authentication using a username and password. |
| `bearerToken`       | Token-based authentication.                         |
| `oauth2`            | OAuth 2.0 support (not covered in this example).    |

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
      name: grafana-agent-secrets
      namespace: monitoring
```

---

## **Advanced Configuration: Additional Fields for Destinations**

To fine-tune your telemetry routing, Grafana Kubernetes Monitoring allows you to configure additional fields within the `destinations` block. These fields provide deeper customization, especially in multi-cluster or dynamically-configured environments.

| Field              | Description                                                             | Supported Destinations |
| ------------------ | ----------------------------------------------------------------------- | ---------------------- |
| `extraHeaders`     | Add extra HTTP headers to telemetry data.                               | Prometheus, Loki, OTLP |
| `extraHeadersFrom` | Dynamically reference HTTP headers from external sources or variables.  | Prometheus, Loki, OTLP |
| `clusterLabels`    | Add labels indicating the cluster name—useful for multi-cluster setups. | Prometheus, Loki, OTLP |
| `extraLabels`      | Add extra labels to metrics before sending (not available for OTLP).    | Prometheus, Loki       |
| `extraLabelsFrom`  | Dynamically add extra labels (not available for OTLP).                  | Prometheus, Loki       |

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

## **Summary**

In this article, we’ve explored the various **destinations** in Grafana Kubernetes Monitoring and how to configure them for routing telemetry data effectively. From **Prometheus** for metrics to **Loki** for logs and **OTLP** for traces, understanding how to route telemetry data to the appropriate backends is essential for efficient observability within your cluster.

Additionally, we covered the different **authentication methods**, including basic authentication, bearer tokens, and Kubernetes Secrets, to securely manage access to your telemetry destinations. Lastly, we examined advanced configuration options, such as adding extra headers, labels, and cluster-specific information, to further customize your telemetry pipeline.

By configuring your destinations and leveraging Grafana’s powerful monitoring tools, you can ensure that telemetry data is securely and efficiently routed, providing you with real-time insights into your Kubernetes environment.

---

## **Additional Resources**

For more details on configuring specific destinations, refer to the following documentation:

* **[Loki Destination Documentation](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/docs/destinations/loki.md)**
* **[OTLP Destination Documentation](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/docs/destinations/otlp.md)**
* **[Prometheus Destination Documentation](https://github.com/grafana/k8s-monitoring-helm/blob/main/charts/k8s-monitoring/docs/destinations/prometheus.md)**

