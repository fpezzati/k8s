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

Do I have to rely on loadbalancer to reach gateway controller from outside?

## Routes
A route identify a way to forward a request hitting a gateway to a specific service in cluster.

There are a few types of route: HTTPRoute, TLSRoute, TCPRoute, UDPRoute, GRPCRoute.

Routes are attacched to gateways in a N to M form.

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
