## How to create user and grant him access to cluster
Suppose you have to add a new user, hohn, to cluster and grant him access.
User john should have permission to create, list, get, update and delete pods in the 'development' namespace.

### Step 1
User john must provide private key and certificate sign request:
```
openssl req -newkey rsa:4096 -keyout john.key -out john.csr
```
### Step 2
Kubernetes cluster hosts its own CA so john's certificate signing request must be signed by that to get access to cluster:
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-csr
spec:
  signerName: kubernetes.io/kube-apiserver-client
  expirationInSeconds: 86400 # not mandatory
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5NQXNHQTFVRUF3d0VhbTlvYmpDQ0FTSXdEUVlKS29aSWh2Y05BU...
  usages:
  - client auth
```
Beware: `spec.request` value is the content of john.csr base64 encoded, to get it encoded and without spurious "\n" at file's end, use command:
```
cat john.csr | base64 | tr -d "\n"
```

Create the csr by using the manifest, `kubectl create -f john-csr-manifest.yaml`.
### Step 3
Command `kubectl get csr` will show `john-csr` in a `Pending` state, cluster admin can approve that csr by command `kubectl approve john-csr`.

Now we grant privileges to user john by command
```
kubectl create role developer --resource=pods --verb=create,list,get,update,delete --namespace=development
```
and bind that role to user john:
```
kubectl create rolebinding developer-role-binding --role=developer --user=john --namespace=development
```
Admin will provides john's certificate signed by cluster CA and user john can finally fill his `.kube/config` this way:
```
apiVersion: v1
kind: Config
clusters:
- name: dev-cluster
  cluster:
    certificate-authority-data: # copy content of cluster CA certficate here
    server: https://dev-cluster-kube-apiserver:6443
users:
- name: john
  user:
    client-certificate-data: # copy john certificate signed by cluster CA here
    client-key-data: # copy content of john.key file here
contexts:
- context:
    cluster: dev-cluster
    user: john
  name: context1
current-context: context1
preferences: {}
```
Done.
