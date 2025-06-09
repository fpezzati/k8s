# Network policies
I am bad at those.

What I know:
- they implement network traffick by enablig/disabling communication between pods,
- never see but you can use selector to do network traffic among different sets of pods. For sure,

What I don't know
- how to shut down everything expect my routes?

## Quick summary from my notes
K8s ensure that all pods can comunicate with each other, even if they're on a different node or namespace. Network policies are here to change that.

Don't use Flannel if you want to use networkpolicies, he doesn't support them.

If you don't know which cni is currently used, you can `ls /etc/cni/net.d/` and check which cni installed files are there.

Networkpolicy example here:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      somekey: somevalue
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            someotherkey: someothervalue
      ports:
      - protocol: TCP # do we need anything else? UDP? Some QoS?
        port: 3306
```
She allows incoming traffic that came from pods labeled as `someotherkey: someothervalue` to pods (see the `ingress` word) labeled as `somekey: somevalue`.

Of course `podSelector` is the main tool about deciding 'who can reach who' in network policies but there are other options using:
- ports,
- namespaceSelector,
- ipBlock,
- dns (well it relies on namespaceSelector),
- who knows..

Network policies are applied altogether, there is no order, this mean you can get weird behavior in a few rules. Always check which rules are working, run `kubectl get networkpolicies -A`, and try not to overlap them.

### Allow all ingress traffic
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}
```

### Deny all ingress traffic
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default  # The namespace this policy will be applied to
spec:
  podSelector: {}  # Selects all pods in the namespace
  policyTypes: # Block all traffic
    - Ingress
```

### Deny all egress traffic to pods labeled as 'app: secure-app'
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: secure-app
  policyTypes:
  - Egress
  egress: []
```

### Tryin' to debunk official documentation example
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```
