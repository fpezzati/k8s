# FAQ and relevant tests

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
