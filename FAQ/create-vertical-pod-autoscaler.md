# VPA
The idea is to beef and/or trimm a pod under certain circumstances.

### Installing
VPA is a component that has to be installed in cluster. Clone this repo and install components:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vpa-release-1.0/vertical-pod-autoscaler/deploy/vpa-v1-crd-gen.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vpa-release-1.0/vertical-pod-autoscaler/deploy/vpa-rbac.yaml
```
Maybe helm has its chart about VPA?

### How does this work
VPA is made of three pieces: recomender, updater, admission controller.

Recomender monitor status and keep tracks of past resource consumption to provide hints, recomendations, about next in time cpu and memory requirements. It relies on VerticalPodAutoscalerCheckpoint to store and fetch history about managed pods to build recomendations. Can also rely on Prometheus. VerticalPodAutoscalerCheckpoint update its model status by using watch, metrics api, then compute recomendation for VPA.

Updater checks managed pods have correct resources set; if they don't updater kills pods to recreate them with updated requests. Fetches live pod information and check them against recomendations.

Admission controller set correct resources requests for new pods. Consider new the pods created because update killing activity.


### Usage
VPA mainly works in four modes:
- 'Auto': when requested resources differ significantly from recomendation, pod is beefed/trimmed without eviction (?),
- 'Recreate': when requested resources differ significantly from last recomendation, pod is evicted and recreated with more resources,
- 'Initial': assign resources once and never change them,
- 'Off': VPA doesn't perform any action but calculates recomendations and log them for administrators to check.

### Examples
Basic:
```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cache-vpa
  namespace: caching
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: cache-statefulset
  updatePolicy:
    updateMode: "Initial"
```

A more significative one:
```
piVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: hamster
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 1
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
```

Default `updatePolicy` is `Auto` (?):
```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: redis-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: redis-master
```

Reference: https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/docs
