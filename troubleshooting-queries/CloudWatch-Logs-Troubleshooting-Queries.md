# CloudWatch Logs Insights Queries


Below are a sample of queries for CloudWatch Insights. These examples will help you view important metrics and information within your EKS cluster, things like latencies, errors, and status changes. Consider this a starting point in your investigation and edit the queries to pin down specific information before exporting the results to a CSV or Markdown.

All of the information can be limited to just the top results by adding this limit statement to the end of the query


```
| limit 10
```



## Show the top clients that are listing pods

This query examines one of the most common scalability concerns that we see in Kubernetes clusters, it looks at all the API queries that are listing pods and counts the number of times that they’re doing it. In a smaller cluster, this usually isn’t a problem. But as the number of pods grows, the amount of work the cluster needs to do to list them grows along with the latency to get the response. This query lets us see if there are any clients that are listing pods more often than they need to, and to see if they’re doing any costly operations like trying to avoid cache. The usual culprits are logging and analytics tools which are trying to investigate everything that’s going on in your cluster.

As written, this query limits the results to the top 10 user agents. Feel free to remove that limit to see a ranking from all query sources. We’re filtering on LIST operations because they tend to be the most problematic, but you can expand the results to all queries if you remove the “verb” filter.

```
filter *@logStream* like "kube-apiserver-audit"
| filter ispresent(requestURI)
| filter verb = "list"
| filter requestURI like "/api/v1/pods"
| stats count(*) as count by userAgent
| sort count desc
# Get top ten LIST drivers
| limit 10
```



## Calculate the average response time for each request URI

This query looks complicated, but the regexes are only necessary to convert timestamps to seconds for some math calculations This is one of the most important tools we have to detect problems in your EKS cluster before they become critical. This query is limited to list operations (they tend to be the most problematic) and compares the timestamp for when the request was first made to the last status change of that query. This gives us the length of time that each query took. We’re averaging that for each request URI to get the average latency for each URI. This will show if there are specific Kubernetes API operations that are taking too long. Any URI over 20 seconds of latency is worth investigating. Feel free to remove the filter limiting the results to LIST operations for investigation purposes, but you should leave the filter removing WATCH operations, as this latency calculation method doesn’t work well for intentionally long running operations.

The results of this query ranks each URI (Kubernetes API endpoint) by latency, gives you the average latency in seconds, and a count of how many times each URI was called. During an investigation, you want to consider the number of requests against a URI along with the latency calculations. While a given operation may take the longest, it may not matter if it only happens a handful of times.


```
fields *@timestamp*, *@message*
| filter *@logStream* like "kube-apiserver-audit"
| filter ispresent(requestURI)
| filter verb = "list"
| filter verb not like "watch"
| parse requestReceivedTimestamp /\d+-\d+-(?<StartDay>\d+)T(?<StartHour>\d+):(?<StartMinute>\d+):(?<StartSec>\d+).(?<StartMsec>\d+)Z/
| parse stageTimestamp /\d+-\d+-(?<EndDay>\d+)T(?<EndHour>\d+):(?<EndMinute>\d+):(?<EndSec>\d+).(?<EndMsec>\d+)Z/
| fields (StartHour * 3600 + StartMinute * 60 + StartSec + StartMsec / 1000000) as StartTime, (EndHour * 3600 + EndMinute * 60 + EndSec + EndMsec / 1000000) as EndTime, (EndTime - StartTime) as DeltaTime
| stats avg(DeltaTime) as AverageDeltaTime, count(*) as CountTime by requestURI
| sort AverageDeltaTime desc

```



This is the same operation as above, but substitutes discovering the maximum latency instead of the average.

```
fields @timestamp, @message
| filter @logStream like "kube-apiserver-audit"
| filter ispresent(requestURI)
| filter verb = "list"
| filter verb not like "watch"
| parse requestReceivedTimestamp /\d+-\d+-(?<StartDay>\d+)T(?<StartHour>\d+):(?<StartMinute>\d+):(?<StartSec>\d+).(?<StartMsec>\d+)Z/
| parse stageTimestamp /\d+-\d+-(?<EndDay>\d+)T(?<EndHour>\d+):(?<EndMinute>\d+):(?<EndSec>\d+).(?<EndMsec>\d+)Z/
| fields (StartHour * 3600 + StartMinute * 60 + StartSec + StartMsec / 1000000) as StartTime, (EndHour * 3600 + EndMinute * 60 + EndSec + EndMsec / 1000000) as EndTime, (EndTime - StartTime) as DeltaTime
| stats max(DeltaTime) as MaxDeltaTime, count(*) as CountTime by requestURI
| sort MaxDeltaTime desc
```




## Count requests to apiserver by user agent

This query simply counts the total number of API queries made by each client (consolidated by user agent) against the Kubernetes API.

```
fields userAgent, requestURI, *@timestamp*, *@message*
| filter *@logStream* ~= "kube-apiserver-audit"
| stats count(userAgent) as count by userAgent
| sort count desc
```




## Count requests by URI and verb (additional filters are optional)

This query gives a number of filter examples that will help you investigate queries against specific URIs within the Kubernetes API. For example, if you want to see all the queries making actions against a replica set, or every API query that describes a cron job. These can be helpful deeper into an investigation when you’ve narrowed down a problem to a specific component or URI.

```
fields *@timestamp*, *@message*, *@logStream*, requestURI, verb
| filter *@logStream* =~ "kube-apiserver-audit"
# | filter strcontains(requestURI, "/apis/coordination.k8s.io/v1beta1/namespaces/kube-node-lease/leases/ip")
# | filter strcontains(requestURI, "/apis/autoscaling/v1/namespaces/default/horizontalpodautoscalers/")
# | filter requestURI like /\/apis\/apps\/v1\/namespaces\/.*\/deployments/
# | filter requestURI like /\/apis\/apps\/v1\/namespaces\/.*\/replicasets/
# | filter requestURI like /\/apis\/batch\/v1beta1\/namespaces\/.*\/cronjobs/
# | filter requestURI like /\/api\/v1\/nodes\/ip.*internal\/status/
# | filter requestURI like /\/api\/v1\/namespaces\/.*\/pods/
# | filter verb != "get"
# | filter verb != "watch"
# | filter verb != "list"
| display *@logStream*, requestURI, verb
| stats count(*) as count by requestURI, verb
| sort count desc
```



## Count all of the API queries that share a URI, verb, HTTP response code, and user agent

This query shows you the most common identical queries that are happening on your cluster. You can optionally filter out a specific verb.

```
fields requestURI, verb, responseStatus.code, userAgent
| filter *@logStream* like "kube-apiserver-audit"
# | filter verb != "get"
# | filter verb != "watch"
# | filter verb != "list"
| stats count(*) as count by requestURI, verb, responseStatus.code, userAgent
```



## List Apiserver 5XX errors by URL, Verb, HTTP Code, and User Agent

This query shows all of the HTTP status 500 errors returned by your Kubernetes API. Ideally this number should be zero, so it’s worth investigating any errors you find.

```
stats count(*) as count by requestURI, verb, responseStatus.code, userAgent
| filter *@logStream* =~ "kube-apiserver-audit"
| filter responseStatus.code >= 500 
| sort count desc
```




## List Apiserver 4XX  errors by URL, Verb, HTTP Code, and User Agent

This query shows all of the HTTP status 400 errors returned by your Kubernetes API. This is usually caused by outdated Kubernetes add-ons or controllers that are trying to use deprecated URIs. Additionally these errors will show up if your EKS cluster’s controller endpoint is exposed to the internet as random external clients will attempt to connect and perform actions on your cluster.

```
stats count(*) as count by requestURI, verb, responseStatus.code, userAgent
| filter *@logStream* =~ "kube-apiserver-audit"
| filter responseStatus.code >= 400
| filter responseStatus.code < 500
| sort count desc
```



## List failed anonymous requests:

This query lists all of the requests made against your EKS cluster without using credentials. This indicates that a client is misconfigured or that your cluster’s controller endpoint is exposed to traffic that you don’t intend.

```
`fields @timestamp, @message, sourceIPs.0
| sort @timestamp desc
| filter user.username="system:anonymous" and responseStatus.code in ["401", "403"]`
```



## List top writes to etcd

This query lists all of the operations that cause mutating actions against the underlying etcd database powering your EKS cluster. These are generally the CREATE, PATCH, and UPDATE verbs.

```
# count mutable requests, transactions, create, patch, update
fields @timestamp, @message, @logStream, requestURI, verb
| filter @logStream like "kube-apiserver-audit"
| filter verb not like "get"
| filter verb not like "list"
| filter verb not like "watch"
| display @logStream, requestURI, verb
| stats count(*) as count by requestURI, verb
| sort count desc
```




## Find updates/state changes for specific Node

Insert the name of the node (eg, “ip-192-168-34-73.us-east-2.compute.internal” in the filter for this query to see all of the updates and state changes made against it. This is useful for seeing when a worker node went out of “Ready” status.

```
filter @logStream like /^kube-apiserver-audit/
  | fields @logStream, @timestamp, @message
  | sort @timestamp desc
  | filter verb == "update" and requestURI like "/api/v1/nodes/<NODENAME>"
```




## List API Server health check failed

Report any API endpoint health check failures. See: https://kubernetes.io/docs/reference/using-api/health-checks/

```
fields *@message*
| sort *@timestamp* asc
| filter *@logStream* like "kube-apiserver"
| filter *@logStream* not like "kube-apiserver-audit"
| filter *@message* like "healthz check failed"
```




## **Report the top caller for actions mutating Secrets.**

Use this query to identify what clients are making modifications to secrets. 

```
fields userAgent, requestURI, *@timestamp*, *@message*
| filter *@logStream* like "kube-apiserver-audit"
| filter objectRef.resource = "secrets"
| filter verb != "list"
| filter verb != "watch"
| filter verb != "get"
| stats count(*) as count by userAgent, requestURI
| sort count desc
```



## **Report the top caller for actions mutating a specific secret.**

Get the same information as above, but for a specific secret.

```
filter @logStream like "kube-apiserver-audit"
| filter objectRef.resource = "secrets"
| filter verb != "list"
| filter verb != "watch"
| filter verb != "get"
| filter objectRef.name = "SECRETNAME"
| filter objectRef.namespace = "SECRETNAMESPACE"
| stats count(*) as count by userAgent, requestURI
| sort count desc
| limit 10 
```




## **Report ALL actions for given object
**

This query shows the updates and modifications to a specific object (like a Replicaset, deployment, or pods)

```
fields stageTimestamp, requestURI, verb, responseStatus.code, userAgent, *@message*
| filter *@logStream* like "kube-apiserver-audit"
| filter objectRef.name like "ResourceName"
# | filter verb != "get"
# | filter verb != "watch"
# | filter verb != "list"
| sort stageTimestamp asc
```



## Find occurrences of unscheduled pods in the scheduler log

Show any unscheduled pods on your cluster. You can optionally filter for a specific pod name to narrow down the results.

```
fields timestamp, pod, err, *@message*
| filter *@logStream* like "scheduler"
| filter *@message* like "Unable to schedule pod"
| parse *@message*  /^.(?<date>\d{4})\s+(?<timestamp>\d+:\d+:\d+\.\d+)\s+\S*\s+\S+\]\s\"(.*?)\"\s+pod=(?<pod>\"(.*?)\")\s+err=(?<err>\"(.*?)\")/
# | filter pod like "PODNAME"
| count(*) as count by pod, err
| sort count desc
```




## Find client side throttling on API components

Show any throttle warnings in the Kubernetes audit logs:
https://github.com/kubernetes/kubernetes/issues/95212

```
filter *@message* like "Throttling request"
```



## Find the top pods by the number of updates caused by unschedulable events

Similar to the query on scheduler logs you can see the number of times each pod has been evaluated as unschedulable by the cluster

```
fields requestURI, *@message*
| filter *@logStream* like "kube-apiserver-audit"
| filter requestURI like /pods/
| filter verb like /patch/
| filter *@message* like /Unschedulable/
# | filter count > 2
| stats count(*) as count by requestURI
| sort count desc
```



## Find the top pods by the number of updates caused by restart events

```
fields requestURI, *@message*
| filter *@logStream* like "kube-apiserver-audit"
| filter requestURI like /pods/
| filter verb like /patch/
| parse *@message* '*restartCount":*,"started"*' as head, restarts, tail
| filter restarts > 2
| sort restarts desc
| display requestURI, restarts
```



## Calculate the size of an object from the Create call

This is a bit janky but with the power of string parsing we should be able to pull out just the “pod spec” from a create call. Then using the `strlen()` function we can get the length in Unicode chars (and since we expect ASCII chars in the spec we can roughly assume ~1byte to a char)
**This may not be exact and could be off entirely depending on the string parsing. I use this as a tool to compare or get an idea of the size, validate by comparing** 

```
parse *@message* '*requestObject":*"spec":*"responseObject"*' as head, meta, spec, tail
| fields responseObject.metadata.name,  strlen(spec) as len
| filter verb like /create/
| filter objectRef.resource like /pods/
| filter requestObject.kind like /Pod/
| display responseObject.metadata.name, len, *@message*
| sort len desc
```


