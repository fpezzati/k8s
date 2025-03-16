# FAQ and relevant tests

## Kubeconfig
A `~/.kube/config` example file:
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: ...
  name: cluster1
contexts:
- context:
    cluster: cluster1
    user: cluster_user1
  name: contex1
current-context: contex1
kind: Config
preferences: {}
users:
- name: cluster_user1
  user:
    client-certificate-data: ...
    client-key-data: ...
```
`certificate-authority-data`: cluster CA certificate.
`server`: kube-apiserver endpoint.
`client-certificate-data`: user certificate.
`client-key-data`: user private key.

You can add clusters, users and contexts to bound them. To switch context: `kubectl config use-context context1`.

## Lightning lab

### Upgrade a kubernetes cluster using kubeadm
On controlplane:
```
# telling apt there is a new kubeadm and kubelet
vi /etc/apt/sources.list.d/kubernetes.list
apt update
apt-cache madison kubeadm kubelet
# installing kubelet and kubeadm, kubeadm is mandatory to upgrade cluster
apt-get install -y kubeadm=1.32.0-1.1 kubelet=1.32.0-1.1
# draining node to manage
kubectl drain controlplane --ignore-daemonsets
kubeadm upgrade plan v1.32.0
kubeadm upgrade apply v1.32.0
# once cluster is updated I restart services and kubelet
systemctl daemon-reload
systemctl restart kubelet
# rejoining node with cluster
kubectl uncordon controlplane
```

On node01:
```
vi /etc/apt/sources.list.d/kubernetes.list
apt update
apt-cache madison kubeadm kubelet
apt-get install -y kubeadm=1.32.0-1.1 kubelet=1.32.0-1.1 --allow-change-held-packages
kubectl drain node01 --ignore-daemonsets # run this one from controlplane
kubeadm upgrade node
systemctl daemon-reload
systemctl restart kubelet
kubectl uncordon node01 # run this one from controlplane
```

To prevent downtimes about application on worker nodes:
```
# Identify the taint first.
kubectl describe node controlplane | grep -i taint

# Remove the taint with help of "kubectl taint" command.
kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-

# Verify it, the taint has been removed successfully.
kubectl describe node controlplane | grep -i taint
```
then drain the worker and apply steps to upgrade it.

### Print deployments with custom columns
```
kubectl get deploy -n admin2406 -o=custom-columns=DEPLOYMENT:.metadata.name,CONTAINER_IMAGE:.spec.template.spec.containers[0].image,READY_REPLICAS:.status.readyReplicas,NAMESPACE:.metadata.namespace
```

### Fix admin.kubeconfig


### Create deployment, then manage it
```
kubectl create deployment nginx-deploy --image=nginx:1.16 --replicas=1 -o yaml > nginx-deploy.yml
kubectl set image deployment/nginx-deploy nginx=nginx:1.17
kubectl annotate deployment nginx-deploy kubernetes.io/change-cause="Updated nginx image to 1.17"
```

### Deployment's pod not running
Check expected pvc name in pods and storage class. Right pvc:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-alpha-pvc
  namespace: alpha
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: slow
```

### ETCD backup
```
export ETCDCTL_API=3
etcdctl snapshot save --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379 /opt/etcd-backup.db
```

### Creating pod with given secret
```
apiVersion: v1
kind: Pod
metadata:
  name: secret-1401
  namespace: admin1401
spec:
  containers:
    - name: secret-admin
      image: busybox
      command:
        - sleep
        - "4800" #doublequotes needed because go cannot unmarshall numbers here
      volumeMounts:
        - mountPath: "/etc/secret-volume"
          name: secret-1401-secret
          readOnly: true
  volumes:
    - name: secret-1401-secret
      secret:
        secretName: dotfile-secret
```


## ME1

1. Deploy a pod named nginx-pod using the nginx:alpine image. Once done, click on the Next Question button in the top right corner of this panel.
You may navigate back and forth freely between all questions. Once done with all questions, click on End Exam. Your work will be validated at the end and score shown. Good Luck!

kubectl run nginx-pod --image=nginx:alpine
---
2. Deploy a messaging pod using the redis:alpine image with the labels set to tier=msg

kubectl run messaging --image=redis:alpine --labels="tier=msg"
---
3. Create a namespace named apx-x9984574.

kubectl create namespace apx-x9984574
---
4. Get the list of nodes in JSON format and store it in a file at /opt/outputs/nodes-z3444kd9.json

kubectl get nodes -o json > /opt/outputs/nodes-z3444kd9.json
---
5. Create a service messaging-service to expose the messaging application within the cluster on port 6379.

Use imperative commands.
```
kubectl create service clusterip messaging-service --tcp=6379:6379
kubectl label svc messaging-service tier=msg
kubectl label svc messaging-service app-
```
I failed this.
Commands are wrong, right command was `kubectl expose pod messaging --port=6379 --name messaging-service`. Difference between mine and provided solution is the `spec.selector` dictionary.
To get the same result I should run `kubectl patch service messaging-service -p '{"spec":{"selector":{"tier":"msg"}}}'`. Maybe I'll have to remove other existing selectors too.
---
6. Create a deployment named hr-web-app using the image kodekloud/webapp-color with 2 replicas.

kubectl create deployment hr-web-app --image=kodekloud/webapp-color --replicas=2
---
7. Create a static pod named static-busybox on the controlplane node that uses the busybox image and the command sleep 1000

vi /etc/kubernetes/manifests/static-busybox.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: static-busybox
spec:
  containers:
    - name: static-busybox-container
      image: busybox
      command:
        - sleep
        - "1000"
```

That worked well but solution uses the imperative approach:
```
kubectl run static-busybox --dry-run=client -o yaml--image=busybox --command -- sleep 1000 >/etc/kubernetes/manifests/static-busybox.yaml
```
---
8. Create a POD in the finance namespace named temp-bus with the image redis:alpine

k run temp-bus --image=redis:alpine -n finance
---
9. A new application orange is deployed. There is something wrong with it. Identify and fix the issue
```
k describe pod orange
k get pod orange -o yaml > orange.yaml
(changed command)
k delete pod orange
k create -f orange.yaml
```
I failed this.
Command of second container in pod was wrong `sleeeep 2;`. I should:
- `k get orange -o yaml > orange.yaml`,
- fix command like this:`sleep 2;`,
- `k replace --force -f orange.yaml` (or delete and create -f ).
---
10. Expose the hr-web-app created in the previous task as a service named hr-web-app-service, accessible on port 30082 on the nodes of the cluster.
The web application listens on port 8080
```
k create service nodeport hr-web-app-service --tcp=8080:30082
k label svc hr-web-app-service app-
k label svc hr-web-app-service app=hr-web-app
```
I failed this.
Solution was:
- `k expose deploy hr-web-app --type=NodePort --port=8080 --name=hr-web-app-service --dry-run=client -o yaml > hr-web-app-service.yaml` to get proper yaml,
- refine yaml by adding `nodePort` field with 30082 under ports section,
- `k create -f hr-web-app-service.yaml`
---
11. Use JSON PATH query to retrieve the osImages of all the nodes and store it in a file /opt/outputs/nodes_os_x43kj56.txt.
The osImage are under the nodeInfo section under status of each node.

k get nodes -o=jsonpath='{$.items[*].status.nodeInfo.osImage}' > /opt/outputs/nodes_os_x43kj56.txt
---
12. Create a Persistent Volume with the given specification: -
Volume name: pv-analytics
Storage: 100Mi
Access mode: ReadWriteMany
Host path: /pv/data-analytics

vi pv-analytics.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 100Mi
  hostPath:
    path: /pv/data-analytics
```
k create -f pv-analytics.yaml

## ME2
1. Create a Pod called redis-storage with image: redis:alpine with a Volume of type emptyDir that lasts for the life of the Pod.

Specs on the below.

kubectl run redis-storage --image=redis:alpine --dry-run=client -o yaml > redis-storage.yaml
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: redis-storage
  name: redis-storage
spec:
  containers:
  - image: redis:alpine
    name: redis-storage
    resources: {}
    volumeMounts:
    - mountPath: /data/redis
      name: redis-storage-pv
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: redis-storage-pv
    hostPath:
      type: Directory
      path: /tmp
status: {}
```
Fuck! emptyDir is a volume's attribute! Right manifest is:
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: redis-storage
  name: redis-storage
spec:
  containers:
  - image: redis:alpine
    name: redis-storage
    volumeMounts:
    - mountPath: /data/redis
      name: temp-volume
  volumes:
  - name: temp-volume
    emptyDir: {}
```
kubectl create -f redis-storage.yaml
---
2. Create a new pod called super-user-pod with image busybox:1.28. Allow the pod to be able to set system_time.

The container should sleep for 4800 seconds.
`kubectl run super-user-pod --image=busybox:1.28 --env="system_time=4800" --command  -- sleep $system_time`

I totally miss this. Right manifest:
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: super-user-pod
  name: super-user-pod
spec:
  containers:
  - command:
    - sleep
    - "4800"
    image: busybox:1.28
    name: super-user-pod
    securityContext:
      capabilities:
        add: ["SYS_TIME"]
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```
To set system time is a container's security context we have to declare in manifest.
---
3. A pod definition file is created at /root/CKA/use-pv.yaml. Make use of this manifest file and mount the persistent volume called pv-1. Ensure the pod is running and the PV is bound.

mountPath: /data
persistentVolumeClaim Name: my-pvc
Modified /root/CKA/use-pv.yaml this way:

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: use-pv
  name: use-pv
spec:
  containers:
  - image: nginx
    name: use-pv
    volumeMounts:
    - name: my-pvc-pod
      mountPath: "/data"
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
    - name: my-pvc-pod
      persistentVolumeClaim:
        claimName: my-pvc
status: {}
```

added this pvc:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
```
---
4. Create a new deployment called nginx-deploy, with image nginx:1.16 and 1 replica. Next upgrade the deployment to version 1.17 using rolling update.

`kubectl create deploy nginx-deploy --image=nginx:1.16 --replicas=1`
`kubectl patch deploy nginx-deploy --type='json' -p='[{"op": "replace", "path":"/spec/template/spec/containers/0/image", "value":"nginx:1.17"}]'`
---
5. Create a new user called john. Grant him access to the cluster. John should have permission to create, list, get, update and delete pods in the development namespace .
The private key exists in the location: /root/CKA/john.key and csr at /root/CKA/john.csr.

Important Note: As of kubernetes 1.19, the CertificateSigningRequest object expects a signerName.

Please refer the documentation to see an example. The documentation tab is available at the top right of terminal.

Cannot even approach this.. I have to study this again.

Solution was:
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  signerName: kubernetes.io/kube-apiserver-client
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5NQXNHQTFVRUF3d0VhbTlvYmpDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0VQQURDQ0FRb0NnZ0VCQUt2Um1tQ0h2ZjBrTHNldlF3aWVKSzcrVVdRck04ZGtkdzkyYUJTdG1uUVNhMGFPCjV3c3cwbVZyNkNjcEJFRmVreHk5NUVydkgyTHhqQTNiSHVsTVVub2ZkUU9rbjYra1NNY2o3TzdWYlBld2k2OEIKa3JoM2prRFNuZGFvV1NPWXBKOFg1WUZ5c2ZvNUpxby82YU92czFGcEc3bm5SMG1JYWpySTlNVVFEdTVncGw4bgpjakY0TG4vQ3NEb3o3QXNadEgwcVpwc0dXYVpURTBKOWNrQmswZWhiV2tMeDJUK3pEYzlmaDVIMjZsSE4zbHM4CktiSlRuSnY3WDFsNndCeTN5WUFUSXRNclpUR28wZ2c1QS9uREZ4SXdHcXNlMTdLZDRaa1k3RDJIZ3R4UytkMEMKMTNBeHNVdzQyWVZ6ZzhkYXJzVGRMZzcxQ2NaanRxdS9YSmlyQmxVQ0F3RUFBYUFBTUEwR0NTcUdTSWIzRFFFQgpDd1VBQTRJQkFRQ1VKTnNMelBKczB2czlGTTVpUzJ0akMyaVYvdXptcmwxTGNUTStsbXpSODNsS09uL0NoMTZlClNLNHplRlFtbGF0c0hCOGZBU2ZhQnRaOUJ2UnVlMUZnbHk1b2VuTk5LaW9FMnc3TUx1a0oyODBWRWFxUjN2SSsKNzRiNnduNkhYclJsYVhaM25VMTFQVTlsT3RBSGxQeDNYVWpCVk5QaGhlUlBmR3p3TTRselZuQW5mNm96bEtxSgpvT3RORStlZ2FYWDdvc3BvZmdWZWVqc25Yd0RjZ05pSFFTbDgzSkljUCtjOVBHMDJtNyt0NmpJU3VoRllTVjZtCmlqblNucHBKZWhFUGxPMkFNcmJzU0VpaFB1N294Wm9iZDFtdWF4bWtVa0NoSzZLeGV0RjVEdWhRMi80NEMvSDIKOWk1bnpMMlRST3RndGRJZjAveUF5N05COHlOY3FPR0QKLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
  usages:
  - digital signature
  - key encipherment
  - client auth
```
Then run:
`kubectl create role developer --resource=pods --verb=create,list,get,update,delete --namespace=development`
`kubectl create rolebinding developer-role-binding --role=developer --user=john --namespace=development`

Some expanation. User who wants access cluster needs private key and certificate, that certificate must be signed by cluster internal CA, so you have to provide a CSR (Certificate Signing
Request) with spec.request containing user certificate base64 encoded. Once CSR is added administrator asks cluster to sign certificate in CSR by running command:
`kubectl certificate approve <csr_name_of_choiche>`
To get the crt base64 value to put in CSR manifest you can run command:
`cat user-to-add.csr | base64 | tr -d "\n"`
Once CSR is in place we can create role and rolebinding.
---
6. Create a nginx pod called nginx-resolver using image nginx, expose it internally with a service called nginx-resolver-service.
Test that you are able to look up the service and pod names from within the cluster.
Use the image: busybox:1.28 for dns lookup. Record results in /root/CKA/nginx.svc and /root/CKA/nginx.pod

kubectl run nginx-resolver --image=nginx
kubectl expose pod nginx-resolver --name=nginx-resolver-service --port=80
WTF is dnslookup with busybox:1.28?

Failed. Solution is:
kubectl run nginx-resolver --image=nginx
kubectl expose pod nginx-resolver --name=nginx-resolver-service --port=80 --target-port=80

Then I have to RUN a dns lookup about the service!
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup nginx-resolver-service > /root/CKA/nginx.svc

And run a dns lookup about the pod to get IP address basically!
kubectl get pod nginx-resolver -o wide
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never -- nslookup <ip-address-from-previouscommand>.default.pod.cluster.local > /root/CKA/nginx.pod

ip addres must have '-' in place of '.'.
---
7. Create a static pod on node01 called nginx-critical with image nginx and make sure that it is recreated/restarted automatically in case of a failure.

Use /etc/kubernetes/manifests as the Static Pod path for example.

Failed. Solution was:
`ssh node01`
`kubectl run nginx-critical --image=nginx --restart=OnFailure --dry-run=client -o yaml > /root/nginx-critical.yaml`

Unsure about that.. Reviewing question and solution I think I didn't make any mistake... I placed manifest in /etc/kubernetes/manifests in node01, I still think is the right place.
---
8. Take a backup of the etcd cluster and save it to
/opt/etcd-backup.db

`export ETCDCTL_API=3`
`etcdctl snapshot save /opt/etcd-backup.db --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key`

Failed.
Solution was:
```
ETCDCTL_API=3 etcdctl --enpoints <ip-port-from-etcd-yaml> snapshot save /opt/etcd-backup.db \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key
```
endpoint, key and certs can be found in `/etc/kubernetes/manifests/etcd.yaml`.
---

## ME3
1. Create a new service account with the name pvviewer. Grant this Service account access to list all PersistentVolumes in the cluster by creating an appropriate cluster role
called pvviewer-role and ClusterRoleBinding called pvviewer-role-binding.
Next, create a pod called pvviewer with the image: redis and serviceAccount: pvviewer in the default namespace.

kubectl create clusterrole pvviewer-role --verb=list --resource=pv
kubectl create serviceaccount pvviewer
kubectl create clusterrolebinding pvviewer-role-binding --clusterrole=pvviewer-role --serviceaccount=default:pvviewer

apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pvviewer
  name: pvviewer
spec:
  containers:
  - image: redis
    name: pvviewer
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  serviceAccountName: pvviewer
  automountServiceAccountToken: false
status: {}
---
2. List the InternalIP of all nodes of the cluster. Save the result to a file /root/CKA/node_ips.

Answer should be in the format: InternalIP of controlplane<space>InternalIP of node01 (in a single line)

Failed. Don't get how to get list.
Solution: `kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}' > /root/CKA/node_ips`
---
3. Create a pod called multi-pod with two containers.
Container 1: name: alpha, image: nginx
Container 2: name: beta, image: busybox, command: sleep 4800

Environment Variables:
container 1:
name: alpha

Container 2:
name: beta

```
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: alpha
    image: nginx
    env:
    - name: NAME
      value: "alpha"
  - name: beta
    image: busybox
    command: [ "sleep", "4800"]
    env:
    - name: NAME
      value: "beta"
```
---
4. Create a Pod called non-root-pod , image: redis:alpine

runAsUser: 1000

fsGroup: 2000

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: non-root-pod
  name: non-root-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 2000
  containers:
  - image: redis:alpine
    name: non-root-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Failed in a dummy way. Right manifest:
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: non-root-pod
  name: non-root-pod
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - image: redis:alpine
    name: non-root-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
---
5. We have deployed a new pod called np-test-1 and a service called np-test-service. Incoming connections to this service are not working. Troubleshoot and fix it.
Create NetworkPolicy, by the name ingress-to-nptest that allows incoming connections to the service over port 80.

Important: Don't delete any current objects deployed.

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
spec:
  podSelector:
    matchLabels:
      name: np-test-1
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"

Failed.
Right manifest:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: np-test-1
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
```
I have to study network policies again..
---
6. Taint the worker node node01 to be Unschedulable. Once done, create a pod called dev-redis, image redis:alpine, to ensure workloads are not scheduled to this worker node. Finally, create a new pod called prod-redis and image: redis:alpine with toleration to be scheduled on node01.

key: env_type, value: production, operator: Equal and effect: NoSchedule

`k taint node node01 env_type=production:NoSchedule`

Failed.
Solution was: `kubectl taint node node01 env_type=production:NoSchedule`, exactly as I did and it prevented pod to be deployed on node01.
And to add a manifest to deploy a tolerant pod:
```
apiVersion: v1
kind: Pod
metadata:
  name: prod-redis
spec:
  containers:
  - name: prod-redis
    image: redis:alpine
  tolerations:
  - effect: NoSchedule
    key: env_type
    operator: Equal
    value: production
```
---
7. Create a pod called hr-pod in hr namespace belonging to the production environment and frontend tier .
image: redis:alpine

Use appropriate labels and create all the required objects if it does not exist in the system already.

`k run hr-pod --image=redis:alpine --labels="environment=production,tier=frontend"`

Failed. Forgot to put that pod in the right namespace.
Solution:
`k run hr-pod --image=redis:alpine --labels="environment=production,tier=frontend" -n hr`
---
8. A kubeconfig file called super.kubeconfig has been created under /root/CKA. There is something wrong with the configuration. Troubleshoot and fix it.

Wrong kube-apiserver port
---
9. We have created a new deployment called nginx-deploy. Scale the deployment to 3 replicas. Has the number of replica's increased? Troubleshoot the issue and fix it.

kubectl scale deployment nginx-deploy --replicas=3

Failed.
Command was right, kubectl scale deploy nginx-deploy --replicas=3, but controller-manager wasn't working.
---
10. Configure Horizontal Pod Autoscaler (HPA)
Instructions:

    Create a Horizontal Pod Autoscaler (HPA) for the deployment named api-deployment located in the api namespace.
    The HPA should scale the deployment based on a custom metric named requests_per_second, targeting an average value of 1000 requests per second across all pods.
    Set the minimum number of replicas to 1 and the maximum to 20.

Note: Deployment named api-deployment is available in api namespace.

LEFT UNTOUCHED
kubectl autoscale deployment api-deployment --min=1 --max=20 --dry-run=client -o yaml > hpa-api-deployment.yaml

Create a custom metric:


Modify hpa-api-deployment.yaml to add metrics
---
