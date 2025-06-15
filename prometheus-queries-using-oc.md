# Prometheus Queries Using `oc` CLI

This document provides common commands and steps to fetch useful metrics from OpenShift’s Prometheus monitoring via the `oc` CLI. These queries help monitor cluster health and performance.

---

## Prerequisites

- Access to OpenShift CLI (`oc`) configured with the necessary permissions  
- Prometheus Operator installed and accessible in the cluster  
- Basic familiarity with Prometheus query language (PromQL) is helpful  

---
## Setup

Before running queries, set the Prometheus route and authentication token:

```bash
# Get Prometheus route hostname
PROM_ROUTE=$(oc -n openshift-monitoring get route prometheus-k8s -o jsonpath='{.spec.host}')
echo $PROM_ROUTE

# Create authentication token for Prometheus access
TOKEN=$(oc -n openshift-monitoring create token prometheus-k8s)
echo $TOKEN
```

## Common Prometheus Queries via `oc` CLI

### 1. CPU Usage per Pod and Namespace (Last 5 Minutes)

This query fetches the CPU usage rate (in CPU seconds per second) aggregated by **pod** and **namespace** over the last 5 minutes. It helps you identify which pods and namespaces are consuming the most CPU resources.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=\"\"}[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

#### Sample Output Explanation

When you run the Prometheus query via the API, you’ll get a JSON response similar to this:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "namespace": "openshift-cluster-samples-operator",
          "pod": "cluster-samples-operator-6b95f78db9-ffx76"
        },
        "value": [
          1749999523.970,
          "0.0013389141736876663"
        ]
      },
      {
        "metric": {
          "namespace": "openshift-network-node-identity",
          "pod": "network-node-identity-67v4t"
        },
        "value": [
          1749999523.970,
          "0.001082561531067391"
        ]
      }
    ]
  }
}
```

#### What does this output mean?

* **`status: "success"`** indicates the query was executed successfully
* **`resultType: "vector"`** means the data is a set of instantaneous values (a vector of metrics)
* **`result`** contains an array of metric objects

Each object in `result` includes:

* **`metric`** — labels identifying the metric instance; here, it includes `namespace` and `pod` names
* **`value`** — an array where:

  * The first element is the **Unix timestamp** (in seconds) when the metric was recorded
  * The second element is the **metric value** as a string (CPU usage rate in this example)

#### How to read the values?

For example, the first item shows the CPU usage rate of approximately `0.00134` CPU cores for the pod `cluster-samples-operator-6b95f78db9-ffx76` in the namespace `openshift-cluster-samples-operator` at the given timestamp.

To convert the Unix timestamp into a human-readable date, you can use the `date` command:

```bash
$ date -r 1749999523
Sun Jun 15 20:28:43 IST 2025
```

---


### 2. Memory Usage per Pod and Namespace

This command retrieves the current memory usage (in bytes) for each pod, grouped by pod and namespace.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (container_memory_usage_bytes{container!=\"\"})'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 3. Container Restarts per Pod and Namespace (Last 1 Hour)

This command shows the rate of container restarts over the past 1 hour, grouped by pod and namespace.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(kube_pod_container_status_restarts_total[1h]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 4. Network I/O per Pod and Namespace (Last 5 Minutes)

This command returns the combined rate of network receive and transmit traffic (in bytes per second) over the last 5 minutes, grouped by pod and namespace.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(container_network_receive_bytes_total{container!=\"\"}[5m]) + rate(container_network_transmit_bytes_total{container!=\"\"}[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 5. 95th Percentile API Server Request Duration (Last 5 Minutes)

This command calculates the 95th percentile (high) latency of API server requests over the last 5 minutes, grouped by resource and verb. The 95th percentile means that **95% of all API server requests were faster than this duration**, and only the slowest 5% took longer.

It's a common way to measure latency while ignoring extreme outliers (e.g., a few very slow requests), giving you a more realistic view of typical performance.


```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('histogram_quantile(0.95, sum by (le, resource, verb) (rate(apiserver_request_duration_seconds_bucket{job=\"apiserver\"}[5m])))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 6. API Server Request Count by HTTP Status Code (Last 5 Minutes)

This command retrieves the total count of API server requests per HTTP status code over the last 5 minutes.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (code) (rate(apiserver_request_total{job=\"apiserver\"}[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 7. API Server Request Count by Status Code, Resource, and Verb (Last 5 Minutes)

This command shows the rate of API server requests over the last 5 minutes, grouped by HTTP status code, resource type, and HTTP verb (GET, POST, etc.).

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (code, resource, verb) (rate(apiserver_request_total{job=\"apiserver\"}[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 9. 99th Percentile Etcd Disk Backend Commit Duration (Last 5 Minutes)

This command calculates the 99th percentile latency for etcd disk backend commit operations over the last 5 minutes. The 99th percentile indicates that 99% of etcd commit operations complete faster than this duration, highlighting the worst-case latency experienced by a small fraction (1%) of operations.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('histogram_quantile(0.99, sum by (le) (rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```
---
### 10. Check if Etcd Server Has a Leader

This command checks whether the etcd server currently has a leader elected.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=etcd_server_has_leader" -H "Authorization: Bearer ${TOKEN}" | jq
```
---


### 11. Total API Server Audit Events Rate (Last 5 Minutes)

This command retrieves the total rate of API server audit events over the past 5 minutes.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum(rate(apiserver_audit_event_total[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

#### Understanding the Output

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [
          1750000858.902,
          "125.52592592592593"
        ]
      }
    ]
  }
}
```

* **`status: "success"`** means the query was executed successfully.
* **`resultType: "vector"`** indicates a set of instantaneous values.
* **`result`** contains the metric data array.

In this example:

* The **timestamp** (`1750000858.902`) shows when the metric was recorded (Unix time in seconds).
* The **value** (`125.5259`) represents the **average rate of API server audit events per second** during the last 5 minutes.

This tells you that on average, about 125 audit events were processed per second in the 5-minute window.

---

### 12. Count of Pods per Namespace

This command returns the count of pods grouped by their namespace.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('count(kube_pod_info) by (namespace)'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```
---

### 13. Node CPU Usage Percentage (Last 5 Minutes)

This command calculates the CPU usage percentage per node by subtracting the average idle CPU percentage from 100%, based on the last 5 minutes.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('100 - (avg by (instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 14. Container Filesystem Usage Percentage

This command calculates how much of the allocated filesystem storage each container is currently using, expressed as a percentage.

- It takes the amount of storage used by the container (`container_fs_usage_bytes`)  
- Divides it by the container’s total storage limit (`container_fs_limit_bytes`)  
- Then multiplies by 100 to convert it to a percentage  

This helps you quickly see which containers are approaching their storage limits.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('(container_fs_usage_bytes / container_fs_limit_bytes) * 100'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 15. Network Transmit Errors per Pod and Namespace (Last 5 Minutes)

This command fetches the rate of network transmit errors observed for each container, grouped by pod and namespace, over the last 5 minutes. It helps identify pods that are experiencing network issues when sending data.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(container_network_transmit_errors_total[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 16. Network Receive Errors per Pod and Namespace (Last 5 Minutes)

This command fetches the rate of network receive errors observed for each container, grouped by pod and namespace, over the last 5 minutes. It helps identify pods that are experiencing network issues when receiving data.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(container_network_receive_errors_total[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 17. Network Bytes Transmitted and Received per Pod and Namespace (Last 5 Minutes)

These commands show the rate of network bytes transmitted and received by each container, grouped by pod and namespace, averaged over the last 5 minutes.

#### Bytes Transmitted

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(container_network_transmit_bytes_total[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

#### Bytes Received

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(container_network_receive_bytes_total[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```
---


### 18. CoreDNS Request Duration – 95th Percentile (Last 5 Minutes)

This command calculates the 95th percentile of DNS request duration handled by CoreDNS over the last 5 minutes.

- It shows how long 95% of DNS requests took to complete.
- Useful for identifying DNS latency or performance issues affecting the cluster.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('histogram_quantile(0.95, sum by (le) (rate(coredns_dns_request_duration_seconds_bucket[5m])))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
````

#### Sample Output

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [
          1750003862.319,
          "0.0019697222222222225"
        ]
      }
    ]
  }
}
```

#### Explanation

* The **value** `0.00197` means 95% of DNS requests completed in under \~2 milliseconds.
* This is a good health indicator for internal DNS resolution time.

---

### 19. Available Endpoint Count per Service and Namespace

This command returns the number of available pod endpoints behind each Kubernetes service, grouped by service and namespace.

- It’s useful for checking how many healthy pod endpoints are backing each service.
- If a service has `0` available endpoints, it may indicate that pods are not running, not ready, or not correctly matched by the service selector.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('count(kube_endpoint_address_available) by (service, namespace)'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```

---

### 20. Network Packet Drops per Node and Interface (Last 5 Minutes)

This command retrieves the rate of network packet drops (both receive and transmit) for each network interface on every node over the last 5 minutes.

- It adds the receive and transmit drop rates using:
  - `node_network_receive_drop_total`
  - `node_network_transmit_drop_total`
- Grouped by `instance` (node) and `device` (network interface)

High values may indicate network congestion, hardware issues, or driver problems.

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (instance, device) (rate(node_network_receive_drop_total[5m]) + rate(node_network_transmit_drop_total[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```
---

## 🧾 Formatting Prometheus API Output in Tabular Format (using `jq` + `@tsv`)

By default, Prometheus queries return long JSON structures that can be hard to read. To make the output of any Prometheus query easier to read, especially in the terminal, you can use this generic and concise `jq` command to tabulate results.

### ✅ Easy-to-Read Format (Works for Any Query)

```bash
curl -s -k "$PROM_URL" -H "Authorization: Bearer ${TOKEN}" | jq -r '
  .data.result[] |
  [.metric[]?, .value[1]] |
  @tsv'
```
---

## Filtering Prometheus Queries by Namespace in OpenShift CLI

When querying Prometheus metrics via the OpenShift CLI (`oc`) or direct API calls, you often want to **restrict results to a specific namespace** to focus on relevant workloads. This allows to focus troubleshooting on a particular project/namespace. Helps reduce noise from unrelated metrics and speed up queries by limiting scope.

### How to Add Namespace Filtering in PromQL Queries

Prometheus metrics can be filtered by labels inside curly braces `{}`. The **namespace label** is commonly named `namespace`.

### Syntax

```promql
metric_name{namespace="your-namespace", other_label="value", ...}
```

### Example: Filtering CPU Usage by Namespace

### Original Query (all namespaces):

```promql
sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
```

### Namespace-filtered Query (`my-namespace`):

```promql
sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{namespace="my-namespace", container!=""}[5m]))
```

---

### How to Use This in Your `curl` Command

1. Set your namespace variable:

```bash
NAMESPACE="my-namespace"
```

2. Use Python to URL-encode the query with the namespace filter:

```bash
curl -s -k "https://${PROM_ROUTE}/api/v1/query?query=$(python3 -c "import urllib.parse; print(urllib.parse.quote('sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{namespace=\"$NAMESPACE\", container!=\"\"}[5m]))'))")" -H "Authorization: Bearer ${TOKEN}" | jq
```


### Summary

* **Add** `namespace="your-namespace"` inside metric selectors `{}` to filter.
* **URL-encode** the entire query when passing via API.
* This approach applies to **any Prometheus query** you use.

---

### Tip: Modify Other Queries

Simply insert the namespace filter label inside the curly braces in your PromQL expressions, for example:

```promql
rate(kube_pod_container_status_restarts_total{namespace="my-namespace"}[1h])
```


