# Have a break: install a cluster
kubeadm is the main player.
On each node:
- setenforce 0 (what is SELINUX?)
- (disable on system restart) vi /etc/sysconfig/selinux: set `SELINUX=permissive` instead of enforcing (or whatever value has),
- vi /etc/hosts: add ip and name of each node in the cluster, master or worker (do I really have to do that?),
- (disable firewall if there is any active) systemctl stop firewalld,
- install kubeadm and docker,
I think I have already did these steps

What is SELINUX? An additional security layer RH added to its distributions. Debian like OS are SELINUX free.

On master node:
- enable and start kubelet: `systemctl enable kubelet && systemctl start kubelet`,
- enable and start docker: `systemctl enable docker && systemctl start docker`,
- disable swap before install kubeadm: `swapoff -a`,
- create cluster: `kubeadm init`. It will do a lot of things,
At the end of init process, kubeadm will tell which command to run:
- `mkdir -p $HOME/.kube`,
- `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`,
- `sudo chown $(id -u):$(id -g) $HOME/.kube/config`.
More important: kubeadm will tell which command to run to join nodes to the cluster:
- `kubeadm join masternode.ip:masternode.port --token some.weird.symbols.sequence.as.token --discovery-token-ca-cert-hash a.more.weird.symbols.sequence.as.token`
Now I have to deploy a basic startup items on cluster:
- `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=k8s.version.of.choice"`. This will deploy a network and a lot more.

On worker nodes:
- enable and restart docker service: `systemctl enable docker && systemctl start docker`,
- join the worker to cluster by using command kubeadm told when we do init on master: `kubeadm join masternode.ip:masternode.port --token some.weird.symbols.sequence.as.token --discovery-token-ca-cert-hash a.more.weird.symbols.sequence.as.token`

Take in account ./k8s/RPiCluster/ansible/k8s.cluster.roles files.

Missing:
- `sudo echo -n 'cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1' >> /boot/cmdline.txt`, but file already has a newline char at the end.. I had to manage that by hand,
- `sudo systemctl disable dphys-swapfile.service`, to disable memory swap.
Both commands need reboot.

Done! I got a master!

To start using your cluster, you need to run the following as a regular user:
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Alternatively, if you are the root user, you can run:
```
  export KUBECONFIG=/etc/kubernetes/admin.conf
```
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root. To retrieve join command:
```
kubeadm token create --print-join-command
```
