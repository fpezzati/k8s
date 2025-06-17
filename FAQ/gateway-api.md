# Gateway API
Focuses on L4 and L7

Replaces Ingress

Like Ingress, there is no ready Gateway controller to use, you choose the one you want and install it by CRD.

Made of:
- routes,
- gateways,
- gateway classes

## Gateway classes
Define a set of configurations about a gateways. You must at least install one to have a functioning gateway.

Gateway classes will be provided by infrastructure providers.

Gateway controller references one gateway class. The gateway class brings the kind of gateway controller.

## Gateway
Describes how traffic must be translated and moved to services within the cluster.

Gateway may be attacched to one or more routes.

Gateway example:
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: foo-gateway
  namespace: gateway-api-example-ns1
spec:
  gatewayClassName: foo-lb
  listeners:
  - name: prod-web
    port: 80
    protocol: HTTP
    allowedRoutes:
      kinds:
      - kind: HTTPRoute
      namespaces:
        from: Selector
        selector:
          matchLabels:
            # This label is added automatically as of K8s 1.22
            # to all namespaces
            kubernetes.io/metadata.name: gateway-api-example-ns2
```
Gateway are namespaced objects.

Gateway can also handle hostnames and TLS of course:
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tls-basic
spec:
  gatewayClassName: example
  listeners:
  - name: foo-https
    protocol: HTTPS
    port: 443
    hostname: foo.example.com
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: foo-example-com-cert
  - name: bar-https
    protocol: HTTPS
    port: 443
    hostname: bar.example.com
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: bar-example-com-cert
```
Here you can see gateway has two listeners one for domain 'foo.example.com' and another for 'bar.example.com' configured with two different certificates, wrapped in two secrets.

Do I have to rely on loadbalancer to reach gateway controller from outside? No, gateway class is a LoadBalancer.

Oh! That is gross! You can get the **RIGTH** gateway api doc page [here](https://gateway-api.sigs.k8s.io/guides), which is reachable by the kubernetes's search page by looking for 'gateway api' and then click on the first link in the page, the one with text 'Gateway API'.

## Routes
A route identify a way to forward a request hitting a gateway to a specific service in cluster.

There are a few types of route: HTTPRoute, TLSRoute, TCPRoute, UDPRoute, GRPCRoute.

Routes are attacched to gateways in a N to M form.
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginxhello
  namespace: nginxhello
spec:
  parentRefs:
    - name: http-gw
  hostnames:
    - 172-18-0-3.nip.io     # to be replaced with your EXTERNAL-IP
  rules:
    - backendRefs:
        - name: nginxhello
          port: 80
```
Routes are namespaced objects.

Route anatomy:
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-portal-httproute
  namespace: cka3658
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  hostnames:
  - "cluster2-controlplane"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-portal-service-v1
      port: 80
      weight: 80
    backendRefs:
    - name: web-portal-service-v2
      port: 80
      weight: 20
```
Any route is bound to a gateway object, that's `spec.parentRefs`.

You can use `spec.hostnames` to specify one or more (or one that uses '*' wildcard, but you must put this on top of `hostnames`). When `spec.hostnames` is present, then http request's hostname header must match with one of hostnames specified there. If no `spec.hostnames` is specified, then no match is asked for.

`spec.rules` is another important part. Here you can specify how an http request is routed to service. If no service is specified for a rule, then requests that match that rule will be discarded and you'll get back a 404 error code. `path`, `method`, `headers` are all type of matches a rule can express. You can also combine them.
