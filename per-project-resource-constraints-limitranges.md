
# Per-Project Resource Constraints: Limit Ranges

## Motivations

Within a namespace, a Pod can consume as much CPU and memory as is allowed by the ResourceQuotas that apply to a namespace. As a cluster operator, or as a namespace-level administrator, you might be concerned about making sure that a single object cannot monopolize all available resources within a namespace.

In fact, Kubernetes users might have further resource management needs within a namespace.

- Users might accidentally create workloads that consume too much of the namespace quota. These unwanted workloads might prevent other workloads from running.

- Users might forget to set workload limits and requests, or might find it time-consuming to configure limits and requests. When a namespace has a quota, creating workloads fails if the workload does not define values for the limits or requests in the quota.

Kubernetes introduces limit ranges to help with these issues. Limit ranges are namespaced objects that define limits and requests for workloads within the namespace.

## An example of a project with a quota

Consider a namespace with the following quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: example
  namespace: example
spec:
  hard:
    limits.cpu: "8"
    limits.memory: 8Gi
    requests.cpu: "4"
    requests.memory: 4Gi
```

The following command creates a deployment:

```sh
$ oc new-app --name example httpd
```

The quota prevents the deployment from creating pods:

```sh
$ oc get event --sort-by .metadata.creationTimestamp
```
```
LAST SEEN   TYPE      REASON              OBJECT                          MESSAGE
...output omitted...
13s         Warning   FailedCreate        replicaset/example-74c57c8dff   Error creating: pods "example-74c57c8dff-rzl7w" is forbidden: failed quota: example: must specify limits.cpu for: httpd; limits.memory for: httpd; requests.cpu for: httpd; requests.memory for: httpd
...
```

Limit ranges can specify the following limit types:

**Default limit**

&nbsp;&nbsp;&nbsp;&nbsp;Use the default key to specify default limits for workloads.

**Default request**

&nbsp;&nbsp;&nbsp;&nbsp;Use the defaultRequest key to specify default requests for workloads.

**Maximum**

&nbsp;&nbsp;&nbsp;&nbsp;Use the max key to specify the maximum value of both requests and limits.

**Minimum**

&nbsp;&nbsp;&nbsp;&nbsp;Use the min key to specify the minimum value of both requests and limits.

**Limit-to-request ratio**
The maxLimitRequestRatio key controls the relationship between limits and requests. If you set a ratio of two, then the resource limit cannot be more than twice the request.

## The LimitRange resource 

The following limit range includes all types of limits:

```yaml
apiVersion: "v1"
kind: "LimitRange"
metadata:
  name: "resource-limits"
  namespace: <ProvideTheNameSpaceName>
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 125m
      memory: 128Mi
    maxLimitRequestRatio:
      cpu: "10"   
```

Limit ranges do not affect existing pods. If you delete the deployment and run the oc create command again, then the deployment creates a pod with the applied limit range.

```sh
$ oc apply -f resource-limits.yaml
```

```sh
$ oc describe pod
...
Containers:
  httpd:
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:        250m
      memory:     256Mi
...
```

