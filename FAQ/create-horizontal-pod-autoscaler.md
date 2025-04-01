## How to configure Horizontal Pod Autoscaler (HPA)
HPA refers to a deployment by adjusting number of replicas accordingly to specific metrics.

HPA scaling process is automatic.

Suppose we have this deployment to scale:
`k create deployment nginx-alpine-deploy --image=nginx:alpine --replicas=3`
Edit that deployment to specify `resources.requests` resource of deploy's container.

### Configure HPA to scale on memory usage maintaining it on average 50% on all pods
The following manifest tells the HPA to nail requirement:
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-alpine-deploy
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: memory # cpu is also allowed as resource name
      target:
        type: Utilization
        averageUtilization: 50 # you can also provide absolute value here
```
This is basically the setup you need to scale on memory or cpu resources: when cpu consumption rises pod are added to a maximum of 15 to keep cpu usage on an average 50%.

Deploy should indicate `resources.requests.cpu="10m"` to specify cpus required or hpa won't have anything to check against while decide to scale.

### Going into HPA metrics details
HPA works with: metrics, custom metrics, external metrics
HPA works with pod resources:
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-alpine-deploy
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Pod
    pods:
      metric:
      ...
```
and object resources:
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-alpine-deploy
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Object
    object:
      metric:
      ...
```

metric-server gives cpu and memory metrics about pods and worker nodes. Metrics are specified in the `spec.metrics` array in hpa manifest. Custom metrics require a metric server to collect such informations. Metric server must be installed, k8s has its own solution that can be found here: `https://github.com/kubernetes-sigs/metrics-server` or by helm chart.

Metrics are too wide to elaborate here, doc is also kinda surfacing the argumen. This is one of the most complete metrics list:

https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md

repo seems to have other useful resources.
