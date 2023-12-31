# Prometheus Queries





CPU Throttling

```
`sum by(container, pod, namespace) (increase(container_cpu_cfs_throttled_periods_total{container!=""}[5m])) / sum by(container, pod, namespace) (increase(container_cpu_cfs_periods_total[5m]))`
```



```
# Example promql query to get API server requests by API Group, API Resource, and
# query verb in intervals of 5min
sum(rate(apiserver_request_total[5m])) by (group, resource, verb)

sum(rate(apiserver_request_total{verb!="watch"}[5m])) by (group, resource, verb)
```



```
# P90 API server latency for all request types
histogram_quantile(0.9, sum(rate(apiserver_request_duration_seconds_bucket[5m])) 
  by (le))

# P90 API server latency per verb
histogram_quantile(0.9, sum(rate(apiserver_request_duration_seconds_bucket[5m])) 
  by (le, verb))
```



```
# ETCD P90 latency
histogram_quantile(0.9, sum(rate(etcd_request_duration_seconds_bucket[5m])) 
  by (le))

# ETCD P90 latency per resource type
histogram_quantile(0.9, sum(rate(etcd_request_duration_seconds_bucket[5m])) 
  by (le, type))

```

```
# Total objects
sum(etcd_object_counts)

# Total objects per resource type
sum(etcd_object_counts) by (resource)

# Total number of events generated by kubernetes
sum(etcd_object_counts{resource="events"})
```




**`kube_pod_status_phase` and `kube_pod_status_scheduled`**
These metrics can be used to catch when pods get stuck in a pending state or fail to be scheduled and can be a warning that something has gone wrong. These state can occur when nodes fail to join the cluster or the Kubernetes scheduler is unable to place Pods for execution. 

**`kube_pod_container_status_restarts_total`**
This metric can be used to identify applications that have restarted due to failures for investigation.

**`kube_node_status_condition` and `kube_node_spec_unschedulable`**
Similar to the `kube_pod_status_*` metrics these can be used to identify problems with nodes. Nodes in a NotReady or unschedulable state would indicate problems with the nodes joining the cluster or encountering a resource problem. 




```
`kube_pod_status_phase{phase=~"Pending|Unknown"}

`
```



Prometheus alert for pods not ready for greater than 15 min, 

```
`name: KubePodNotReady
expr: sum by(namespace, pod) (max by(namespace, pod) (kube_pod_status_phase{job="kube-state-metrics",namespace=~".*",phase=~"Pending|Unknown"}) * on(namespace, pod) group_left(owner_kind) topk by(namespace, pod) (1, max by(namespace, pod, owner_kind) (kube_pod_owner{owner_kind!="Job"}))) > 0
for: 15m
labels:
severity: warning
annotations:
description: Pod {{ $labels.namespace }}/{{ $labels.pod }} has been in a non-ready state for longer than 15 minutes.
runbook_url: https://runbooks.prometheus-operator.dev/runbooks/kubernetes/kubepodnotready
summary: Pod has been in a non-ready state for more than 15 minutes.`
```


Promtheus query for API server reponse sizes by verb

```
histogram_quantile(0.9, sum(rate(apiserver_response_sizes_bucket[5m])) 
  by (le, verb))
```


CPU Usage:

```
sum(sum by(namespace, pod, container) (irate(container_cpu_usage_seconds_total{image!="", namespace="NAMESPACE", pod="POD"}[5m])) * on(namespace, pod) group_left(node) topk by(namespace, pod) (1, max by(namespace, pod, node) (kube_pod_info{node!=""}))) by (container)
```


Percentage CPU throttling


```
sum by(container, pod, namespace) (increase(container_cpu_cfs_throttled_periods_total{container!=""}[5m])) / sum by(container, pod, namespace) (increase(container_cpu_cfs_periods_total[5m]))
```
