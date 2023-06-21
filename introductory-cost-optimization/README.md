# Introductory Cost Optimization Dashboard

## Requirements
This dashboard assumes that you have the Kube Prometheus Stack installed on your Kubernetes cluster. You can find the latest version of that project here:

https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

## Description
Each pannel in this dashboard is numbered, the descriptions below correspond with the labeled panels:


1. Panel 1: Compares the total memory available across all of your nodes versus the amount of memory that has been requested by your pods. The gap between these numbers is memory that you are paying for but cannot use in your pods.
2. Panel 2: shows unrequested memory by worker node. This will let you see if there are any outliers hidden in the cluster-wide view of the first panel. Use this to identify any nodes that need deeper investigation
3. Panel 3: Is a cluster-wide count of how many gigabytes of memory are not allocated to pods. The portion of this graph shaded in orange is the potential savings from optimizing memory requests. The blue line on this graph charts the actual memory utilization in the cluster. You want a small delta between the blue line (utilization) and the green line (requested memory)
4. Panel 4: Cluster-wide view of the accuracy of your pod memory requests. This is a sum of the number of gigabytes of memory your pods have requested but aren’t using. Reducing this number means your nodes will have tighter bin-packing and less wasted resources.
5. Now we’re going to look at the CPU resource utilization in the cluster.  Panel 5: Compares the total CPU available across all of your nodes versus how much of that has been requested by your pods. Since pods can better share CPU resources, this panel is more of a relative view of unused CPU in your cluster.
6. Panel 6: shows unrequested CPU by worker node. This will let you see if there are any outliers hidden in the cluster-wide view of the first panel. Use this to identify any nodes that need deeper investigation
7. Panel 7: Is a cluster-wide count of how many CPU cores are not requested by pods. The portion of this graph shaded in orange is the potential savings from optimizing memory requests. The blue line on this graph charts the actual CPU utilization in the cluster. While unallocated cores can be used by other pods, a high number over a long period of time suggests that the requests are not set properly
8. Panel 8: Here we’re looking at the amount of CPU used against the amount requested. We’re breaking it down by node so you can focus on the workloads with the most impact.


