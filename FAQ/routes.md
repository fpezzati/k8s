## Network routes and Gateway API
>Create an HTTPRoute resource named web-route in the default namespace. This route should direct traffic from the >web-gateway to a backend service named web-service on port 80.
>
>web-gateway is in the nginx-gateway namespace. And service web-service and deployment web-app are in the default >namespace.
>
>Is the web-route deployed to as HTTP route?
>
>Is the route configured to gateway web-gateway?
>
>Is the route configured to service web-service?

HTTPRoute is a Gateway API object for implementing HTTP based behavior. HTTPRoute has the following structure:
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: default
spec:
  parentRefs:
    - name: web-gateway #gateway name to which the route will be attached to
      namespace: nginx-gateway #gateway namespace
  rules:
    - backendRefs:
        - name: web-service #service the route will bring traffic to
          port: 80 #service port
```
