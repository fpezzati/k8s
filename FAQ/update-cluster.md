## How to upgrade the cluster
You have to upgrade your cluster to newer version, let that be 1.32.0. Proceed to minimize downtime for hosted services, use kubeadm.

Start from master nodes, one node at a time, then proceed upgrading worker nodes. Apply steps for each master and worker node.

### On master nodes: step 1
Tell apt about new kubeadm and kubelet versions by updating apt with desired version:
`vi /etc/apt/sources.list.d/kubernetes.list`

Then update apt:
`apt update`

Check for issues about kubeadm and kubelet:
`apt-cache madison kubeadm kubelet`

Then proceed with kubeadm and kubelet installation:
`apt-get install -y kubeadm=1.32.0-1.1 kubelet=1.32.0-1.1`

### On master nodes: step 2
Drain the node:
`kubectl drain master01 --ignore-daemonsets`

Check for issues about cluster upgrade:
`kubeadm upgrade plan v1.32.0`

Then proceed with upgrade:
`kubeadm upgrade apply v1.32.0`

Once node is updated, restart services and kubelet:
`systemctl daemon-reload`
`systemctl restart kubelet`

Rejoin node with cluster:
`kubectl uncordon master01`

### On worker nodes: step 1
Tell apt about kubernetes newer version:
`vi /etc/apt/sources.list.d/kubernetes.list`

Then update apt:
`apt update`

Check for issues about kubeadm and kubelet:
`apt-cache madison kubeadm kubelet`

Then proceed with kubeadm and kubelet installation:
`apt-get install -y kubeadm=1.32.0-1.1 kubelet=1.32.0-1.1`

### On worker nodes: step 2
Drain the node, command must be executed from a master node:
`kubectl drain worker01 --ignore-daemonsets`

Proceed with upgrade. Upgrade worker node doesn't need any version, worker node will rely on masters to get which version to upgrade to:
`kubeadm upgrade node`

Once worker node is upgraded, proceed by restarting services:
`systemctl daemon-reload`
`systemctl restart kubelet`

Now it is time to uncordon the worker. Go to a master node and run:
`kubectl uncordon worker01`

### Final notes
To prevent downtimes about application on worker nodes, identify nodes topology and taints:
`kubectl describe node master01 | grep -i taint`
Proceed a sub-type of nodes at a time: upgrade all the database nodes, then all the nodes hosting an application, then others hosting another application and so on. This way pod will be rescheduled on nodes that can host them.

Eventually remove taints:
`kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-`
and apply them again later.
