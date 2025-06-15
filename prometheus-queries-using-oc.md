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
