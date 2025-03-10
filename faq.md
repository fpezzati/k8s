# FAQ and relevant tests

## Lightning lab

### Upgrade a kubernetes cluster using kubeadm
Start from draining each controlplane node one at a time, then drain a worker at a time and upgrade, then join the node again.

Can't upgrade kubeadm from 1.31.x to 1.32.y...

On each node:
```
# tells apt that newer kubernetes version exists:
vim /etc/apt/sources.list.d/kubernetes.list
# modify file's entry this way:
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /

kubectl drain controlplane --ignore-daemonsets
apt update
apt-cache madison kubeadm #wtf is 'madison'? 'madison' tells existing version of given package...

apt-get install kubeadm=1.32.0-1.1

kubeadm upgrade plan v1.32.0
kubeadm upgrade apply v1.32.0
```

Forgot to install kubelet 1.32.0-1.1 and
```
systemctl daemon-reload
systemctl restart kubelet
```
then uncordon the node.

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

Worker nodes need this command: `kubeadm upgrade node`, without `apply v1.32.0` because they rely on controlplane!!

Join each node again with `kubectl uncordon <node>`.
