# Endpoints and EndpointSlices
Services expose pods or deployments (so, pods)by using `selector`:
```
apiVersion: v1
kind: Service
metadata:
  name: persistence-layer
spec:
  type: ClusterIP
  ports:
    - targetPort: 80
      port: 80
  selector:
    app: myapp
    type: sometype
```
`selector` object allows you to expose a set of pods by a service, all the pods that match `labels` pointed in `spec.selector`.

When service has no `spec.selector` you may use `Endpoint` or `EndpointSlice`:
```
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: http # should match with the name of the service port defined above
    appProtocol: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses:
      - "10.4.5.6"
  - addresses:
      - "10.1.2.3"
```
`metadata.labels.kubernetes.io/service-name` must match the name of the service this `EndpointSlice` is supporting.
