# CKA

## Core concepts

### Cluster architecture
Worker nodes host containers, master nodes implement the control plane. They both need a container engine, it usually is docker.


Components of a control plane are:
- ETCD: stores info about nodes and containers.
- kube-scheduler: a component that stays with the contol plane and are in charge to load containers on worker nodes. kube-scheduler choose the node by verifying container requirements and node related constraints (node capacity, hardware or other user and environmental constraints).
- controller-manager, node-controller, replication-controller: they manage communication between nodes, nodes status, containers status. They check the cluster fits user's requirements.
- api-server: is in charge to keep in touch all the control plane's components. It orchestrates the control plane components.

Components of a worker node are:
- kubelet: an agent running on each node. Listens for instructions from the api-server, deploys or destroys containers on node.
- kube-proxy: allows comunication between containers

### Docker vs ContainerD
Kubernetes conforms with the OCI specs by defining a ContainerRuntimeInterface, or CRI; this means that kubernetes can operate with every container runtime who complains with OCI and so with CRI.

ContainerD is part of Docker itself, it is a complete project itself, a container runtime which complains with OCI. You can install ContainerD with Docker or by itself. ContainerD comes with a tool, `ctr`, to perform container activities. It is not that frendly, `nerdctl` is far better, quite similar to Docker cli but with some features added.

`crictl` is another CLI to interact with CRI compliant runtimes. Not that good, quite a debugging tool.

### ETCD
Distributed key-value store. etcd is the daemon (listening on port 2379 by default), etcdctl is the default client.
`./etcdctl set key1 value1` stores... you get it, don't you? ETCD uses gRPC (RPC? in 2021?).

Stores info about the whole cluster, its status and every change. ETCT is deployed by kubeadm.

### kube apiserver
It is the k8s main component. When running a kubectl command, this command reach the kube apiserver, apiserver validates the request and fetch/push data from/to ETCD component. It is possible to interact directly with apiserver too.

When user wants to deploy a POD:
- user asks to apiserver,
- apiserver validates request and write some info on ETCD,
- kube-scheduler, which is aware of some ETCD's segment of data, choose the right node and tells the apiserver he wants to deploy that POD on that worker node,
- apiserver move the kube-scheduler request to the indicated node's kubelet,
- kubelet deploys the POD, then tells apiserver the POD has been deployed,
- apiserver listen the info about the new POD and push the info on ETCD to update cluster status: cluster has one more POD.

apiserver is aware about what's going on the cluster and he constantly update cluster's status on ETCD. He is the only one that pushes data on ETCD.

kubeadm deploys the apiserver too.

apiserver configuration can be seen at `/etc/kubernetes/manifests/kube-apiserver.yaml`, if apiserver is bootstrapped by kubeadm, or at `/etc/systemd/system/kube-apiserver.service`, if is installed by hands.

### kube controller-manager
Is in charge to monitor status of every components in clusters.

node-controller checks node status every short of time by using the apiserver. If node-controller does not get a node's heartbeat for a while, he marks that node as unreachable.

An unreachable node has 5 minutes to come back. After that while, pods or other hosted components are 'moved' to healtly nodes.

replication-controller is in charge to monitor the replica-sets in the cluster. He is also in charge to check if the number of pods bound to a replica-set are in the right number.

There are a lot more controllers in k8s, more or less one each component can be deployed in cluster. They all are in the kube-controllermanager. kube-controllermanager is deployed by kubeadm or can be installed manually. If kube-controllermanager is deployed by kubeadm, his configuration is in `/etc/kubernetes/manifests/kube-controller-manager.yaml`, or if he has been installed manually, in `/etc/systemd/system/kube-controller-manager.service`

### kube-scheduler
Decides which pod goes on which node. Given a pod, scheduler filter nodes to find proper candidates, then ranks the candidates. The better rank indicates which node will hosts the pod.

scheduler can be installed manually or by kubeadm, however configuration is in `/etc/kubernetes/manifests/kube-scheduler.yaml`.

### kubelet
Is the joint between cluster and the node. He monitors the state of each pods, he deploys pods when apiserver asks to, reports periodically to apiserver about node status.

kubelet must be installed manually.

### kube-proxy
Each pod can reach the others in cluster by using a pod network. kube-proxy is in charge to monitor the network and provide routes to reach every pod. kube-proxy manage services in the node too.

Definition of service: an abstraction, which label a logical set of pods in order to get them reachable even if they are new or destroyed or respawn. Usually, service provides some sort of facade to all the pods marked by a certain selector. Of course there are other uses for services..

kube-proxy can be downloaded or bootstrabbed by kubeadm.

### pods
k8s orchestrates containers shipping an app among a set of nodes. Containers aren't shipped directly, but encapsulated in pods. Pods is an instance of an application, pods are one to one with containers type, a pod can host multiple containers but they all are running different images. In pods containers share network and disk.

kubectl can deploy pods. Run `kubectl run nginx --image nginx` and a new pod will be started with nginx as container's image. Pods can be deployed by using .yml configuration file.

A pod's .yml file root elements:
```
apiVersion:
kind:
metadata:

spec:
```
kind/apiVersion allowed values are:
| kind | version |
|:----:|:-------:|
| POD | v1 |
| Service | v1 |
| ReplicaSet | apps/v1 |
| Deployment | apps/v1 |

metadata can:
```
apiVersion:
kind:
metadata:
  name: my-app
  labels:
    app: myapp
spec:
```
name is a label that marks a pod or a set of pods of that kind.

labels is a dictionary, any key-value pair can be stored.

specs section:
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    type: sometype
spec:
  containers:
    - name: mysimpleapp
      image: fpezzati/simpleexpressapp
```
handles info about which containers to wrap into the pods created by this file. Run `kubectl create -f mypods.yml` to instruct cluster to create pods and host containers.

Pods can be listed by running `kubectl get pods` and specific pod configuration can be read by running `kubectl describe pod my-app` (was the pod metadata's name attribute value in the previous example).

Command `kubectl edit pod podname` allows to edit a pod without editing its .yml and reapply. Not all the attributes are editable, but only:
- `spec.containers[*].image`,
- `spec.initContainers[*].image`,
- `spec.actoveDeadòomeSecpmds`,
- `spec.tolerations`,
other updates will be rejected on save.

Used commands in tests:
- kubectl run nginx --image=nginx,
- kubectl get pods (-o wide to get some more infos),
- kubectl describe pod pod-name,
- kubectl describe pod pod-name | grep property-name,
- kubectl delete pod pod-name,
- kubectl run nginx --image=inginx --dry-run=client -o yaml > pod.yaml (allows to create the .yaml definition to recreate the same pod instead of running that one , very useful. The 'dry-run' options prevent object creation, only the .yaml file will be produced),
- kubectl edit pod nginx (allows to edit the pod's running configuration).

### replicationcontroller and replicaset
replicationcontroller is a controller. Replicasets allow HA application by replicating pods in node or automatically provide a new pod if the lone one crashes, even among nodes. Deprecated by replicaset. replication-controller may be defined as:
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-rc
  labels:
    app: myapp
    type: sometype
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: sometype
    spec:
      containers:
        - name: mysimpleapp
          image: fpezzati/simpleexpressapp
  replicas: 3
```
`template` section is where to specify what has to be replicated. In the case above, `template` contains the previous pod .yaml definition without `apiVerion` and `kind`. When running command `kubectl delete replicationcontroller rc-name` replication controller and all the pods he spawn are removed.

replicaset aims the same objectives as replicationcontroller. replicaset may be defined by:
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-rs
  labels:
    app: myapp
    type: sometype
spec:
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: sometype
    spec:
      containers:
        - name: mysimpleapp
          image: fpezzati/simpleexpressapp
  replicas: 3
  selector:
    matchLabels:
      type: sometype
```
`selector` attribute allows replicaset to manage pods that were not defined in the same file. Having a bunch of existing pods, user can define a replicaset that applies to all of those pods satisfying the `selector` indication, or, in other words, all the pods that have a matching `label` attribute or satisfy the `selector` criterias, this is a big advance. replicaset is in charge to spawn pods when existing pods are less than indicated by the `replicas` attribute value.

User may change the number of replicas by updating the .yml file and running it again or using commands:
- `kubectl scale --replicas=6 -f replicaset.file.definition.yml`,
- `kubectl scale --replicas=6 replicaset replicaset.to.scale`.
File won't be updated.

Used commands in tests:
- kubectl edit replicaset replicaset.name,
- kubectl get replicaset -o wide,
- kubectl delete --all pods.

### deployments
An object representing a desire state of pods and replicasets.

Definition example:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name:
  labels:
    app: myapp
    type: sometype
spec:
  template:
    metadata:
      name: mypod
      labels:
        app: myapp
        type: sometype
    spec:
      containers:
      - name: simpleexpressapp
        image: fpezzati/simpleexpressapp
replicas: 3
selector:
  matchLabels:
    type: sometype
```

Used commands in test:
- kubectl get deployments,
- kubectl apply -f deployment-definition.yaml,
- kubectl create deployment http-frontend --image=httpd:2.4-alpine | kubectl scale deployment --replicas=3 http-frontend

### namespaces
namespaces are labels that define sets of k8s components. k8s itself has three namespaces: kube-system, default, kube-public. Resources (CPU, RAM, storage) can be bound to a namespace. Components in the same namespace can address each other by using their name (it resolves as hostname). Addressing components from outside a namespace requires to append namespace to address: `component-name.namespace.svc.cluster.local`.
`cluster.local` is the k8s's default domain, `svc` is the subdomain for services.

Create a component with the `--namespace` option to put him in that namespace: `kubectl create -f pod.yaml --namespace=sandbox`.

namespace definition example file:
```
apiVersion: v1
kind: Namespace
metadata:
  name: somename
```

It is also possible to create a namespace by command: `kubectl create namespace namespace.name`.

To switch to a specific namespace: `kubectl config set-contex $(kubectl config current-context) --namespace=somename`.

namespaces may also require resources that could be listed in a ResourceQuota component:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: somequota
  namespace: somename
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```
the `hard` part indicates the limit namespaces can reach in terms of resources.

Commands used during test:
- kubectl get namespaces (or kubectl get ns),
- kubectl get pods --namespace=somenamespace (or kubectl get pods -n somenamespace),
- kubectl run redis --image=redis --namespace=somenamespace,
- kubectl get pods -o wide --all-namespaces,
- kubectl get svc -n dev (kubectl returns all the services in namespace dev).

## Services
Enable comunication between cluster's components and components and the world outside.

Each node in the cluster has an IP, we can always ssh into a node.

Service came in several flavour:
- NodePort,
- ClusterIP,
- LoadBalancer,

### NodePort
Opens a port into the node, the NodePort, and port inside the node, simply called Port, which points to a specific pod's port, the targetPort. NodePort has an IP internal to the cluster.

NodePort definition file example:
```
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: myapp
    type: sometype
```
`nodePort` value must be in the allowed range. If `port` is not mentioned then its value will be the same as `targetPort`. The `selector` attribute is crucial here because the NodePort must be aware about at which pod route messages on port `port`. If that pod is not uniquely identified, NodePort will forward the message to each pod acting as a load balancer. If pods are on different nodes, then NodePort became a sort of cross-nodes-service applying the same behavior on each node so pod is reachable in the same way on each node, only the IP address changes of course.

### ClusterIP
ClusterIP service allows pods to comunicate each other. Having a ClusterIP selecting to some label, allows a pod to comunicate to one of the pod that has that label. No target pod IP is required, if target pod goes down sender port can still interact with ClusterIP while target port is respawned, or, if more than one pod satisfy ClusterIP's selector, just another pod. ClusterIP allows pods to scale without impacting on comunication.

ClusterIP definition example:
```
apiVersion: v1
kind: Service
metadata:
  name: persistence-layer
spec:
  type: ClusterIP
  ports:
    - targetPort: 80
      port: 80
  selector:
    app: myapp
    type: sometype
```

### LoadBalancer
Works only on supported cloud platforms.

Provide a way to harmonize url to cluster's node.

LoadBalancer definition example:
```
apiVersion: v1
kind: Service
metadata:
  name: persistence-layer
spec:
  type: LoadBalancer
  ports:
    - targetPort: 80
      port: 80
  selector:
    app: myapp
    type: sometype
```

Commands used during test:
- kubectl get svc -o wide,
- kubectl describe svc service-name,
- kubectl get deployments,
- kubectl expose deployment simple-webapp-deployment --name=webappservice --target-port=8080 --type=NodePort --port=8080 --dry-run= client -o yaml > svc.yaml (you get the service .yaml file, later you have to edit it to tell which port open to the outside).

## imperative vs declarative approach in k8s
Imperative way:
- to give detailed instructions about how to achieve an objective (install this, run that, copy that one there...),
- if something goes wrong a set of additional instructions are necessaries to handle such event,
- use kubectl directly to handle components. Even using kubectl to create by .yml file is imperative. Everything I did this far is imperative,

Declarative way:
- to say what the objective is, then delegate to the system or tool or whatever,
- if something goes wrong hope system is smart enought to recover or to graceful presents error to caller,
- kubectl apply command is declarative. Given the .yml file as objective, it tells k8s to do necessary changes to update the cluster to what the file states. No command is told.

Imperative commands are useful to manage and accomplish tasks on a cluster but, functionality is limited (to the command itself, so prepare to give a lot of command or to combine them in complicated) and they will be forgotten (only history command can tell which commands have been launched to produce a cluster state). Imperative commands are good to short distance objectives or for scout a solution (remember the `--dry-run=client` option).

Declarative commands rely on .yml or manifest or something that has been wrote down because they ask for the change specified in the file. k8s try to apply the file in a idempotent way doing the only changes that are mandatory. Moreover you do this by relying on a file which can be versioned.. This is the real streght: idempotence and versioning.

Commands used during test:
- kubectl run nginx-pod --image=nginx_alpine,
- kubectl delete pod nginx-pod,
- kubectl run nginx-pod --image=nginx:alpine,
- kubectl run redis --image=redis:alpine --dry-run=client -o yaml > redis.pod.yml,
- kubectl run redis --image=redis:alpine --labels=tier=db,
- kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml > redis.clusterip.yml,
- kubectl expose pod redis --name redis-service --port 6379 --target-port 6379,
- kubectl create deployment webapp --image=kodekloud/webapp-color --replicas=3
deployment.apps/webapp created,
- kubectl run custom-nginx --image=nginx | kubectl expose custom-nginx --port=8080,
- kubectl expose pod custom-nginx --port=8080,
- (as alternative to the two previous commands) kubectl run custom-nginx --image=nginx --port 8080),
- kubectl create deployment redis-deploy --image=redis --namespace=dev-ns --dry-run=client -o yaml,
- kubectl run httpd --image=httpd:alpine --port=80 --expose --dry-run=client -o yaml > something.yml.

### kubectl apply command
Allows handling objects in a declarative way. Command:
- picks .yml configuration and transforms it in a json doc,
- matches it agains the living configuration status of the same object,
- every difference is applied.

Living object configuration is in k8s memory. It also has the json doc representing latest object creation configuration.

## About scheduling

### manual scheduling
Every pod has the attribute `nodeName` not valued by default in `spec`. Scheduler looks for `nodeName` attribute in pods. If it is not valued, then scheduler choose a node and deploy the pod there by setting pod's `nodeName` value as the node name.

Scheduler is deployed in cluster as pod: `kube-scheduler`.

Manual scheduling is fill that attribute with the name of the node in which we want to deploy that pod or, to mimick scheduler more precisely, create a bind object. Bind object can be defined this way:
```
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node-of-choiche-name
```
Now to complete operation is mandatory to make a http POST indicating as payload the Binding object in a json format:
```
curl --header "Content-Type:application/json" --request POST --data '{

"apiVersion": "v1", "kind": "Binding", "metadata": { "name": "nginx" }, "target": { "apiVersion": "v1", "kind": "Node", "name": "node-of-choiche-name" } }' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
```
When no scheduler is present, a newly created pod will fall in status pending and will stay until a scheduler is created or a node is manually assigned to him.

Oh, `kubectl apply -f ...` does not work if no node was associated to the .yaml.

### labels and selectors
Labels are a standard method to group components together. Labels can be defined in `metadata.labels`. The `selector.matchLabels` key/value defined in `spec` are used to find a specific group on which apply some feature, for example replication by replicaset.

Selectors are a way to define a certain group of components given a set. Selectors can be used by kubectl: `kubectl get pods --selector somelabel=somevalue`.

If a ReplicaSet cannot find any component it is not created. The same for Service.

Annotations are some sort of special labels. As labels can be used to attach arbitrary metadata to components, but they are not identifying, a selector cannot use them to identify a component.

Commands used during test:
- kubectl get pods --namespace=dev,
- kubectl get all --selector env=prod,
- kubectl get pods --selector env=prod,bu=finance,tier=frontend (is the same to use -l instead of selector, i.e.: kubectl get pods -l env=prod,bu=finance,tier=frontend),
- kubectl get pods -l 'environment in (production, qa)' (it puts values in OR combination).

### Taints and tolerations
Taints and tolerations are tools to set restrictions about which kind of pods can be scheduled on a node. A node is tainted about blue will hosts only blue tolerant pods and scheduler will not schedule pods on that node unless he will find a tolerant blue pod. Taint applies on node, toleration applies on pods.
```
kubectl taint nodes node-name key=value:taint-effect
```
taint is implemented by the `key=value` pair. `taint-effect` tells what happens to a non tolerant pod:
- NoSchedule: pod is not scheduled on node,
- PreferNoSchedule: does not guarantee that pod will not be scheduled,
- NoExecute: pod is scheduled but other non tolerant pods are immediately removed from node if not tolerant. Otherwise, tolerant pods will stay.

Pretty messy...

Example:
```
kubectl taint nodes node1 app=frontend:NoSchedule
```
Tolerations can be defined by `spec.tolerations` attribute in pod's definition:
```
apiVersion: v1
kind: Pod
metadata:
  name: heyyou
spec:
  containers:
  - name: alpinebox
    image: alpine
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
```
Taints and tolerations do not say which pod will be scheduled on which node, but which pod will be not scheduled on which node. Master nodes have a taint to prevent application pod to be scheduled into.

Taint has an explicit imperative command while toleration not, by using taint on a pod a tolerance is added instead, so taint differs if is run against a node or a pod.

Commands used during test:
- kubectl describe node node01 | grep taint,
- kubectl taint nodes node01 spray=mortein:NoSchedule,
- kubectl run mosquito --image=nginx,
- kubectl edit node controlplane,
- kubectl taint node master node-role.kubernetes.io/master:NoSchedule- (removes the taint thanks to symbol `-` at the end of command).

### Node selectors
Nodes can be specific crafted for some tasks, then is desirable that not all the pods can land on that nodes. Node selectors are tailored to achieve this. Node selector can be specified in pod .yaml file:
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myappcontainer
    image: myappimage:v1
  nodeSelector:
    size: Large
```
the `nodeSelector` keep a map of label:values.

To label a node use command `kubectl label nodes node01 size=Large`, and the node will host pods having a `spec.nodeSelector.size: Large` entry.

Node selectors are easy but not that powerful: you can express 'exact match like' constraints only, not OR or NOT expressions.

### Node affinity
Node affinity aims the same objective than node selector but is far more powerful.

Let's rewrite the same node selector constraint by using node affinity:
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myappcontainer
    image: myappimage:v1
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
```
Some allowed operators: `In`, `NotIn`, `Exists` (checks label not value)...

`required`: means in a mandatory way, satisfy node's labels or go away pod.
`preferred`: means that scheduler must try its best to schedule, if no best solutions are available then land pod here.
`DuringScheduling`: means the pod must satisfy node's labels when he is created for the first time.
`DuringExecution`: means the pod must satisfy node's labels while running on it. I a change on node invalidates a pod, it will be removed from the node.

`affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution` tells that pod must satisfy labels the first time, then he may stay here even if node's labels change.

`affinity.nodeAffinity.preferredDuringSchedulingIgnoredDuringExecution` tells that pod may land on the node, if no better option is available, but if pod does not satisfy node's labels changes then it will be removed.

`affinity.nodeAffinity.requiredDuringSchedulingRequiredDuringExecution` is planned in the future.

Commands used during test:
- kubectl label node node01 color=blue,
- kubectl create deployment blue --image=nginx --replicas=3,
- kubectl describe node node01,
- kubectl get nodes node01 --show-labels,
- kubectl create deployment blue --image=nginx | kubectl scale deployment blue --replicas=3,
- kubectl get deployment blue -o yaml > blue.yaml (and then edit the file to set up affinity and nodeAffinity),
- kubectl create deployment red --image=nginx --replicas=3 | kubectl edit deployment red (and add affinity.nodeAffinity to deployment's spec.template.spec attribute).

### Node affinity vs taints and tolerations
Taints and tolerations tell which pods a node won't accept. Node affinity tells which pods a node will accept. They both are insufficient to be sure that a specific pod will go into a specific node only and that node will accept that kind of pods only.
To taint a node and provide a toleration to a pod does not prevent that pod to be scheduled on another node.
Setting an affinity will ensure that a specific pod (labeled to match the affinity) will be scheduled on a specific node, but it won't prevent other pods to be scheduled on that specific node.
Node affinity and taints and tolerations can be combined to ensure that only some specific pods will be scheduled only on some specific nodes.

### Resource requirements and limits
Each pod requires resources: cpu, ram and disk. Scheduler is in charge to deploy a pod on a node by verifying node's resources. If resources are sufficient node will accept pod, otherwise pod will be scheduled on another node. If no node is able to host the pod, then pod is left in a 'pending' state and not executed.

k8s assumes each pod requires 0.5 cpu, 256Mi (Mebibyte, damn) ram if no specific value is provided. Resources can be specified in `spec.containers.resources.requests` pod's attribute, i.e.:
```
apiVersion: v1
kind: Pod
metadata:
spec:
  containers:
  - name: dummy-app
    image: dummy-app-image
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "1G"
        cpu: 1
```
cpu value may range from 0.1 (or 100m) to whatever. To make a dummy comparison, 1 cpu means 1 AWS vCPU more or less.

To express RAM size: 1G (gigabyte), 1M (megabyte), 1K (kilobyte), 1Gi (gibibyte), 1Mi (mebibyte), 1Ki (kibibyte).

k8s set a limit to 1 cpu and 512 Mi ram if no limit has been specified. However a container can ask for more resources (ram or cpu) than pod's limit says. Containers cannot get more cpu than pod limit if it consumes all the available, but if containers try to catch all the ram pod has, then that pod will be terminated.

Command used in test:
- kubectl describe pod podname,
- kubectl get pod podname -o yaml.

*BEWARE:* the `get .. -o yaml` command produces a file which first part is some sort of schema, the second part is where real configuration resides. Don't mess with the initial schema part, no configuration will be updated and errors can occurs because of typo or misconfigurations (schema part is not trivial to read, at least for me).
Moreover, the `apply -f ..` command does not updates the new resources specified, I had to destroy and recreate the pod. Not sure that it is an issue or I (most probably) miss something.

### Daemon set
Looks similar to replicaset: they both allow to run multiple instances of the same pod. But DaemonSets ensure that at least a copy of the daemonset-ted pod runs on every node. DaemonSet suits perfectly monitor or log agents usecase. `kubeproxy` is run as a DaemonSet on every node.

DaemonSet specification file:
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
  namespace: some-namespace
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: monitoring-app
        image: my-monitoring-agent
```
it is exactly as the replicaset except for the name.

Commands used during test:
- kubectl get daemonset --all-namespaces -o wide,
- kubectl get ds (ds is the short for daemonset),
- kubectl describe daemonset kube-flannel-ds -n kube-system,
- kubectl create deployment elasticsearch --image=some-iimage:some-version --dry-run -o yaml > daemonset.yml (this creates a Deployment which is very similar to DaemonSet in its config file. Desired config can be achieved with minimal edit of the obtained file; basically edit the proper kind, remove the replicas and strategy entries and everything is not needed).

DaemonSet specifications override kube-scheduler decisions.

### Static pods
kubelet can manage its node even if there is no master. Without any api-server, kubelet can be configured to read pod specification files from a directory. Node polls that directory and keep its pods consistent with definition files. Pods created by kubelet this way are called static pods.

kubelet can create this way only pods.

Directory containing the pods configuration files is passed to kubelet as argument. See this `kubelet.service` file:
```
ExecStart=/usr/local/bin/kubelet \\
--container-runtime=remote \\
--container-runtime-endpoint)unix:///var/run/caninerd(containerd.sock \\
--pod-manifest-path=/etc/Kubernetes/manifests \\ (<-- this is the param)
--kubeconfig=/var/lib/kubelet/kubeconfig \\
--network-plugin=cni --register-node=true --v=2
```
or configuration can be specified separately:
```
ExecStart=/usr/local/bin/kubelet \\
--container-runtime=remote \\
--container-runtime-endpoint=unix:///var/run/caninerd(containerd.sock \\
--config=kubeconfig.yml \\ (<-- path will be specified in that file)
--kubeconfig=/var/lib/kubelet/kubeconfig \\
--network-plugin=cni --register-node=true --v=2
```
and in the `kubeconfig.yml` just set `staticPodPath: /etc/kubenetes/manifest`.

Because kubectl works with kube-api server, in a static pods scenario `docker ps` is the command to be used to get wich pods are running in the node.

When node is part of a cluster, kubelet register static pods to kube-api server by creating mirror components. Those mirror components are read-only, to modify static pods is mandatory to edit files stored in the manifest directory.

Why static pods? Because static pods are not dependant from controlplane, they can be used to deploy controlplane components itself. So no binary to install (apart kubelet) and node automatically takes care about crashing, just define the manifest directory and place a .yml file for each pod (controllermanager, apiserver, etcd...) to deploy.

Because pods are indicated explicitly in node's manifest directory, kube-scheduler takes no place in deployment of these pods.

Trick to identify a static pod: pod's name has '-node-name' as suffix.

Commands used during test:
- ps aux | grep kubelet (to find manifest folder passed as parameter),
- kubectl get pods --all-namespaces -o wide,
- kubectl describe node controlplane,
- ps -aux | grep kubelet,
- cat /etc/kubernetes/manifests/kube-apiserver.yaml,
- kubectl run static-busybox --image=busybox --dry-run -o yaml > static-busybox.yml | vi static-busybox.yml (and add command part),
- kubectl run static-busybox --image=busybox --command sleep 1000 --dry-run=client -o yaml > static-busybox.yml,
- mv static-busybox.yml /etc/kubernetes/manifests/,
- vi /etc/kubernetes/manifests/static-busybox.yml,
- kubectl get pod static-busybox-controlplane -o wide,
- kubectl describe pod static-busybox-controlplane,
- rm /etc/kubernetes/manifests/static-busybox.yml.

The following command were used to find and remove a static pod on node01.
- kubectl get pods --all-namespaces -o wide,
- ssh node01 (ssh admin@10.244.1.2 did not work.. there may be some config I ignore),
- ps -aux | grep kubelet
- cat /etc/kubernetes/kubelet.conf
- cat /var/lib/kubelet/config.yaml
- ls -al /etc/just-to-mess-with-you/
- rm /etc/just-to-mess-with-you/greenbox.yaml

### Multiple schedulers
k8s allows to write custom scheduler. A cluster can have multiple scheduler at the same time. To deploy a custom scheduler is mandatory to provide a .service file where to specify a unique `--scheduler-name=` value.

kubeadm deploys scheduler as pod.

scheduler uses a leader-election algorith when multiple masters are running in cluster.

Pod can specifically choose a scheduler by specifying `spec.schedulerName` value.

To see scheduler's log, type `kubectl logs my-custom.scheduler --name-space=kube-system`.

Commands used during test:
- kubectl get pods -n=kube-system,
- kubectl describe pod kube-scheduler-master -n=kube-system,
Command used in test to deploy a custom scheduler:
- cd /etc/kubernetes/manifests/ (here are config files about static pods by default),
- copy kube-scheduler.yaml /somewhereelse/my-scheduler.yaml,
- vi /somewhereelse/my-scheduler.yaml (add `--leader-elect=false` and `--scheduler-name=my-scheduler` in `spec.template.spec.containers.command`, also change `metadata.name` and `spec.template.spec.containers.name` accordingly),
- kubectl create -f /somewhereelse/my-scheduler.yaml,
Command used in test to deploy a pod by previously defined custom scheduler:
- touch nginx-pod.yml | vi nginx-pod.yml (define a pod with nginx as image, add `spec.schedulerName: my-scheduler`),
- kubectl create -f nginx-pod.yml,
- watch "kubectl get pods".

## Logging and monitoring

### Monitoring cluster components
A k8s cluster can monitor by using external components.

Metric server is an in-memory metric store solution.

kubelet ships a component called cAdvisor. cAdvisor pick values from pods and aggreagate them in metrics. Metrics are exposed throught kubelet api to metric server.

metric server can be added to a cluster as deploy or as an addons on minikube. Once installed, metric server will take some times to start collecting metrics, before he can be queried. To query metric server:
- `kubectl top node`,
- `kubectl top pod`.

Commands used during test:
- kubectl get pods --all-namespaces
- git clone https://github.com/kodekloudhub/kubernetes-metrics-server.git | kubectl create -f .
- kubectl top node
- watch "kubectl top node"
- kubectl top pod

### Managing application logs
Docker has a specific command to get log: `docker logs`. In k8s pod's log can be shown by `kubectl logs -f pod-name` but only if there is a lone container in the pod. If there are two or more containers in a pod, the container name must be specified, otherwise command will fail: `kubectl logs -f pod-name container-name`.

Commands used during test:
- kubectl get pods
- kubectl describe pod webapp-1
- kubectl logs webapp-1
- kubectl describe pod webapp-2
- kubectl logs webapp-2 simple-webapp
- kubectl logs webapp-2 -c (will show containers if more than one in the pod)

## Application lifecycle management

### Rolling updates and rollbacks
Creating a Deployment starts a rollout, when container in Deployment is updated a new rollout starts. Let's call first rollout revision01 and the second one revision02. By typing `kubectl rollout status deployment/my-deployment-name`, k8s shows how deployment of containers for each element of replicaset is going. Command `kubectl rollout history deployment/my-deployment-name` shows each revisions a Deployment was subjected to.

k8s allows some deployment strategies:
- Recreate: destroys all the pods/deployments before create the newer one, has downtime.
- Rolling Update: take down and immediately recreate each replica element one by one, has no downtime, it is the default deployment strategy.

Deployment strategies left trace in the deployment description (what you get when you `kubectl describe..`). `StrategyType` will be evaluated Recreate or RollingUpdate. Even the events will track that pods were destroyed then created in a Recreate case, or that pods containing the old image container were downscaled to zero while pods containing the new image were upscaled to replicas's value.

How to update a Deployment?
- by updating the container's image and run `kubectl apply -f deployment.yml`,
- by command `kubectl set image deployment/deployment-name container-name=image-name:version`

When a Deployment is created, a replicaset is created too with pods. When updating, the Deployment creates a new replicaset, fills it with pods shipping the updated image while removing the pods from the first replicaset. One by one, or few by few, user can tell how many at a time by configuration. Running `kubectl get replicasets` will show the oldest empty replicaset and the newer one filled up.

Command `kubectl rollout undo deployment/my-deployment-name` performs the opposite by bringing down the newer replicaset in favor of the older one, deploying again the pods with the old image.

Quick recap:
- `kubectl create -f deployment-definition-file.yml` creates a deployment,
- `kubectl get deployments` shows all deployments in current namespace,
- `kubectl apply -f updated-deployment-definition-file.yml` or `kubectl set image deployment/deployment-name container-name=image-name:version` both trigger an update rollout (recreate/rollingupdate) updating all pods in deployment's replicaset,
- `kubectl rollout status deployment/my-deployment-name` shows how rollout is going,
- `kubectl rollout history deployment/my-deployment-name` shows all the rollouts that occurred for that deployment,
- `kubectl rollout undo deployment/my-deployment-name` to rollback a deployment to previous revision,
- `kubectl rollout undo deployment/my-deployment-name --to-revision=3` to rollback a deployment to a specific revision number (history gives you revisions).

Commands used during test:
- kubectl get pods,
- kubectl get deployments -o wide,
- kubectl describe deployment frontend,
- kubectl edit deployment frontend (then change the spec.strategy attribute),
- kubectl set image deployment/frontend simple-webapp=kodekloud/webapp-color:v3

### Commands
In docker `CMD` tells what container do. Once `CMD` ends, container is terminated. docker run command accepts command as parameter: `docker run ubuntu sleep 5`.

Another option is to use `ENTRYPOINT`, parameters specified in the docker run command will be appended to the bottom of `ENTRYPOINT` value. `CMD` can be combined with `ENTRYPOINT` to provide default values or whatever to append when no argument is specified in docker run. `ENTRYPOINT` may be overridden by specifying `docker run --entrypoint some-command-to-run some-image argument1 argument2 ...`.

### Commands and arguments in k8s
The following docker command:
```
docker run --name ubuntu-sleeper ubuntu-sleeper
```
can be arranged on a pod this way:
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
```
This docker command:
```
docker run --name ubuntu-sleeper ubuntu-sleeper 10
```
can be arranged as a pod this way:
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]
```
To override image's `ENTRYPOINT` in k8s, the spec.containers[?].command overrides:
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-sleeper
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["dosomething"]
    args: ["10"]
```
spec.containers[?].command overrides `ENTRYPOINT` and spec.containers[?].args overrides `CMD` in docker image.

Commands used during test:
- kubectl get pods,
- kubectl describe pod ubuntu-sleeper,
- vi ubuntu-sleeper-2.yaml,
- kubectl create -f ubuntu-sleeper-2.yaml,
- vi ubuntu-sleeper-2.yaml,
- vi ubuntu-sleeper-3.yaml,
- kubectl create -f ubuntu-sleeper-3.yaml,
- vi ubuntu-sleeper-3.yaml,
- kubectl delete pod ubuntu-sleeper-3 | kubectl create -f ubuntu-sleeper-3.yaml,
- kubectl run webapp-green --image=kodekloud/webapp-color --restart=Never --dry-run=client -o yaml > my-pod-definition.yaml (then update the pod definition to add the args attribute).

### Configuring environment variable in k8s
Given the docker command
```
docker run -e APP_COLOR=pink simple-webapp-color
```
in k8s, to obtain the same:
```
apiVersion: v1
kind: Pod
metadata:
  name: fancy-app
spec:
  containers:
  - name: fancy-app
    image: simple-webapp-color
    env:
    - name: APP_COLOR
      value: pink
```
k8s provides other ways to set up environment variable: `ConfigMap` and `Secrets`.

### Configuring ConfigMaps in applications
Having multiple .yml defining components, it is difficult to manage environmental variable among all of them in a solid way. ConfigMaps resolve this problem.

ConfigMap is a dictionary:
```
APP_COLOR: blue
APP_MODE: prod
```
ConfigMap can be imperatively created in different ways:
- `kubectl create configmap my-configmap-name --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod`,
- `kubectl create configmap my-configmap-name --from-file=path-to-file`.

Or in the declarative way:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap-name
data:
  APP_COLOR: blue
  APP_MODE: prod
```
using command `kubectl create -f config-map.yml`.

`kubectl get configmaps` returns all the configmaps in the default namespace.

Configmaps can be injected in pod .yml this way:
```
apiVersion: v1
kind: Pod
metadata:
  name: fancy-app
spec:
  containers:
  - envFrom:
    - configMapRef:
        name: my-configmap-name
  - name: fancy-app
    image: simple-webapp-color
```

Commands used during test:
- kubectl get pods -o wide,
- kubectl describe pod webapp-color,
- kubectl get pod webapp-color -o yaml > webapp-color.yml | vi webapp-color.yml,
- kubectl delete pod webapp-color,
- kubectl create -f webapp-color.yml,
- kubectl get configmaps --all-namespaces,
- kubectl describe configmap db-config,
- kubectl create configmap webapp-config-map --from-literal=APP_COLOR=darkblue,
- kubectl create configmap webapp-config-map --from-literal=APP_COLOR=darkblue --dry-run=client -o yaml > webapp-config-map.yaml.

### Configure Secrets in applications
Similar to configmaps they are used to store sensitive infos. Secrets can be created in the imperative way by declaring the key/value pair,
```
kubectl create secret generic my-secret-name --from-literal=key=value
```
or by specifying a key/value map as file:
```
kubectl create secret generic my-secret-name --from-file=path-to-my-file
```
file `path-to-my-file` is a property file, keys and values are read and stored there.
Otherwise secrets can be created in the declarative way:
```
kubectl create -f my-secret.yml
```
and this is a secret definition:
```
apiVersion: v1
kind: Secret
metadata:
  name: my-secret-name
data:
  key1: value1
  key2: value2
```
Because .yml is a plain file, sensitive values must be stored in an encoded form. An encoded form could be the base64:
```
echo -n 'my-secret-passwd' | base64 (to get the value encoded)
echo -m 'gR0fk3jv' | base64 --decode (to get value back)
```
The `kubectl describe secrets my-secret` displays the secret opaqued, it is necessary to run `kubectl get secret my-secret -o yaml` to see secret's values (encoded or secured in some way).

Secrets are referenceable by pods:
```
apiVersion: v1
kind: Pod
metadata:
  name: fancy-app
spec:
  containers:
  - name: some-container
    image: some/image:v1
    envFrom:
    - secretRef:
        name: my-secret
```
This way keys in my-secret will become environment variables the pod can use.

As configmaps secrets are referenceable as files in a component by using volumes:
```
apiVersion: v1
kind: Pod
metadata:
  name: fancy-app
spec:
  containers:
    - name: fancy-app
      image: fancy-img
      volumesMounts:
      - name: secret-volume
        mountPath: where-my-secret-property-file-is
  volumes:
    - name: secret-volume
      secret:
        secretName: my-secret
```
by using volumes a file for each property in secret file will be created in the container.

Secrets are sent to a node only if a pod require them and kubelet stores them in a temporary filesystem. Once the pod referencing that secret is removed, the secret local copy is removed too.

Well secrets do not look so secure.. A better way could be to use ETCD as store and 'Encryption at Rest' to serve them. Other ways to handle secrets are available.

Commands used during test:
- kubectl explain pod (useful command to get syntax about component format, copy the required snippet and paste in your .yml),
- kubectl get secrets,
- kubectl get secrets -o wide,
- kubectl describe secret secret-name,
- kubectl get secret secret-name -o yaml,
- kubect create secret generic db-secret --from-literal=DB_Host=sql01 --from-literal=DB_User=root --from-literal=DB_Password=Password123 --dry-run=client -o yaml > db-secret.yml | kubectl create -f db-secret.yml,
- kubectl get pod webapp-pod -o yaml > webapp-pod-secret.yml (then edit the .yml to set up pod correctly),
- kubectl delete pod webapp-pod | kubectl create -f webapp-pod-secret.yml

### Multi Container PODs
Breaking the monolith in a bunch of microservices brings modularity, flexibility and efficiency by scaling the parts that really need instead of the whole thing.

Having two containers in a pod aim to have a sidecar service without merging functionalities in the same codebase. Multi container pods allow this:
```
apiVerion: v1
kind: Pod
metadata:
  name: fancy-webapp
spec:
  containers:
  - name: fancy-webapp
    image: fancy/webapp:v1
  - name: fancy-log-agent
    image: log-agent
```
A multi container pod really reminds me of docker-compose.

Additional containers in pod came because the main service needs some additional effort that is out of its main scope (i.e.: a log service). An additional container may occur as SIDECAR, ADAPTER or AMBASSADOR, these are distribute system design patterns.

Commands used during test:
- kubectl get po red -o wide,
- kubectl get pods,
- kubectl describe pod blue,
- touch yellow.yml | vi yellow.yml (and create a pod with two containers),
- kubectl create -f yellow.yml
- kubectl get pods -n elastic-stack
- kubectl -n elastic-stack describe pod app
- kubectl -n elastic-stack exec -it app -- /bin/bash
- kubectl -n elastic-stack exec -it app -- sh
- kubectl -n elastic-stack get pod app -o yaml > app.yaml
- vi app.yaml
- kubectl -n elastic-stack delete pod app | kubectl create -f app.yaml

### InitContainers
A container is supposed to stay alive as long as the hosting pod. If container stops or fails pod restarts. When instead, container's life is not bound to pod life, for example when container is supposed to run once and stops, then InitContainers comes in. InitContainers specify a set of processes that are executed at pod first creation. InitContainers can define a sequence of containers to run on pod creation. If a container in sequence fails, pod restart and InitContainers is executed again.

So, if some operation is needed before main container to start, InitContainer can trigger a container execution before the main container is executed.

Commands used during test:
- kubectl get pods -o wide,
- kubectl describe pod blue,
- kubectl get pods,
- kubectl describe pod purple,
- kubectl get pod red -o yaml > red.yml | vi red.yml,
- kubectl delete pod red,
- kubectl create -f red.yml,
- kubectl get pod orange -o yaml > orange.yml,
- kubectl delete pod orange,
- kubectl create -f orange.yml,

### Self healing applications
ReplicaSet and ReplicationController ensure application has the desired number of pods at all times.

### Introduction to autoscaling
Vertical scaling: add or remove resources to a host or instances.

Horizontal scaling: add or remove hosts or instances.

Horizontal scaling cluster means to add or remove nodes to cluster. Vertical scaling cluster means to add or remove resources to a node in the cluster. Commands `kubeadm join ..` and `kubectl drain ..` are useful tools here.

Horizontal scaling workload means to add or remove pods to cluster. Vertical scaling workload means to add or remove resources to a pod in the cluster. Commands `kubectl scale ..` and `kubectl edit ..` or `kubectl patch ..` are useful tools here.

### HPA
Hpa is the automated way to horizontal scaling workload in cluster.

It is mandatory to install and run a metric server to get metrics to scale on.

Hpa add or remove pods based on cpu, memory or custom metrics.

`kubectl autoscale deployment your-deploy --min=2 --max=10` scales pod imperatively.

A declarative example:
```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  creationTimestamp: null
  name: hpa-your-deploy
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: your-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resources:
    - name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

About metric server:
- you have to install it,
- provides metrics about cpu and memory usage out of the box,
- can fetch other internal metrics by custom metrics adapters,
- can fetch other external metrics by using external adapters.



## Cluster manteinance

### OS upgrade
When a node goes down pods fall with him, if pod is covered by a ReplicaSet then new pods will spawn in the cluster, but if a missing pod has no replica then that service is lost.

If node recovers then kubelet starts again and pods are online again. If node is down for more than 5 minutes pods are terminater and node is considered dead, ReplicaSet pods will respawn in available nodes. The 5 minutes wait can be configured by: `kube-controller-manager --pod-eviction-timeout=4m0s`, for example here time to declare node as dead is set to 4 minutes.

A node that came back after being declared dead is a fresh one with no pods inside. It is unschedulable.

Command `kubectl drain node01` causes the pods to be removed and node to be not schedulable, to mark the node schedulable again command `kubectl uncordon node01` is mandatory.

Command `kubectl cordon node01` marks the node as unschedulable, no new pod will be scheduled here.

Commands used during test:
- kubectl get nodes,
- kubectl get deployments,
- kubectl get pods -o wide,
- kubectl drain node01 (fails if there are pods running on the node),
- kubectl drain node01 --ignore-daemonsets (fails if there are pods not covered by a replicaset),
- kubectl drain node01 --ignore-daemonsets --force,

### K8s software versions
Running `kubectl get nodes` current k8s version is visible as most right column. k8s follows a standard release version path: major.minor.patch version number.

### CLuster upgrade process
Cluster main components in controlplane and worker node must have the same k8s version or be complaint with the following schema:

| version | apiserver | controllermanager | scheduler | kubelet | kube-proxy | kubectl |
|:-------:|:--------------:|:------------------:|:--------------:|:-------:|:----------:|:-------:|
| x | x-1 or x | x-1 or x | x-2 or x-1 or x | x-2 or x-1 or x | x-1 or x or x+1 |

kube-apiserver leads the k8s version. kubectl may be higher version than kube-apiserver. This may be useful on upgrading cluster one item at a time with a newer version.

k8s supports only the three higher software version. A newer version is released, then once was the third is no longer supported (WTF). Every three releases upgrading cluster is desiderable. kubeadm helps in upgrade the cluster.

Upgrading a cluster involves two major steps:
- upgrade first the master node,
- then upgrade workers.

While upgrading master node, controlplane components, apiserver, controllermanager, scheduler, go down, then the whole master node goes down. While master is applications are not impacted but no management is available, kubectl does not work.

When master came up again, is time for workers.

Workers can be upgraded together at a time: downtime will occur of course. Workers can be upgraded few at a time: no downtime, pods are evicted and recreated among available nodes. New workers can be added with the newest version: while new nodes are added old ones can be disposed until all the workers have new version.

By running command `kubeadm upgrade plan` a lot of info about cluster and k8s available versions is displayed.

To update a cluster:
- first update kubeadm, `apt-get upgrade -y kubeadm=1.20.0-00`,
- then run `kubeadm upgrade apply v1.20.0` to update controlplane component,
- then proceed about kubelet if master has one, `apt-get upgrade -y kubelet=1.20.0-00 && systemctl restart kubelet`,
- then, for each worker node:
  - `kubectl drain nodename` to remove pods and mark node as not schedulable,
  - `apt-get upgrade -y kubeadm=1.20.0-00` to upgrade worker node's kubeadm,
  - `apt-get upgrade -y kubelet=1.20.0-00` to upgrade the node's kubelet,
  - `kubeadm upgrade node config --kubelet-version v1.20.0`,
  - `systemctl restart kubelet`, and the node will be up again with the desired k8s version,
  - `kubectl uncordon nodename` to mark node as schedulable again

### Cluster upgrade demo
Following commands already are in the right queue to upgrade cluster.

Type `kubectl get nodes` to see if cluster is up or all its actors (controlplane, workers) are up and which k8s version is running.

Type `kubeadm token list` to see what tokes are available, you have to pick the right one for the cluster of choice (description is key to pick the desired one).

Type `kubectl get pods --all-namespaces` to see what to upgrade, type `cat /etc/*release*` to see current OS host is running.

As documentation says first check which version to upgrade to: `apt update && apt-cache madison kubeadm` to see kubeadm available versions. Upgrade kubeadm first: `apt-get update && apt-get install -y kubeadm=1.20.0-00`. `kubeadm version` should prompt the desired version.

Move to the controlplane (the next component you have to upgrade) node and run:
```
sudo kubeadm upgrade plan
```
to check if controlplane is upgradable to which version (check that upgradable and desired are matching). Command also tells which components to update by hand; `upgrade plan` shows upgrade recomendations. Now you are sure you can run:
```
sudo kubeadm upgrade apply v1.20.0-00
```
to safely upgrade the cluster. Now it is time to upgrade kubectl and kubelet. Before upgrade kubelet you have to drain pods from controlplane (the node in which you are from before and the one you want to upgrade first) and then upgrade:
```
kubectl drain controlplane --ignore-daemonsets
apt-get update && apt-get install -y kubelet=1.20.0-00 kubectl=1.20.0-00
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon controlplane
```
Now it is time to upgrade the worker nodes, following commands are intended for each node. Use some salt grain with them.

Move on the worker node, upgrade kubeadm:
```
apt-get update && apt-get install -y kubeadm=1.20.0-00
```
Then run
```
sudo kubeadm upgrade node-name
```
to check if node is upgradable to which version (check that upgradable and desired are matching). You now have to drain nodes and then upgrade:
```
kubectl drain node-name --ignore-daemonsets
```
Remember that `drain` is a command you must run from controlplane, you cannot perform that from another node. Once pods have been drained, move angain on the worker node and upgrade both kubelet and kubectl:
```
apt-get update && apt-get install -y kubelet=1.20.0-00 kubectl=1.20.0-00
sudo systemctl daemon-reload && sudo systemctl restart kubelet
kubectl uncordon node-name
```
as before, daemon-reload and kubelet are restarted, worker node uncordoned.

Commands used during test:
- kubectl get nodes,
- kubectl version --short,
- kubectl get deployments,
- kubectl describe deployment blue,
- kubectl describe deployment red,
- kubectl describe node node01
- kubectl describe node controlplane,
- kubectl get pods -o wide,
- kubeadm upgrade plan,
- kubectl drain controlplane --ignore-daemonsets
- kubectl drain --help
- (I was automatically ssh-ed into controlplane during test) apt-get update && apt-get install -y kubeadm=1.19.0-00,
- kubeadm upgrade apply v1.19.0 (upgrades kubectl),
- apt-get update && apt-get install -y kubelet=1.19.0-00,
- systemctl daemon-reload && systemctl restart kubelet,
- apt-get update && apt-get install -y kubectl=1.19.0-00 (did twice because I forgot I already upgraded it),
- systemctl daemon-reload && systemctl restart kubectl,
- kubectl uncordon controlplane,
- ssh node01,
- kubectl drain node01 --ignore-daemonsets,
- apt-get update && apt-get install -y kubeadm=1.19.0-00,
- apt-get update && apt-get install -y kubelet=1.19.0-00 kubectl=1.19.0-00 (didn't do kubeadm upgrade apply here, maybe on a worker node is not strictly necessary or maybe test missed this),
- sudo systemctl daemon-reload && sudo systemctl restart kubelet,
- (exit to return on controlplane) kubectl uncordon node-name.

### Backup and restore methods
What to backup in a k8s cluster? Well:
- configuration,
- ETCD cluster,
- persistent volumes.

Configurations are saved when used in declarative mode: configmap, namespaces, secrets for example. They could be easy saved on a git repo while created in declarative mode. When created in imperative mode, then commands like `kubectl get all --all-namespaces -o yaml > all-stuff.yml` are useful to get a copy directly from the kube-apiserver.

ETCD cluster stores status of k8s cluster. ETCD cluster is hosted on master/controlplane nodes. ETCD requires a path where to store its data, specified when started by `--data-dir` param, to backup that directory is a wise move. ETCD also provides command to fetch data to backup: `etcdctl snapshot save snapshot-file.db`. To see basic info about a snapshot: `etcdctl snapshot status snapshot-file.db`. To import a saved snapshot, command `service kube-apiserver stop && etcdctl snapshot restore snapshot-file.db` stops the apiserver, which stop the ETCD cluster and creates a brand new ETCD one from the snapshot. Then restart the services: `systemctl daemon-reload && service etcd restart && service kube-apiserver start`. Beware to provide again certificate and certificate CA to ETCD cluster.

What about persistent volumes?

### Working with etcdctl
etcdctl is etcd command line tool. To enable backup and restore features to etcd is mandatory to set the ETCDCTL_API to version 3 by exporting the proper environment variable, `export ETCDCTL_API=3`, on the master/controlplane node.

`etcdctl snapshot save -h` and etcdctl will shows all the options available for such command. Of course there is the same feature for the restore: `etcdctl snapshot restore -h`.

Because etcd database relies on TSL, following options are mandatory:
- --cacert,
- --cert.

etcd is exposed by default on port 2379.

Commands used during test:
- kubectl get deployments --all-namespaces,
- kubectl get pods --all-namespaces -o wide,
- kubectl describe -n kube-system pod etcd-controlplane,
- export ETCDCTL_API=3
- etcdctl snapshot save /opt/snapshot-pre-boot.db --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key
- kubectl get all --all-namespaces -o wide
- service kube-apiserver stop
- etcdctl snapshot restore /opt/snapshot-pre-boot.db --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key
- systemctl daemon-reload && service etcd restart && service kube-apiserver start
- kubectl get nodes

Can't complete last test's task. I tried by using solution but no luck. Here are commands I tried:
- ssh controlplane
- systemctl daemon-reload
- service etcd restart
- kubectl get deployments
- service kube-apiserver start
- ETCDCTL_API=3 etcdctl --data-dir /var/lib/etcd-from-backup snapshot restore /opt/snapshot-pre-boot.db
- vi /etc/kubernetes/manifests/etcd.yaml
- watch "docker ps | grep etcd"

And this is the lab's solution:

*start*
We have now restored the etcd snapshot to a new path on the controlplane - /var/lib/etcd-from-backup, so, the only change to be made in the YAML file, is to change the hostPath for the volume called etcd-data from old directory (/var/lib/etcd) to the new directory /var/lib/etcd-from-backup.
```
volumes:
 - hostPath:
   path: /var/lib/etcd-from-backup
     type: DirectoryOrCreate
   name: etcd-data
```
With this change, /var/lib/etcd on the container points to /var/lib/etcd-from-backup on the controlplane (which is what we want)

When this file is updated, the ETCD pod is automatically re-created as this is a static pod placed under the /etc/kubernetes/manifests directory.

Note: as the ETCD pod has changed it will automatically restart, and also kube-controller-manager and kube-scheduler. Wait 1-2 to mins for this pods to restart. You can run a watch "docker ps | grep etcd" command to see when the ETCD pod is restarted.

Note2: If the etcd pod is not getting Ready 1/1, then restart it by kubectl delete pod -n kube-system etcd-controlplane and wait 1 minute.

Note3: This is the simplest way to make sure that ETCD uses the restored data after the ETCD pod is recreated. You don't have to change anything else.
*end*

This is the course's solution
*start*
First, snapshot is taken by command:
```
etcdctl snapshot save --cert="/etc/kubernetes/pki/etcd/server.crt" --cacert="/etc/kubernetes/pki/etcd/ca.crt" --key="/etc/kubernetes/pki/etcd/server.key" /opt/snapshot-pre-boot.db
```
because etcd database relies on TSL, using the wrong (I choose peer.cert and peer.key) certificate and key will mess everything.

In simulation I was asked to check cluster status, it seems something went wrong:
```
kubectl get deployments,services
```
underlines cluster is in wrong status after maintenance.

To restore back the cluster:
```
etcdctl snapshot restore /opt/snapshot-pre-boot.db --data-dir=/var/lib/etcd-from-backup
```
No need to provide keys or certs, they are already setted up with the snapshot itself. Instead to run `vi /etc/kubernetes/manifests/etcd.yaml` is mandatory to restart etcd with the new config, which goes to `spec.containers.volumes[]hostPath.path`. Pick the `hostPath` with `name` matching the `volumeMounts[]mountPath: /var/lib/etcd` which is where etcd stores its data internally.

etcd pod will be recreated automatically.
*end*

## Security
Kubernetes actors rely on TLS certs to communicate each other.

Controlplane, or kubeapi-server, yaml can be found in /etc/kubernetes/manifest/kube-apiserver.yaml. Here are listed all the .crt and .key used to communicate with other k8s actors. You can also describe the controlplane pod on kube-system namespace to get the same infos. It is quite intuitive to get which files are used to communicate with which service.

Usually tls files are under `/etc/kubernetes/pki` folder.

Etcd service finds its tls into `/etc/kubernetes/pki/etcd` folder.

`openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout` command is your friend when you want to check what's inside a .crt.

apiserver and etcd use different CAs.

### Manage TLS certificates
User who wants to access cluster creates a .key and .crt file. .crt file must be signed by k8s CA.

Kubernetes has its CA server, with its .key and .crt; if you want to sign a .crt you should access that server.

controllermanager component is in charge about certificate operations. He has two specific subcomponents: csr-approving and csr-signing.

Kubernetes offers api to interact with its CA server.

Create a k8s object of type CertificateSigningRequest.

User creates .key and .csr: `openssl req -new -key user.key -subj "/CN=username" -out user.csr`.

User convert his .csr in base64: `cat user.csr | base64 | tr -d "\n"`.

A new CertificateSigningRequest object is defined:
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: csr_name_of_choiche
spec:
  expirationSeconds: 36000 #just an hour, not required
  usages: # look at k8s documentation pages
  - digital signature
  - key encipherment
  - client auth
  signerName: kubernetes.io/kube-apiserver-client
  request: <put base64 output here>
```
Cluster admins can see user signing request by command `kubectl get csr`.
By command `kubectl certificate approve csr_name_of_choiche`, admin asks k8s's CA to sign that csr. Now, by running `kubectl get csr csr_name_of_choiche` -o yaml, user can get his cert:
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  creationTimestamp: 2025-02-14T11:30:00Z
  name: csr_name_of_choiche
specs:
  groups:
  - system:<something>
  - system:authenticated
  expirationSeconds: 36000
usages:
  - digital signature
  - key encipherment
  - client auth
  username: <something>
status:
  certificate:
<signed certificate is here>
  conditions:
  - lastUpdateTime: 2025-02-14T11:32:00Z
  - message: This CSR was approved by kubectl certificate approve.
  - reason: KubectlApprove
  - type: Approved
```
Copy-n-paste the signed certificate and decode it as base64.

Other commands that complete csr management:
- `kubectl get csr <csr_name_of_choiche> -o yaml`
- `kubectl certificate deny <csr_name_of_choiche>`
- `kubectl kubectl delete csr <csr_name_of_choiche>`

### KubeConfig
Of course you can pass parameters to kubectl in order to interact with kube-apiserver:
```
kubectl get pods -n somenamespace --server someserver:6443 --client-key user.key --client-certificate user.crt --certificate-authority ca.crt
```
kubeconfig is far more handy. Kubeconfig has its configuration in `$HOME/.kube/config` and looks like this:
```
apiVersion: v1
kind: Config
current-context: k3d-test
preferences: {}
clusters:
- cluster:
    certificate-authority-data: <some-crt-data-base64-encoded>
    server: https://0.0.0.0:38645
  name: k3d-test
contexts:
- context:
    cluster: k3d-test
    user: admin@k3d-test
  name: k3d-test
users:
- name: admin@k3d-test
  user:
    client-certificate-data: <some-crt-data-base64-encoded>
    client-key-data: <some-key-data-base64-encoded>
```
There are three main subdocs:
- `clusters`: here go all cluster user may access,
- `users`: profiles user may use to access cluster listed in `clusters`,
- `contexts`: a list of blending about `users` and `clusters`.

`current-context`: indicates which combination of cluster and profile user is using.

- `kubectl config view`: shows kubeconfig file content,
- `kubectl config view --kubeconfig=some-kubeconfig-file`,
- `kubectl config use-context context-name`: change `current-context` pointing to given context name,
- `kubectl config -h`: prints help,

A context may specify a `namespace` attribute to indicate a default namespace in which operate.

### API Groups
kube-apiserver can be consulted by http api: `curl https://<host-where-kube-apiserver-is-hosted>:6443`. Paths matches with k8s api groups:
- /metrics,
- /healthz,
- /version,
- /api,
- /apis,
- /logs

Groups organize resources based on their functionality. Especially, /api and /apis find match into kubectl.

/api/v1 match with core group which is about essential standard types and functionalities: /api/v1/pods, /api/v1/namespaces, /api/v1/nodes, /api/v1/sevices and so on, this group is pretty flat. It match with `apiVersion: v1` in manifest.

/apis match with named group which is about /apis/apps, /apis/extendsions, /apis/networking.k8s.io, /apis/storage.k8s.io, /apis/authentication.k8s.io, /apis/certificates.k8s.io. All these groups have their resources.

Summarizing: top /api and /apis groups organize resources. /api refers to core resources organized in a flat way. /apis refers to named resources, types that are extensions or custom resources in the k8s landscape. Each resource has a set of verb associated, read a set of action that user can apply to them. Not all the resources have the same actions.

To interact kube-apiserver by http you have to specify key, certificate and CA's certificate. By using `kubectl proxy`, a proxy that fetch infos from kubeconfig is instantiated turning interaction by http more handy (like kubectl).

### Authorization
k8s allows following authorization modes: node, ABAC, RBAC, webhooks.

Node authorization: implemented by Node Authorizer.
Manage api operations for specific kubelets:
- read about service, endpoints, nodes, pods, secrets, configmaps, persistent volume claims, persistent volumes,
- write about nodes and node status, pods and pod status, events,
- auth-related ops like read/write about CertificateSigningRequests api, create TokenReviews and SubjectAccessReviews.
To enable NodeAuthorizer, kubeapi-server must be started with `--authorization-mode=..,Node` or specify a node authorizer in config file .
Any incoming request by user having system:node as CN cert name, are evaluated by the Node Authorizer which grants access.

ABAC is difficult to implement and manage.

RBAC relies on permissions, which grant action over resources, organized in roles. User are associated to roles to grand them rights about applying an action on a resource. This is the main
method authorization is built in kubernetes.

Webhook allows to externalize authorization, kube-apiserver routes requests to external service.

kube-apiserver has default mode as 'always allow'. kube-apiserver allows to use multiple authorization mechanism at once: when request arrives that is denied if and only if all mechanisms fail
to allow request. If one mechanism of them allows, then request is good to go.

### RBAC
This is the facto way to go into kubernetes authorization.

Create a `Role` object:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
...
```
Using property `resourceName` allows to get a finer grain in defining grants:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
  resourceNames: ["pod1", "pod2"]
...
```

Resources and verbs are kubernetes entities. Each `resource` type needs a specific rule.

Role object is bound to user by `RoleBinding`:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
`Role` and `RoleBinding` can point to any namespace.

As kubernetes objects, `kubectl get ..` and `kubectl describe ..` allow to check that objects.

Useful: `kubectl auth can-i <some-verb> <some-resource-kind>` tells if I can do that. As administrator, you can impersonate user and test her permissions:
```
kubectl auth can-i <some-verb> <some-resource-kind> -as <some-user> -n <some-namespace>
```
option `--as` works with every commands.

`kubectl config get-users` returns users.

This is wrong I don't get why:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: crud-pods
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: crud-pods-dev-user
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: crud-pods
  apiGroup: rbac.authorization.k8s.io
```
Solution uses imperative way:
```
kubectl create role developer --verb=list,create,delete --resource=pods
kubectl create rolebinding dev-user-binding --user=dev-user --role=developer
```
I mistake both role and rolebinding names... ok.

Useful command I forgot about: `kubectl edit role developer -n namespace`. Opens a 'vi', let's you update the resource and leave by ':wq'. Nice!

### Cluster Roles and Role Bindings
Roles and role bindings are namespaced. Nodes for examples aren't namespaced, they're clusterroled. By command `kubectl api-resources --namespaced=true` you can see namespaced resources, using `kubectl api-resources --namespaced=false` resources that aren't namespaced.

`ClusterRole` defines a set of grant about a not namespaced resource. That's pretty the same as `Role`.
`ClusterRoleBinding` binds a cluster role to a user.

You can create cluster role or bindings even in a namespace, they act globally anyway.

### Service Accounts
Are accounts used by non-humans that have to interact with kube-apiserver.

`kubectl create serviceaccount <service-name>`. Command create a service account and a token bound to it that should be used to implement confidential interaction as 'Authentication: Bearer <token>'. Token is store into a `secret` object and mounted as volume into bounded applications.

There is a `serviceaccount` named 'default' into every namespace.

By command `kubectl create token <service-account-name>` a token is created for that service account. Each pod using that serviceaccoutn will mount a volume containing the 'default' serviceaccount token. Volume is mounted into `/var/run/secrets/kubernetes.io/serviceaccount` as three separate files; 'token' contains the token and it is a jwt one and it is provided by 'token request api'.

To use a serviceaccount into a pod, you have to specify `serviceAccountName` as pod spec's property. You may choose not to automatically mount serviceaccounts in a pod by specifying `automountServiceAccountToken: false` in pod specs.

### Image Security
In `image: nginx`, `nginx` tells the image/repository.
In `something/nginx`, `something` indicates the user, `nginx` tells the image/repository.
In `somedomain/something/nginx`, `somedomain` points the registry from where to pull/push the image, `something` indicates the user, `nginx` tells the image/repository.

You may want restrict access to a specific registry by forcing login: `docker login somedomain` allows you to use credentials to access it. Docker will store a token in `~/.docker/config.json`.

You may store registry credentials in kubernetes as secret:
```
kubectl create secret docker-registry <registry-secret-name> --docker-server=<registry-url> --docker-username=<username> --docker-password=<password> --docker-email=<user-email>
```
The newly created `<registry-secret-name>` must be provided in pod/deployments into `spec/containers`:
```
...
metadata:
  name: my-pod
spec:
  containers:
  - ...
  - ...
  imagePullSecrets:
  - name: <registry-secret-name>
...
```

### Docker Security
Docker and Linux shares kernel. Processes have their namespaces and namespace is a boundary they cannot trepass, so containers are bound to a namespace and are isolated that way.

Docker run as root. Use `docker run --user=1000 ..` to mitigate or set that in the Dockerfile.

Container root user is not powerful as host root. Use `--cap-add` or `--cap-drop` to add or remove capabilities to docker user while running container.

### Security Contexts
As in docker, `--user=..`, `--cap-add=..` or `--cap-drop` can be used to change kubernetes behavior. These settings can be used at container or pod level:
```
apiVersion: v1
kind: Pod
metadata:
  name: someapp
spec:
  containers:
  - ...
  securityContext:
    runAsUser: 1000
```
Pod applies security context to each container but are overridden by security context configured in a specific container:
```
apiVersion: v1
kind: Pod
metadata:
  name: someapp
spec:
  containers:
  - ...
    securityContext:
      runAsUser: 1001
      capabilities:
        add: [..]
        drop: [..]
```

Tests: examples I wrote were wrong.. I don't get where to put `securityContext` attribute.. Provided solution does not use `kubectl edit` as I did in test but she gets the wide output, update it, delete the pod and apply the obtained yaml. Damn, quoting:
>it is intended that the spec of a Pod could not be updated after its creation except for the containers.image field.

That's why my changes didn't stay.

### Network Policies
Ingress traffic indicates infos incoming, Egress traffic indicate infos outgoing respect a certain resource.

Pods can comunicate with each other despite they are on different nodes because they share the same virtual network by default.

You can change default behavior by using network policies. NetworkPolicy is a kubernetes object you can link to a pod by using labels:
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
      - protocol: TCP # do we need anything else?
        port: 3306
```

Flannel does not support network policies: you can create them but they'll take no effect. Calico or Weave-net instead support.

### Developing network policies
By placing:
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
        namespaceSelector:
          matchLabels:
            name: somenamespace
      ports:
      - protocol: TCP # do we need anything else?
        port: 3306
```
we are enforcing a network rule to all pods that match `somekey: somevalue` among their labels. Rule ensure that only TCP incoming TCP traffic on port 3306 coming from a pod that match `someotherkey: someothervalue` as label is allowed. Replies to that incoming traffic does not need any additional rule to flow to requestor. Btw affected pods cannot query other services because network policy allows them to only do one thing, in this case reply to traffic that came form `someotherkey: someothervalue` pod on TCP on port 3306.

Network policies are not namespaced, so the `namespaceSelector` come to an hand by allowing incoming traffic only for specific labelled namespaces.

To enable external resources to do ingress traffic to a pod, we can use `ipBlock`:
```
...
    - podSelector
      ...
    - ipBlock:
      cidr: 192.168.5.4/32
...
```
the `ingress:from` expects an array, Each element can describe a rule or a set of rules. From the perspective of incoming (or outgoing in case of egress rule) traffic, just an element of `ingress:from` must be satisfied to be allowed. If the element is a set of rules, traffic must satisfy all rules in that set to be allowed. It is a sort of logical 'disjunction of conjunctions'.

This is a rule made of two sub-rules, traffic must satisfy both:
```
...
- from:
  - podSelector:
      matchLabels:
        someotherkey: someothervalue
    namespaceSelector:
      matchLabels:
        name: somenamespace
...
```

These are two rules, traffic must satisfy one:
```
...
- from:
  - podSelector:
      matchLabels:
        someotherkey: someothervalue
  - namespaceSelector:
      matchLabels:
        name: somenamespace
...
```
Small changes make big difference here.

As for `ingress`, we can specify `egress` rules too:
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
  - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            someotherkey: someothervalue
        namespaceSelector:
          matchLabels:
            name: somenamespace
      ports:
      - protocol: TCP # do we need anything else?
        port: 3306
  egress:
    - to:
      - podSelector:
        ...
        namespaceSelector:
        ...
        ipBlock:
        ...
      ports:
```
egress rules allows/denies outgoing traffic. Example:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
    - Egress
  egress:
    - to:
      - podSelector:
          matchLabels:
            name: payroll
      ports:
      - protocol: TCP
        port: 8080
    - to:
      - podSelector:
          matchLabels:
            name: mysql
      ports:
      - protocol: TCP
        port: 3306
```

### Custom Resource Definition (CRD)
By command or by applying a yml, you change etcd status. A controller checks etcd status and applies changes to cluster in order to keep it solid with status.

Resources can be listed by command `kubectl api-resources`. So, you put down a spec for a resource type, you apply it to etcd, a controller will detect the change and apply that change to cluster too, keeping it solid with etcd. There existis a controller for each type of resource.

Let's introduce a new resource. You want to manage that resource into etcd and you want cluster to implement that, to make it real. So you have a Custom Resource which defines a new type in etcd and a Custom Controller which creates an entity of that custom type into cluster.

By using `Custom Resource Definition` we can add types in kubernetes
```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: my-custom-resource #custom resource name
spec:
  scope: Namespaced #tells resource is namespace scoped and not cluster-wide
  group: some.group
  names:
    kind: MyCustomKind #what to put into kind to tell that resource is our brand new
    singular: mycustomkind #label about how to interact by kubectl
    plural: mycustomkinds #label to interact by kubectl
    shortnames: #aliases
      - mck
      - <someothershortname>
  versions: #manage multiple version of CRD
    - name: v1
      served: true
      storage: true
  schema: #define our brand new resource spec definition
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            ...
```
Once applied, the CRD allows you to define new entities of the just introduced new resource type and store them to etcd.

### Custom Controllers
Once you have your CRD you have to develop a custom controller to get that in your cluster. Google provides a git repo from which you can fetch a sample controller, some sort of template: `git clone https://github.com/kubernetes/sample-controller.git`.

Custom controllers are usually distributed as docker images and run as pod into kubernetes.

### Operator Framework
CRD and Custom Controllers are different beasts: operators 'merge' them together for smooth installation and manage (backup, restore).

## Storage

### Docker Storage
Storage driver and volume drivers.

### Storage in Docker
When you install docker, it creates a folder tree:
```
/var/lib/docker
  ./aufs
  ./containers
  ./image
  ./volumes
```
Docker stores files in layers: each line in dockerfile is a layer in docker image, it is incremental; docker uses that approach to save time and space.

Once `docker build` is called the layers build an image and turn immutable. When `docker run` is run, a new layer of mutable data is created, made by all the files produced by the
containerized service (e.g.: log files). This mutable layer is destroyed alongside the container. You cannot change the immutable files, by modifying them, a new copy is created in the
additional layer created by the container (that's the COPY-ON-WRITE mechanism).

When using volumes, docker creates a volume object in its `/var/lib/docker/volumes` directory: `docker run -v somefolder:/tmp some image`.The shared directory is mount in the container in
its additional fs layer, this is called 'Volume Mount'. You can specify existing path to mount as volume, this is called `Volume Bind`.

Docker uses 'Storage drivers' to handle fs layers, docker uses which storage driver is available in operating system.

### Volume Driver Plugins
Docker uses 'Volume drivers' to handle volumes. Default volume driver is 'Local', there are many more. You may choose a volume driver by:
```
docker run -it --volume-driver somevolumedriver -v /usr/local/myfiles:/tmp myimage
```

### Container Storage Interface
Kubernetes doesn't rely on docker to run contaienrs, he can use other contanerization tools like 'rkt' or 'cri-o', basically every container runtime that is compliant to 'CRI': Container
Runtime Interface. For volumes, the Container Storage Interface is mandatory to be implemented to allow mounting and managing fs resources.

### Volumes
Pods are transient by nature. Providing a volume to a pod allows us to keep data pod may produce. Given:
```
apiVersion: v1
kind: Pod
metadata:
  name: dummypod
spec:
  container:
  - image: alpine
    name: dummy-alpine
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/result.data;"]
    volumeMounts:
    - mounthPath: /tmp # where you can find files in container
      name: some-volume-name
  volumes:
  - name: some-volume-name
    hostPath:
      path: /data # this is the host directory
      type: Directory
```
Here pod and volume are defined in the same resource. It is better to separate them in different yml.

### Persistent Volumes
Are kubernetes objects to share node fs with pod container.
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  accessModes:
    - ReadWriteOnce # allowed values: [ReadWriteOnce, ReadOnlyMany, ReadWriteMany]
  capacity:
    storage: 1Gi # disk space associate to volume
  hostPath: # could be any other volume driver supported as CSI
    path: /tmp # host folder to be shared as volume
```

### Persistent Volume Claims
k8s objects that implement need for a persistent volume. Once you define a pvc, kube-apiserver looks for a pv to bind with. Pvc and pv are bound 1 to 1.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```
If there is no perfect fit, pvc will be bound to pv who can satisfy required capacity; `my-pv` has storage bigger that required by `my-claim` so could be a good fit if there is no better
match.

When you delete a pvc, `kubectl delete persistentvolumeclaim my-claim`, if persistence volume has:
- `persistentVolumeReclaimPolicy: Retain`, pv is kept but unavailable to other pvcs. This is the default behavior,
- `persistentVolumeReclaimPolicy: Delete`, pv is deleted as well,
- `persistentVolumeReclaimPolicy: Recycle`, pv is kept and available to other pvcs.

I always forget that only containers part can be updated.

Even `kubectl apply -f ...` doesn't work on existing pod if changes affect parts other than containers. You can use kubectl `replace --force -f your-file.yml`.

Can't delete pvc if it is used by pod. Delete the pod and you'll also delete the pvc..

### Using PVCs in Pods
Once you create a pvc, use it in a pod definition file by specifying it's name under `persistentVolumeClaim` section in the `volumes` section like this:
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```
The same is true for `ReplicaSets` or `Deployments`. Add this to the pod template section of a `Deployment` on `ReplicaSet`.

### Storage Class
Operates dynamic provisioning.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
```
Provides automatically the pv to get the volume. So the pvc that want to use that storage should be like this:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: google-storage
```
Usually used for external provisioners. Different provisioners may require or allow different parameters to be specified.

Better for checkout which provisioners are available, even if exists for local fs.

Question about storage class that doesn't support dynamic volume is broken.. How do I know that? From solution: sc PROVISIONER get info about type of volume provisioning, provisioner
kubernetes.io/no-provisioner does not allow dynamic volumes.

Sc I defined:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: local-storage
```

## Networking

### Prerequisite: switching routing
Switch is a device capable of forwarding infos to proper host among all that are connected to. He knows that ip address is bound to some ethernet port, so when packet arrives, he forward it
throught the right port. Switch deliver ip packets in a network: when an ip packet arrives, he knows to which port forward it.

Router is a device capable of connecting networks together.

Gateway is the way to reach what is outside the network. Usually a gateway play the role of 'default gateway': when you don't know who should receive a packet, then move it to the default
gateway.

Ip addresses are bound to network interfaces.

Linux prevent packets to be moved from an ethernet to another one on the same device. Run: `echo 1 > /proc/sys/net/ipv4/ip_forward` and edit `/etc/sysctl.conf` accordingly to enable packets
forwarding from ehternet to another. Run `echo 0 > /proc/sys/net/ipv4/ip_forward` and put `0` again in `/etc/sysctl.conf` to disable.

`ip link`: list and modify network interfaces on host.
`ip addr`: shows ip addresses assign to network interfaces.
`ip addr add 192.168.1.10/24 dev eth0`: set ip addresses to interface.
`ip route`: shows the routing table.
`ip route add 192.168.1.0/24 via 192.168.2.1`: add entry into routing table.
`cat /proc/sys/net/ipv4/ip_forward`: to check if packet forwarding is allowed among network interfaces on that host, 1 means yes, 0 means no.

### Prerequisite: DNS
Dns is a service capable of name resolution: he knows that a given name correspond to a specific ip address. When host doesn't recognize a name, he'll ask to dns service.

On linux set `nameserver   some-ip-address` in `/etc/resolv.conf` to tell host who can help on resolving names. Host first looks in it's /etc/hosts file, then asks to dns. We can change this
behavior by modifying `/etc/nsswitch.conf`.

`www.something.org` is a domain name. `org` is the top level domain name, `something` is the name and `www` is the subdomain.

So, resolving `www.something.org`, you will first look at host's `/etc/hosts` or `/etc/resolv.conf`, then you'll ask to some dns about top level domain, `.org`, then hierarchically to another
dns about `something` and then `www`.

`nslookup` is useful to query dns services about names, `dig` is similar.

### Prerequisite: CoreDNS
DNS services manage name resolution to ip, they are mandatory in large environments.

This part is barely touched because not in CKA scope. Saying that, is better to improve this part with some basic knowledge of CoreDNS.

### Prerequisite: Network Namespaces
Linux uses namespaces to keep resources separate or bound to processes. A process is aware about resources in its namespace, knowing nothing about others.

Once a container is created it is sealed in a namespace. He also has a network namespace: a virtual network interface with routing and arp tables.

`ip netns` list existing network namespaces.
`ip netns add some-name` creates a network namespace named `some-name`.
`ip netns exec some-name ip link` or `ip -n some-name ip link` executes command `ip link` in network namespace `some-name`.
`ip link add someveth-name type veth host-name someotherveth-name` creates a virtual ethernet connection between virtual network interfaces `some-name` and `someother-name`
(unsure about that command).
`ip link set someveth-name netns some-name` adds virutal network interface to specified network namespace.
`ip -n some-name addr add 192.168.10.1 dev someveth-name` applies ip address to virtual network interface `someveth-name`.

`ip link add v-net0 type bridge` adds new virtual network interface of type `bridge`. `ip link set dev v-net0 up` turns the network namespace up. Type `bridge` can be used by other virtual
network interfaces in network namespace to link together and with the host: `ip link set someveth-name master v-net00` and `ip link set someotherveth-name master v-net01` then set virtual
network interfaces to network namespaces, add ip addresses and turn everything up.

Network interfaces in namespaces are aware of each other and aware of virtual bridge network on host. To reach something outside host you have to:
`ip nets exec some-name ip route add 192.168.1.0/24 via <ip-address-of-bridge-virtual-network-interface>`
Now to reach something outside host and get reply, you have to NAT and tell host to act as a gateway:
`iptables -t nat -A POSTROUTING -s <ip-subnet-of-bridge-virtual-network> -j MASQUERADE`
and to act as a default gateway to allow network namespaced network interfaces to reach internet, so:
`ip netns exec some-name ip route add default via <ip-address-of-bridge-virtual-network-interface>`.

To let entities outside our host to reach elements inside a network namespace we can rely again on iptables:
`iptables -t nat -A PREROUTING --dport 80 --to-destination <ip-address-of-a-virtual-network-interface>:80 -J DNAT`
this way we are telling host to route all traffic about port 80 to specific virtual network interface.

You can easily see how cumbersome that is.

### Prerequisite: docker networking
When running containers we can choose different options about networking by `--network` option.

Option `host` allows host and containers see each other because they share the host's network interface.

Option `bridge` creates a bridge network namespace with containers within, so a virtual private network is built. Docker creates a `bridge` network named docker0 when installed, exactly as we run `ip link add docker0 type bridge`. When a container is started docker creates virtual network intefaces and virtual connection to link container's network namespace with `bridge` network.

Docker also create iptables rules to NAT containers and allows port forwarding.

### Prerequisite: CNI
CNI is a standard interface to implement networking. This allows different providers offer their network solutions as plugins that conform with CNI: calico, weaveworks, flannel, vmware nsx etc...

Docker doesn't implement CNI, he has its own standard knows as Container Network Model. You can use specific CNI by running container with `--network none` and add networking stuff yourself with your preferred CNI tool.

### Cluster networking
Kubernetes cluster is made of master and worker nodes. Each node has its own network interface, host name and mac-address.

Master nodes should accept connections:
- on port 6443 because of kube-apiserver,
- on port 10250 because of kubelet,
- on port 10259 because of kube-scheduler,
- on port 10257 because of kube-controller-manager,
- on port 2379 because of etcd.

Worker nodes should accept connections:
- on port 10250 because of kubelet (also in worker nodes),
- on port 30000-32767 because of hosted services.

ip netns
ip -n cni-346e56ca-012c-2744-b8f2-586d63554345 link show
ip addr (virtual networks are listed on host)
ip addr also show network namespace

cannot find mac adress assigned to node..
tried with:
- ip addr (and got the network namespace)
- ip netns exec cni-c53e54ff-ca48-4c9d-7773-badf04610a43 ip link
solution was to ssh into node01, `ssh node01`, and then do `ip address` to have all the infos about node's network interface, including its mac-address.

`ip address show type bridge` lists all bridge type networks on host.

`netstat -npl | grep -i program-name` to check which ports `program-name` uses.

cannot get how many connections on port, maybe is a tricky question.
Solution states `netstat -npa | grep -i program-name` shows all connections estabilished on ports and kept by `program-name`. So that wasn't a tricky question.

### Pod networking
Kubernetes does not provide a networking solution at pod level, cni you'll install does.

Kubernetes states:
- every pod should have an ip address,
- every pod should be able to comunicate with pods in the same node,
- every pod should be able to comunicate with pods hosted on other nodes without NAT.

So, each node creates a bridge network and, for each pod, it creates a network namespace, two virtual network interfaces, one bound to pod network and the other bound to the bridge network. Each bridge network has its own subnet and then, accordingly to that subnet, pods will have ip addresses. In the end network interfaces are turned up. To let pods communicate across nodes, each bridge network uses node's network interface as default gateway.

That's what CNI, along with other tasks, does in a way or another.

### CNI in kubernetes
CNI:
- creates container network namespaces,
- find network containers must attach to,
- binary config in JSON format.

Kubernetes cni binaries are hosted in `/opt/cni/bin` and configuration are in `/etc/cni/net.d/`.

### CNI weaveworks (weaveworks shuts down)
Weaveworks's CNI uses agents: agents are deployed and live in nodes sharing infos each other about which pods are hosted where they are. When a packet is sent, weaveworks agent in the node wrap it up, send to target agent accordingly to network topology he store. Target agent unwrap and move the packet to proper pod.

Weaveworks shuts down, it would be good to get some infos about Calico.

Kubectl describe node controlplane gives infos, even which service container runtime kubernetes is using. You can also use `ps -aux | grep -i kubelet | grep container-runtime`.

CNI binaries are in `/opt/cni/bin/`. Here you can also find installed plugins, they're the binaries in that directory.

### Ip address managment in weaves (weaveworks shuts down)
CNI plugin assings ip to containers. Usually CNI solution uses DHCP, Host-Local or other custom solution.

Finding one weave pod and describing it will gives a bunch of infos about weave.

### Service networking
Services are the backbone of high level communication among pods. Pods won't use ip:port to call each other because they're ephemeral, they rely on services who are in charge to act as load balancer and do service discovery.

Each service receives an ip address and is set to expose a specific port. A service is accessible to all pods in cluster even if they're in different nodes, they are not bound to node, Service is an entity that traverse cluster. They don't get any namespace or network interfaces they're totally virtual.

When service is created, kube-proxy pick an ip adress from a specific set of addresses and creates forwarding rules on each node in the cluster

ClusterIP: only accessible inside cluster
NodePort: creates a port forwarding rule that allows pod to be accessible. You usually use this to access application from outside cluster.

kubelet gets changes from kube-apiserver, kube-proxy uses cni to gives ip address to service (by default, kube-proxy uses iptables's forward rule) by picking ip address from special set which is configured in kube-apiserver, `kube-apiserver --service-cluster-ip-range <default 10.0.0.0/24>`.

A good source of infos is the kube-apiserver configuration: `/etc/kubernetes/manifests/kube-apiserver.yaml`.

I don't get how to get ip address range used by nodes.
Pick a node's ip address and check by `ip addr` to see its subnet.

I don't get how to get ip address range used by pods.
Check the cni pod and look at its logs for command `ipalloc` and get which range is using.

I don't know how to get kube-proxy type.
Get logs of a kube-proxy pod to see which type of kube-proxy is.

I don't know how kube-apiserver ensures there is a kube-proxy on each node in the cluster.
Looking at resources in cluster you can see no deployments or replicaset about kube-proxy pods but there is a daemonset about it, so daemonset is how it does.

### Cluster DNS
Kubernetes relies on DNS to solve pod names and he install one.

When a service is created, an entry in DNS is made storing it name and ip address. If service `balancer` is deployed in a specific namespace, say `app`, then the dns will store an entry like:

| hostname | namespace | type |     Root      | IP address |
|:--------:|:---------:|:----:|:-------------:|:----------:|
| balancer | app       | svc  | cluster.local | <ip add>   |

So the service will be resolved with name `balancer.app.svc` or `balancer.app.svc.cluster.local`.

For pods:

| hostname | namespace | type |     Root      | IP address |
|:--------:|:---------:|:----:|:-------------:|:----------:|
| my-pod   | app       | pod  | cluster.local | <ip add>   |

where `my-pod` is the value of pod's `spec.hostname` attribute. If no `spec.hostname` value is provided, then hostname in dns's table will be the pod's ip address with dashes instead of dots.

### Core DNS in kubernetes
Fore each pod that is created, an entry in its `/etc/resolv.conf` file is added with hostname and ip address of cluster's dns server. This way each pod can resolve pods ip addresses and reach each other.

CoreDNS is deployed as two pods in kube-system namespace, its configuration stays in `/etc/coredns/Corefile`. In that file there are few plugins specified, the most important is the `kubernetes` plugin. Configuration file is passed as configmap to CoreDNS pods.

CoreDNS pods rely on service to be reached, by default service is a `ClusterIP` with name `kube-dns`. This service is configured in each pod's `/etc/resolv.conf` file when pod is created.

Services can be reached by their name while pods needs the fully qualified name: `<pod-host-name-or-dashed-ip-addr>.<namespace>.pod.cluster.local`.

What is a root domain/zone for a cluster?
Get a look to CoreDNS configmap, root domain is just after `kubernetes` plugin entry.

To change pod which has been deployed by deploy, edit the deploy not the pod.

`kubectl -n <namespace> exec <pod> -- <command>`.

### Ingress
Allows user to reach application in cluster, a boundary component.

Ingress = Ingress controller + Ingress resources.

Ingress controller has a load balancer, kubernetes doesn't have a default one, you have to install by yourself. As load balancer, ingress controller relies on some load balancer like nginx,
traefik, haproxy, etc. he use that load balancer/reverse proxy as inner engine, ingress controller itself is basically an interface.

So, in order to get Ingress on your cluster you have to:
- deploy a `Deployment` that uses nginx or traefik or kong or whatever, specifying container's `args` and `env` required,
- deploy a `ConfigMap` that wrap configuration,
- deploy a `Service` to expose Ingress outside the cluster,
- deploy a `ServiceAccount`, `Role`, `ClusterRole`, `Rolebindings` objects to let ingress manage configuration changes (and much more).
```
# getting an ingress controller is far more easier by using the helm resource
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.1
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          volumeMounts:
          - name: custom-snippets
            mountPath: /etc/nginx/custom-snippets/
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
      volumes:
      - name: custom-snippets
        configMap:
          name: custom-snippets
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  client-body-buffer-size: 60k
  client-header-buffer-size: 16k
  error-log-level: warn
  http2-max-field-size: 16k
  large-client-header-buffers: 4 16k
  log-format-escape-json: "true"
  log-format-upstream: '{"time":"$time_iso8601","remote_addr":"$remote_addr","proxy_protocol_addr":"$proxy_protocol_addr","proxy_protocol_port":"$proxy_protocol_port","x_forward_for":"$proxy_add_x_forwarded_for","remote_user":"$remote_user","host":"$host","request_method":"$request_method","request_uri":"$request_uri","server_protocol":"$server_protocol","status":$status,"request_time":$request_time,"request_length":$request_length,"bytes_sent":$bytes_sent,"upstream_name":"$proxy_upstream_name","upstream_addr":"$upstream_addr","upstream_uri":"$uri","upstream_response_length":$upstream_response_length,"upstream_response_time":$upstream_response_time,"upstream_status":$upstream_status,"http_referrer":"$http_referer","http_user_agent":"$http_user_agent","http_cookie":"$http_cookie"}'
  location-snippet: "include /etc/nginx/custom-snippets/location-custom.conf;"
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

Ingress resources are:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: some-name
spec:
  backend:
    serviceName: that-service-name
    servicePort: that-service-port
```
a sort of forwarding rule. Can route by path or domain. Here is a by path example:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: some-name
spec:
  rules:
  - http:
      paths:
      - path: /resource1
        backend:
          serviceName: that-service-name
          servicePort: that-service-port
      - path: /resource2
        ...
```
and this is by domain:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: some-name
spec:
  rules:
  - host: some1.domain.com
    http:
      paths:
      - backend:
          serviceName: that-service-name
          servicePort: that-service-port
  - host: some2.domain.com
    http:
      paths:
      - backend:
          serviceName: that-service-name
          servicePort: that-service-port
  ...
```
### Annotations and rewrite-target
Ingress controller relies on external software, for example nginx. Nginx offers feature that someother loadbalancer/reverse-proxy does not, one of these features is url rewrite. Here:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myapi-ing
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite */tmp^ / break; # unsure this works, probably not, but this is not the point here
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /auth/api
        backend:
          serviceName: myapi
          servicePort: myapi-port
```

This is wrong:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
  namespace: ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /stream
        backend:
          service:
            name: video-service
            port:
              number: 8080
        pathType: Prefix # what is pathType?
```
don't know why.

I totally miss that. All that was required was just edit the existing Ingress to change path in one rule. I added a new rule out-of-the-box causing a mess.

This is wrong too!
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-pay
  namespace: ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /pay
        backend:
          service:
            name: pay-service
            port:
              number: 8282
        pathType: Prefix # what is pathType?
```
This was wrong because of its namespace!!! Why?

```
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress-nginx-admission
  namespace: ingress-nginx
```

Unable to fix all the issues in the big .yml containing nginx-ingress-controller deployment, service, configmap and namespace.

### Introduction to Gateway API
The new generation of ingress controller. Introduces new concepts and objects: GatewayClass, Gateway, HTTPRoute, TCPRoute, UDPRout etc.

Like ingress, gateway needs a controller too.

GatewayClass tells how a Gateway is implemented.
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "someurl.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
    backendRefs:
    - name: example-svc
      port: 8080
```
Gateway API leverage differences about controllers in its routes:
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
  name: split-traffic
spec:
  parentRefs:
  - name: app-gatewayClassNamerules:
  - backendRefs:
    - name: app-v1
      port: 80
      weight: 80
    - name: app-v2
      port: 80
      weight: 20
```
It allows to switch between controllers without changing configuration.

How to link services to gateway routes?
By using `backendRefs` attribute in rule.
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
spec:
  parentRefs:
    - name: nginx-gateway
  rules:
  - matches:
    - path:
        value: "/path"
    backendRefs:
    - name: frontend-svc
      port: 80
```

Mandatory to improve this lecture, Gateway API is the future.

Looks like ingress controller will live alongside Gateway API objects.

Gateway's `allowedRules` indicates which namespaces can attach rules.

## Design and Install a kubernetes cluster

### Design a kubernetes cluster
Depends on your needs. Nothing impacting on CKA.

### Choosing kubernetes infrastructure
Kubeadm expects have ready machines.

### Configure high availability
Multiple master nodes is mandatory in any production environment to avoid SPOF.

In a multi master node scenario, kube-apiserver instances should stay behind a load balancer.

ControllerManager, scheduler, kube-apiserver they don't work in parallel but in an active-standby mode by electing who is the active and keeping the others passive.

ETCD can be in the master node or can stay outside the cluster. Using ETCD as external service spare him from failures in cluster (but you may experience outage or whatever).

### ETCD in ha
Etcd keeps the same data among all instances in its cluster. Instances elect a leader who is in charge to satisfy write requests, other instances are followers. When a write request land on
a follower, it forward that request to the leader, then the leader spread the data change among all the followers assuring that the majority of the followers get that message.

Etcd uses RAFT algorithm to elect leader. When leader gone, other followers re elect another leader.

Etcd uses a N/2+1 quorum to says cluster works properly. Choose proper number of master nodes to get best from etcd cluster, if it loose quorum you lose the cluster.

## Install kubernetes the kubeadm way
Bootstraps a kubernetes cluster.

Steps:
- provide machines, they'll become nodes,
- choose who are masters and who are workers,
- install containerd on all machines,
- install kubeadm on each machine (official doc says kubelet, kubeadm and kubectl),
- initialize master nodes on machines of choice,
- setup POD network by joining worker nodes.

### Deploy with kubeadm
Having machines that are solid with minimum system requirements, install on each: containerd, kubeadm, kubelet.

k8s 1.21 and later uses systemd as default. Container driver should be configured to use systemd or the installed alternative.

It's a sort of 'read the docs and apply these commands'.

Initialize master nodes: use `--control-plane-endpoint` to tell loadbalancer about your master node. This is the command:
```
kubeadm init --apiserver-advertise-address <node-ip-address> --pod-network-cidr "<a-subnet>" --upload-certs
```
Command does a lot of stuff. At the end, you have to pick configuration for your kubelet (copy a file to put in $HOME/.kube/config) and the command to run on each worker in order to join the cluster:
```
kubeadm join <master-node-ip-address> --token <some-token> --discovery-token-ca-cert-hash <some-hash>
```
If you have multiple master nodes cluster command differs a bit.

Choose a pod network: installing cni is the last step, it may be calico, flannel, whatever of your choiche.

Now join workers into cluster.

Issues while installing kubeadm and kubelet
```
sudo apt-get install -y kubelet=1.32.0-1.1 kubeadm=1.32.0-1.1
```
apt-get doesn't resolve.. I guess I miss some initial passage.

I don't get how to create a cluster with multiple master nodes..

## Helm

### Why helm
Simplest app can turn into 5+ different objects in a kubernetes cluster, each one craftly configured. Managing that can be painful.

Helm helps manage that objects as a whole app instead as a collection of yaml:
- `helm install someapp`,
- `helm upgrade someapp`,
- `helm rollback someapp`,
- `helm uninstall someapp`.

Helm is a packet manager, sort of.

### Installing helm
You need a cluster, then type:
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Helm 2 vs helm 3
Pay attention because there are breaking changes. Helm 3 came out in 2019, as kubernetes provides RBAC and custom resource definition types.

Helm provides a snapshot system by keeping tracks of changes throught revisions. Installing a chart:
```
helm install some-software
```
initiates a revision number bound to that installation. By doing changes to that installation, helm increase its revision number. You can use helm to rollback a previous revision, this will also increase the revision numer. So, rollback installation to previous states generates a newer revision number bound to a new snapshot. See table:
| helm action | software version | revision number |
|:-----------:|:----------------:|:---------------:|
| install | 1.0.0 | 1 |
| upgrade | 2.0.0 | 2 |
| rollback | 1.0.0 | 3 |

Helm uses a 3-way-merge to perform upgrades and rollbacks by checking chart against newer/older chart and current cluster state. So if you rollback, helm will check differences between newer and older chart and cluster state, if charts are the same but status changed then rollback applies.

### Helm components
helm cli: main tool for helm commands,

charts: collection of yml docs describing a software in a consistent way. Helm charts are organized into values and templates yaml. Values files store key-value set while templates have key in the doc. So helm picks the values file and apply values inside to template docs to define a chart.

release: to apply chart by helm cli.
When chart is installed, `helm install <revision-name> <chart-name>` a revision number spun up for the specified revision.

public chart repo: as for docker hub
artifacthub.io is where charts and chart repos are listed.

helm secret: helm stores charts metadata as secrets in kubernetes cluster,

### Helm charts
Charts are insturctions manual about how to set up, build and deploy a consistent software into a k8s cluster.

Charts use templates and values, which are organized into multiple .yml files. Root of this files is the `Chart.yaml` file.

Minimalistic chart:
```
apiVersion: v2 # v1 -> helm 2, v2 -> helm 3 (jesus..)
name: dummychart
description: A Helm chart for Kubernetes

type: application # could be 'application' or 'library'
dependencies: # indicates which helm chart this one needs to get things done, no need to merge manifests for dependencies
  - ...
version: 0.1.0 # track chart version
keywords:
  ...
maintainers:
  ...
appVersion: "1.16.0"
```

Chart structure:
```
my-chart-folder
|- templates # folder containing templates
|- values.yaml
|- Chart.yaml
|- LICENSE # optional
|- README.md # optional
|- charts # dependencies as charts
```

### Working with helm: basic
`helm` cli is your main tool. It uses a gRPC style command.

`helm search some-software`
`helm search [repo|hub] some-software # specify where to find some-software`

Usually charts have a README.md file where are instructions about how to install chart.

`helm repo add repo-url # add a repo to search in for charts`
`helm install my-app chart-name`

`helm list # shows releases, you should see my-app listed here`
`helm uninstall my-app # remove everything at once about my-app release`

### Customizing chart parameters
`helm install some-name some-software` installs `some-software` with default values. We can customize default values:
- by using `--set <template-key>=<value> some-name some-software`,
- by using a custom value file and pass it by `helm install --values custom-value.yaml some-name some-software`.

You can also: `helm pull [--untar|] some-software` to get a [.tgz|folder] containing all the .yml docs to install 'some-software' consistently and change docs accordingly to your needs.

why `helm search hub wordpress`? Better check differences between hub and repo.

### Lifecycle management with Helm
Helm manage charts 'at once', all docs that define a release (say app at specific version) are managed altogether.

`helm upgrade` is no exception and generates a new release.

`helm list` show releases.
`helm history` gives more informations about release lifecycle.
`helm rollback` takes release to its previous revision but it does not go back, it spin a newer revision number. Rollback does not restore volumes, they'll stay the same.
`helm upgrade` may need some grants to perform upgrade of specific charts.

## Kustomize basics

### Kustomize problem statement & idealogy
What's kustomize: a tool to do DRY on .yaml.

Kustomize is structured with base and overlays. Base can be a common .yml file about a kubernetes object, overlays are segments of that file overriding a piece of configuration. So, kustomize
structure is:
```
...
|- base/ # basic configuration, common across all overlays
|  |- kustomization.yaml
|  |- nginx-depl.yaml
|  |- service.yaml
|  |- redis-depl.yaml
|- overlays/ # customizations
   |- dev/
   |  |- kustomization.yaml
   |  |- config-map.yaml
   |- stage/
   |  |- kustomization.yaml
   |  |- config-map.yaml
   |- prod/
   |  |- kustomization.yaml
   |  |- config-map.yaml
```

Kustomize takes the base and apply the overlay of choose to get the final manifest.

Kustomize is shipped with kubectl: `kubectl apply -k ...`, the `-k` turns on kustomize.

### Kustomize vs Helm
Do they solve the same issue? I don't think so.. Helm is a different beast. He not only allows you to parametrize manifest through templates, he provides a service, a database of resources.

### Installation
Kustomize can be installed by script.

### kustomization.yaml file
`kustomization.yaml` file contains two things:
- a list of manifests to handle,
- the customization you want to apply to those manifests
`kustomization.yaml` must stay in the root directory along the manifests.

Command `kustomize build <folder-containing-manifests>` produces the final manifests: takes the base file and applies changes.

### Kustomize output
`kustomize build <folder-containing-manifests>` does not deploy anything. `kustomize build <folder-containing-manifests> | kubectl apply -f -` does, no big deal.

It's recomended to use to delete resources too: `kustomize build <folder-containing-manifests> | kubectl delete -f -`. I guess, because of kustomize, you may be uncertain about which resources were previously produced by `build` command, so it is better to rely on kustomize on deletion too. I think because kustomize introduces a layer of indirection he also reduces readability.. Or maybe I am overthiking.

### Kustomize apiversion & kind
`kustomization.yaml` has:
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
...
resources:
  - <a-resource>
  - <another-resource>
```
as its top. They're totally optional but it is better to kept them to tell which version of kustomize was intended to be used when resources were created.

### Managing directories
We can add folders and subfolders to better organize manifests, kustomize can leverage that complexity by specifying relative paths in `resources` array of `kustomization.yaml`.

You can also provide `kustomization.yaml` in each subfolder, focused only on resources that stay in that folder. The root `kustomization.yaml` can now have `resources` array pointing to subfolders instead of the resources inside becaming leaner.

### Common transformers
What's a transformer? A tool that apply a change to a kustomize resource while it is build.

Common transformers are:
- commonLabel, adds label to kubernetes resources,
- namePrefix/nameSuffix, adds prefix/suffix to resource names,
- Namespace, adds a namespace to resources,
- commonAnnotations, adds annotation to resources.

Let's say we want to add a label to a resource, I add this to kustomization.yaml:
```
commonLabels:
  org: some-label
```
the commonLabel transformer will add that label to kustomize resources:
```
apiVersion: v1
kind: Service
metadata:
  labels:
    org: some-label
spec:
...
```
To add:
```
namespace: some-n
```
the namespace transformer will be choosed for the job changing namespace to resources.

Specifiyng this change:
```
namePrefix: dont-
nameSuffix: -that
```
the namePrefix/nameSuffix transformer will change the `metadata.name` property adding `dont-` before value and `-that` after.

CommonAnnotation transformation adds annotations to resources.

### Image transformers
We can specify this in out kustomization.yaml:
```
images:
  - name: nginx
    newName: haproxy
```
The image transformer will change image name to `haproxy` on each resource that uses `nginx`. Or use this:
```
images:
  - name: nginx_alpine
    newTag: 2.11
```
to tell image transformer to pick another tag.

In general, scope of a kustomize.yaml is defined by which resources he has to manage.

### Patches
Another method to modify kubernetes resources. Provides a way to change attributes in a very precise way.

Patch is organized by three parameters:
- operation type, could be add, remove or replace,
- target, indicates what resource will be affected, could be kind, version/group, name, namespace, labelSelector, annotationSelector,
- value, what will take place.

You can provide multiple 'target' to better point the desired resource.

So, a kustomization.yaml like this:
```
...
patches:
  - target:
      kind: Deployment
      name: dpl-name
    patch: |-
      - op: replace
        path: /metadata/name
        value: new-deployment-name
```
will replace name value of a Deployment named 'dpl-name' to 'new-deployment-name'. The `path` attribute in `patch`, indicates in an unambiguos way which attribute to change by specifying its yaml path in document tree.

There is another way to define patches, it is by 'strategic merge patch'. This is an example of the previous patch, in strategic merge way:
```
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: new-deployment-name
```
this piece of manifest will be merged with all matching resources, so basically renames all deployments to 'new-deployment-name'. Maybe using the same data to specify changes and selection params is not the best...

### Diffetent types of patches
You can specify patches in kustomization.yaml or in separate files by specifying a path for the patch:
```
patches:
  - target:
      kind: Deployment
      name: dpl-name
    path: patch-file.yml
```
`patch-file.yml` will contains the patch:
```
- op: replace
  path: /metadata/name
  value: new-deployment-name
```
The same for strategic-merge-patch.

### Patches dictionary
Let's use a patch to add an attribute:
```
patches:
  - target:
      kind: Deployment
      name: dpl-name
    patch: |-
      - op: add
        path: /spec/template/metadata/labels/some-key
        value: some-value
```
this will add a 'some-key: some-value' entry among attributes of indicated deployment. In a strategic-merge way:
```
patches:
  patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: dpl-name
    spec:
      template:
        metadata:
          labels:
            some-key: some-value
```
How to remove (Json 6902 way):
```
patches:
  - target:
      kind: Deployment
      name: dpl-name
    patch: |-
      - op: remove
        path: /spec/template/metadata/labels/some-key
```
That will remove label 'some-key' from pointed deployment. How to remove the strategic-merge way:
```
patches:
  patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: dpl-name
    spec:
      template:
        metadata:
          labels:
            some-key: null
```
Setting to `null` marks the attribute to be removed by the patch.

### Patches list
To patch a list you can rely on position of attribute of choiche in the list itself. Here is a Json6902 way example:
```
patches:
  - target:
      kind: Deployment
      name: dpl-name
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0 # to get the first in the list
        value:
          name: haproxy
          image: haproxy
```
As you may guess `0` is the element we want to patch index in the list. In this case patch changes name and image about the first container in manifest.

Here is a strategic-merge way example:
```
patches:
  patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: dpl-name
    spec:
      template:
        spec:
          containers:
            - name: nginx
              image: haproxy
```
Patch uses no index, value will be merged.

This one adds a new container by Json6902:
```
patches:
  - target:
      kind: Deployment
      name: dpl-name
    patch: |-
      - op: add
        path: /spec/template/spec/containers/- # tells to add element at the end of the list
        value:
          name: haproxy
          image: haproxy
```
This one is the same by strategic-merge:
```
patches:
  patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: dpl-name
    spec:
      template:
        spec:
          containers:
            - name: haproxy
              image: haproxy
```
since there is no match for that container, it will be added to list.

This one delete a container by Json6902:
```
patches:
  - target:
      kind: Deployment
      name: dpl-name
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0 # tells to remove first element of the list
```
This is the same declined into strategic-merge way:
```
patches:
  patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: dpl-name
    spec:
      template:
        spec:
          containers:
            - name: nginx
              $patch: delete # tells to remove matching element by merge
```
you see it relies on the `$patch: delete` indication.

### Overlays
Overlays are layer that applies to a set of manifests when specific condition matches. Given this:
```
your-kustomize-folder
|- base/
|  |- kustomization.yaml
|  |- ...
|- overlays/
   |- dev/
   |  |- kustomization.yaml
   |  |- ...
   |- stage/
   |  |- kustomization.yaml
   |  |- ...
   |- prod/
      |- kustomization.yaml
      |- ...
```
`kustomization.yaml` in base folder has nothing that we have already seen. `kustomization.yaml` in overlays's subfolders instead declares a patch that applies to a resource in the base folder:
```
bases:
  - ../../base
patch: |-
  - op: replace
    path: /spec/template/metadata/labels/some-key
    value: some-value
```

In overlays you can also define new resources that will be added to what's define in 'base'.

Can't patch deployment container to use another image and tag using Json6902 way. Damn, Overlay lab has no solution. I'll do it again. This was the right patch:
```
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: caddy
```

### Components
Reusable pieces of configuration. Useful when optional feature needs to be used in a group of overlays.

Let's say we have three overlays: A, B and C, and we want to add service1 only to overlay A and B. We cannot add service1 to base because it will be inherited by overlay C. Copy service1 manifest into A and B increase maintenance burden and is error prone.

Give our kustomize folder a new shape:
```
your-kustomize-folder
|- base/
|- overlays/
|  |- A/
|  |- B/
|  |- C/
|- components/
   |- service1/
   |  |- kustomization.yaml
   |  |- service1-depl.yaml # service1 deployment manifest
   |  |- ...
   |- service2/
```
`components` folder hosts components. Component is defined at least two files: a `kustomization.yaml` and a manifest; of course it can also contains patches and any other manifest we need to add..

The service1 `kustomization.yaml` is nothing new:
```
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - service1-depl.yaml

patches:
  - ...
```

The `service1-depl.yaml` is a manifest:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service1
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: service1
          image: nginx_alpine
```

To use a component in an overlay we have to define a specific import in the overlay's kustomization.yaml file; so, overlays A and B can simply import component service1 this way:
```
bases:
  - ../../base
components:
  - ../../components/service1
```

## Troubleshooting

### Application failure
To troubleshoot applications, start from user and then move to cluster throught each layer:
- do user interact with right layer?
- is service ok? Is it configured correctly (selectors, ports, whatever)?
- is pod ok? Check its events, use `kubectl log <pod>`.
- is backing service ok? Is it configured correctly?
- is backing service's pod ok?..

Never drop objects without getting their yaml manifest.

Always look for deployments before go for pods. If you change pod but it looks untouched, then there is a deployment that overrides your changes.

`kubectl replace --force -f <new-file>` is handy.

### Control plane failure
Control plane components can be delivered as pods or services (service kubelet status).

Can't update kube-scheduler-controlplane, don't get why. I can't do it by kubectl, I have to go after `/etc/kubernetes/manifests/kube-controller-manager.yaml`.

Also `journalctl -u kube-apiserver` in case is delivered directly on host.

When pod doesn't get files that exists, check if volume is correctly mounted.

Remember:
- `alias k=kubectl` to shortcut command,
- `source <(kubectl completion bash)` to get autocomplete about kubernetes.

Pod named with `-controlplane` as suffix in `kube-system` namespace are static so their manifest can be found in `/etc/kubernetes/manifests/`.

### Worker node failure
Check node status, then use `describe`.

Check for last node's heartbeat, also use `top` or `df -h` to check space and memory.

Check the kubelet: `service kubelet status`, or by using `sudo journalctl -u kubelet`. Check kubelet certs.

Once the node is fixed, run `ssh <the-node-to-restart> "service kubelet start"` to start node again.

The `journalctl -u kubelet` is so hard to read, there is too much stuff. I didn't perform well here...

check nodes status. ssh the failing node. do systemctl status containerd or systemctl status kubelet to check what could be wrong. Do systemctl start kubelet to start service again and see
what is happening.

What's ephemeral-storage?

Node's kubelet is configured into `/var/lib/kubelet/config.yaml` and in `/etc/kubernetes/kubelet.conf`. Btw `systemctl status kubelet` shows where service picks configurations.
I change configuration in `/var/lib/kubelet/config.yaml`, now node addresses controlplane at port 6443. However I still see wrong port after service restart... Because I was doing no restart
but start only. To get restart you have to stop then start service by `systemctl` or do `service restart kubelet`.

### Network troubleshooting

DNS
---
Kubernetes uses CoreDNS. CoreDNS is a flexible, extensible DNS server that can serve as the Kubernetes cluster DNS.

Memory and Pods
---------------
In large scale Kubernetes clusters, CoreDNS's memory usage is predominantly affected by the number of Pods and Services in the cluster. Other factors include the size of the filled DNS answer
cache, and the rate of queries received (QPS) per CoreDNS instance.

Kubernetes resources for coreDNS are:
- a service account named coredns,
- cluster-roles named coredns and kube-dns,
- clusterrolebindings named coredns and kube-dns,
- a deployment named coredns,
- a configmap named coredns,
- a service named kube-dns.

While analyzing the coreDNS deployment you can see that the the Corefile plugin consists of important configuration which is defined as a configmap.

Port 53 is used for for DNS resolution.
```
kubernetes cluster.local in-addr.arpa ip6.arpa {
   pods insecure
   fallthrough in-addr.arpa ip6.arpa
   ttl 30
}
```
This is the backend to k8s for cluster.local and reverse domains.
```
proxy . /etc/resolv.conf
```
Forward out of cluster domains directly to right authoritative DNS server.

Troubleshooting issues related to coreDNS
-----------------------------------------
If you find CoreDNS pods in pending state first check network plugin is installed.

Check if coredns pods have `CrashLoopBackOff` or `Error` state.

If you have nodes that are running SELinux with an older version of Docker you might experience a scenario where the coredns pods are not starting.
To solve that you can try one of the following options:
- upgrade to a newer version of Docker,
- disable SELinux,
- modify the coredns deployment to set allowPrivilegeEscalation to true:
  ```
  kubectl -n kube-system get deployment coredns -o yaml | \
    sed 's/allowPrivilegeEscalation: false/allowPrivilegeEscalation: true/g' | \
    kubectl apply -f -
  ```
- another cause for CoreDNS to have `CrashLoopBackOff` is when a CoreDNS Pod deployed in Kubernetes detects a loop. There are many ways to work around this issue, some are listed here:
  - add the following to your kubelet config yaml: `resolvConf: <path-to-your-real-resolv-conf-file>` This flag tells kubelet to pass an alternate resolv.conf to Pods. For systems using
    `systemd-resolved`, `/run/systemd/resolve/resolv.conf` is typically the location of the "real" resolv.conf, although this can be different depending on your distribution,
  - disable the local DNS cache on host nodes, and restore /etc/resolv.conf to the original,
  - a quick fix is to edit your Corefile, replacing forward . `/etc/resolv.conf` with the IP address of your upstream DNS, for example forward . 8.8.8.8. But this only fixes the issue for
    CoreDNS, kubelet will continue to forward the invalid resolv.conf to all default dnsPolicy Pods, leaving them unable to resolve DNS.


If CoreDNS pods and the kube-dns service is working fine, check the kube-dns service has valid endpoints.
```
kubectl -n kube-system get ep kube-dns
```
If there are no endpoints for the service, inspect the service and make sure it uses the correct selectors and ports.

Kube-Proxy
---------
`kube-proxy` is a network proxy that runs on each node in the cluster. kube-proxy maintains network rules on nodes. These network rules allow network communication to the Pods from network
sessions inside or outside of the cluster.

In a cluster configured with `kubeadm`, you can find `kube-proxy` as a `daemonset`.

`kubeproxy` is responsible for watching services and endpoint associated with each service. When the client is going to connect to the service using the virtual IP, the kubeproxy is
responsible for sending traffic to actual pods.

If you run a kubectl describe ds kube-proxy -n kube-system you can see that the kube-proxy binary runs with following command inside the kube-proxy container.
Command:
```
/usr/local/bin/kube-proxy
--config=/var/lib/kube-proxy/config.conf
--hostname-override=$(NODE_NAME)
```
So it fetches the configuration from a configuration file ie, /var/lib/kube-proxy/config.conf and we can override the hostname with the node name of at which the pod is running.

In the config file we define the clusterCIDR, kubeproxy mode, ipvs, iptables, bindaddress, kube-config etc.

Troubleshooting issues related to kube-proxy. Steps:
- Check kube-proxy pod in the kube-system namespace is running.
- Check kube-proxy logs.
- Check configmap is correctly defined and the config file for running kube-proxy binary is correct.
- kube-config is defined in the config map.

Check kube-proxy is running inside the container:
```
# netstat -plan | grep kube-proxy
tcp        0      0 0.0.0.0:30081           0.0.0.0:*               LISTEN      1/kube-proxy
tcp        0      0 127.0.0.1:10249         0.0.0.0:*               LISTEN      1/kube-proxy
tcp        0      0 172.17.0.12:33706       172.17.0.12:6443        ESTABLISHED 1/kube-proxy
tcp6       0      0 :::10256                :::*                    LISTEN      1/kube-proxy
```

How do I restart kube-proxy?

What exactly is a daemonset pod?

## JSON Path

### Introduction to YAML
It's a file format for data rapresentation.

yaml array elements could be dictionaries, so you can add `key: value` pair straightforward.

### JSON Path
`$` is root of document in query language.

Query result is always an array.

`$[?(@ > 40)]` returns all the elements that value is greater than 40. `?` means querying and `@` means 'each element'.

`$.car.wheels[?(@.location == "rear-right")].model`, for each car, query pick model of the 'rear-right' wheel.

### JSON Path usecases kubectl
`kubectl get pods -o=jsonpath='{$.items[0].spec.containers[0].image}'`

You can also add multiple jsonpath queries, `kubectl get pods -o=jsonpath='{$...}..{$..}'` to get results altogether. You can add basic format by adding:
- `{"\n"}` to add newline between results,
- `{"\t"}` to add tabs.

You can use loop statements for better formatting:
```
kubectl get pods -o=jsonpath='{range $.items[*].spec.containers[0].image}{"\t"}{$.status.capacity.cpu}{"\n"}{end}'
```
`{range ...}` and `{end}` determines start and end of the loop statement.

`kubectl get pods -o=custom-columns=<column-name>:<jsonpath-query>` adds custom column for each query.

kubectl `--sort-by=...` option does its magic even in jsonpath queries.

### Security primitives
First secure your hosts: use SSH key based authentication. kube-apiserver must be kept secure by configuring proper authentication and authorization services.

TSL certificates are used by cluster components (schedule, apiserver, kubelet...) to communicate each other in a secure way.

Network policies can reduce access between pods in a node.

### Authentication
Cluster is accessed by four kind of users:
- cluster admins,
- developers,
- end users,
- bots.

They can go on two main categories in k8s. Admins and developers are Users, bots are ServiceAccounts in k8s. End users are out of discussion because they use the application the cluster hosts, so application is in charge to authenticate end users.

k8s does not create Users, but he can create ServiceAccounts.

kube-apiserver authenticates while using kubectl or api, he can check authentication in several ways:
- static password files (yes, username and password list file),
- static token file (username and token list),
- certificates,
- identity services (LDAP, kerberos...).

Of course static password files is the easiest of the four. Arrange a .csv file:
```
password1,user1,userid1
password2,user2,userid2
password3,user3,userid3
```
and pass it to kube-apiserver as option:
```
ExecStart=/usr/local/bin/kube-apiserver ... --basic-auth-file=usr-pwd-list.csv
```
or add it to the kube-apiserver.yaml file:
```
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - name: kube-apiserver
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.3
    command:
    - kube-apiserver
    - --authorization-mode=Node,rback
    ...
    - --basic-auth-file=usr-pwd-list.csv
```
Otherwise user can be registered by http api:
```
curl -v -k https://master-node-ip:6443/api/v1/pods -u "username1:password1"
```
The .csv file can also have a fourth column indicating which group each user belongs.

Static token file works exactly the same. The .csv file looks like this:
```
some-symbol-sequence-representing-token1,user1,userid1,group1
some-symbol-sequence-representing-token2,user2,userid2,group1
some-symbol-sequence-representing-token3,user3,userid3,group2
```
and can be provided to cluster by `--token-auth-file=usr-tkn-list.csv` in the ways already discussed except the http api:
```
curl -v -k https://master-node-ip:6443/api/v1/pods --header "some-symbol-sequence-representing-token1"
```
Don't use static files as authentication, even if provided by mounting volumes. k8s has roles as components (yes, `kind: Role` and `kind: RoleBinding`).

*BEWARE:* using static password files is no longer supported in k8s 1.20.0.

### TLS basic
A TLS certificate is used to ensure confiden communication between client and server.

Asymmetric encription uses a keys pair. User generates a pair by running `ssh-keygen`, it produces two files:
- id_rsa (or whatever name user chooses), the private key,
- id_rsa.pub (or whateve name user chooses), the public key.
Public key is provided to each server user want access to (put in `~/.ssh/authorized_keys`).

Asymmetric encryption is used to share a secret and start a confident communication by symmetric encryption once secret is shared.

A certificate contains a public key and optionally other infos/metadata. Certificates and keys can be obtained by openssl:
```
openssl genrsa -out my.k8s.key 2048 (generates the private key)
openssl rsa -i my.k8s.key -pubout > my.k8s.pem (generates the public key)
```
When user hit the server by `https://something`, server provides his publick key to user. User generates a session key and encrypt that with the server's public key. Only who has server's private key is able to decrypt user's message and pick the session key to start a confident communication.

Certificates are public keys with metadata. In the metadata there is a value, a sign, which testify the certificate user has is valid. To sign a certificate a request to a certificate authority (CA) has to be sent by generating a Certificate Signing Request. CA validate informations and, if everything is fine, send back a signed certificate. CA apply sign by encrypting a specific value with its private key, this way everyone can check the sign because CA public keys are easily found (browsers ships the most importants).

To provide certificate authority and get that level of confidence in a private environment, a private CA can be hosted.

Client certificates are required by server to ensure they are what they say are.

### TLS in kubernetes
Comunication between nodes in k8s cluster must be secure.

| component | need certificate | need private key | talks with |
|:---------:|:----------------:|:----------------:|:----------:|
| kube-apiserver | [x] | [x] | etcd server, kubelet server of each node |
| etcd server | [x] | [x] | replies to kube-apiserver |
| kubelet server | [x] | [x] | kube-apiserver |
| admin user | [x] | [x] | kube-apiserver |
| kube-scheduler | [x] | [x] | kube-apiserver |
| kube-controllermanager | [x] | [x] | kube-apiserver |
| kube-proxy | [x] | [x] | kube-apiserver |

k8s requires to have a CA to validate certificates, who provides her own certificate:private key pair.

### TSL in kubernetes - certificate creation
To generate a key: `openssl genrsa -out ca.key 2048`.

To generate a certificate signing request: `openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr`. This is a certificate without sign, to sign it: `openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt` to obtain a self signed certificate.

Use the same steps to generate certificate and key for admin:
```
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -signkey ca.key -out admin.crt
```
This time CA key is used to sign admin's certificate, now admin.crt is a valid certificate testified by our cluster's CA. The attribute `/O=system:masters` qualifies the user providing this certificate as it belongs to the system:masters group, the cluster admins.

Repeats the commands for kube-scheduler, kube-controllermanager, kube-proxy need to prefix their name with 'system' in certificate generation. This completes certificate and key pair generation for client components. Remember: files should have meaningful names.

To comunicate with components is mandatory to provide certificates, keys and CA's certificate. Could be done by providing them as parameters or by using a kube-config.yaml:
```
apiVersion: v1
kind: Config
clusters:
- cluster:
    name: kubernetes
    certificate-authority: ca.crt
    server: https://kube-apiserver:6443
users:
- name: admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

All components need a copy of CA's certificate.

etcd-server needs its own certificate/pkey pair, they can be generated by steps above. etcd can be deployed as cluster, so we need additional cert/pkey pair for each etcd peer in cluster. Peer's cert/pkey must be provided as parameter while running etcd.

kube-apiserver interact with many clients who know him by several names, all those names must be specified in kube-apiserver certificate. To do that an openssl config file is necessary:
```
[req]
req_extensions = v3_req
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation,
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```
and pass the file path to certificate request command:
```
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -subj "/CN=kube-admin" -out apiserver.csr -config openssl.config.file
openssl x509 -req -in apiserver.csr -signkey ca.key -out apiserver.crt
```
kube-apiserver needs:
- etcd-cafile (the public key used by a CA trusted by etcd and kube-apiserver),
- etcd-certfile (the public key to comunicate with etcd),
- etcd-keyfile (a pkey to comunicate with kubelet),
- kubelet-certificate-authority (the public key used by a CA trusted by kubelet and kube-apiserver),
- kubelet-client-certificate (the public key to comunicate with kubelet),
- kubelet-client-key (a pkey to comunicate with kubelet),
- client-ca-file (CA cert to ensure confidentiality),
- tls-cert-file (cert file as server),
- tls-private-key-file (pkey as server).

kubelet needs:
- a CA cert (to ensure confidentiality),
- a kubelet pkey (to comunicate as server),
- a kubelet cert (to comunicate as server),
for each node. File must be registered in kubelet-config.yaml file. Certificates must provide `system:node:name.of.the.node` as CN value because kube-apiserver will use those names to grant rights.

### View certificate details
How to manage certificates? Cluster can be installed 'the hard way', so pkeys and certs are manually generated, or by 'the kubeadm way', all pkeys and certs needed are generated automatically by the tool while deploying components as pods.

To collect such infos, read the components config files or commands that were used to run them and retreive where the pkeys, certs and CA's cert are located. Use command `openssl x509 -in /path/to/cert.file.crt -text -noout` to pick common-name, issuer name, all the eventually present alternative names and expiration date of certificate. Once all the infos are collected put them in a table or dashboard to have immediate eye on them when need arises.

When there are issues the first place where to check is the log, `journalctl -u etcd.service -l` if cluster is installed 'the hard way'. Otherwise, because components are pods, `kubectl logs etcd-master` should be enought to get wich pkeys or certs were used. Docker can be useful too: `docker logs container.id`.

Commands used during test:
- kubectl get ns,
- kubectl get pods -n kube-system,
- kubectl describe kube-apiserver-controlplane -n kube-system,
- kubectl describe pod kube-apiserver-controlplane -n kube-system,
- kubectl describe pod etcd-controlplane -n kube-system,
- openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout,
- kubectl describe pod etcd-controlplane -n kube-system,
- openssl x509 -in /etc/kubernetes/pki/etcd/ca.crt -text -noout,
- openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text -noout,
- openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout,
- kubectl describe pod kube-apiserver-controlplane -n kube-system,
- openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout,
- vi /etc/kubernetes/manifests/etcd.yaml,
- docker ps -a,
- docker logs 8e45e3428d95,
- ls /etc/kubernetes/pki/,
- ls /etc/kubernetes/pki/etcd/,
- vi /etc/kubernetes/manifests/etcd.yaml,
- kubectl describe pod etcd-controlplane -n kube-system,
- kubectl get pods -n kube-system -o wide,
- kubectl describe pod etcd-controlplane -n kube-system,
- docker logs bbaf64bcb201,
- ls /etc/kubernetes/manifests/,
- cat /etc/kubernetes/manifests/etcd.yaml,
- vi /etc/kubernetes/manifests/kube-apiserver.yaml,
- systemctl daemon-reload.

### Certificates API
CA is just a pair of pkey/certificate placed in a safe place: usually the master node. This is what kubeadm does. But when number of users who need to access the cluster grows, logging to master node and generate certificates for new users or renew expired certificates could be tedious. k8s has a certificates api to automate this.

certificates api is a service:
- new user asks for certificate by sending a certificate-signing-request directly to api,
- certificate request can be reviewed and approved or refused,
- created certificates can be shared to user.

Timeline:
1. Alice is a new administrator. She generates a pkey: `openssl genrsa -out alice.key 2048`,
2. Alice generates a certificate signing request: `openssl req -new -key alice.key -subj "/CN=kube-admin" -out alice.csr`,
3. Alice sent the `alice.csr` file Bob, the older administrator,
4. Bob pick Alice's .csr and bundle it in a csr object:
```
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: alice-csr
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  signerName: kubernetes.io/kube-apiserver-client
  request:
    LS0slktncklisjlgij
```
Value in spec.request is the certificate signing request encoded base64 (for what?),
5. new certificate signing request can now be managed by kubectl: `kubectl get csr`,
6. Bob approve Alice's request: `kubectl certificate approve alice-csr`. Certificate can be seen by running `kubectl get csr alice-csr -o yaml`. Certificate is outputted in base64 format.

controllermanager is the controlplane component in charge to manage certificate signing requests. controllermanager is aware about which pkey and cert to use by respectively `--cluster-signing-key-file` and `--cluster-signing-cert-file` options.

Commands used during test:
- ls -ll /root/,
- cat akshay.csr,
- cat akshay.key,
- touch akshay.csr.yml,
- echo "-----BEGIN CERTIFICATE REQUEST-----
MIICVjCCAT4CAQAwETEPMA0GA1UEAwwGYWtzaGF5MIIBIjANBgkqhkiG9w0BAQEF
AAOCAQ8AMIIBCgKCAQEAzpsJx74zWvuhRna7/eKynVr2h+r+F4BK1wfPfZdqAVVt
6d2EhalgbdNqZ0l6hOcbnj9aosWwEGHYG3IzRq4jaIgHeiruY1yFdbz0SPeq2f+q
HX/67zKxO1JU+FDrta8prxg9YaLjRDuUxodSzgo7xI2jay2ICMIySxuHdbfjyRlw
rS4Ow+F1+Q3+2lTHJz6TakNAH2kRW8OixpXnEvU8nDeyM8LrcyZpQOuyebirP7cF
rEItEQ5+5yjNoigu3I3jeGMOMGppojACll9GyBdJEHhkGDoSi6Q8j7r75ofTnEUN
//KY8WFbw0OlyJP530jO2fbZqqQoSmnULNpywrkI8wIDAQABoAAwDQYJKoZIhvcN
AQELBQADggEBAAWxZDOClq99oJ7U8AXAc7YsoXlYLVCZtGPYK+5s0aTaUYUXFkC0
S1i0boQsttN28iQRe8+tPxe+9frpLdxmFdmpr0DfEiSLLERgf+PYP373PDEGHWne
Mo13llpGGm+QeUn31+o34cEpA7Eowfsu+XJsbGKkJzwgoNUWwmpiU4sFcSRfKnmy
0Ru8mm6MNu2aocMPBacKel9cyEnIhH4C5aiNAg3IpL17nGbLOd+cl6bgVpSkBQNm
S24NDjOujYxUOJSiiJSNv8WybkfuIw3DEk+iFW2aFrJqQttPJS2BlCAGXtXiyocN
4snI+eyxQk0cM5Ot6s5BHA7bpOuwnP5NsI4=
-----END CERTIFICATE REQUEST-----" | base64 (this must go in spec.request in a single line),
- vi akshay.csr.yml,
- kubectl create -f akshay.csr.yml ,
- kubectl get csr -o wide,
- kubectl certificate approve akshay,
- kubectl get csr -o wide,
- kubectl describe csr akshay,
- kubectl get csr agent-smith -o yaml > agent-smith.csr.yml,
- cat agent-smith.csr.yml,
- kubectl certificate deny agent-smith,
- kubectl delete csr agent-smith,

### Kubeconfig
Kubeconfig provides a place where to store configuration parameters.

By default the config file is `~/.kube/config` and is provided by default in each command. To use a different one add `--kubeconfig path.to.alternative.kubeconfig.file` on each comand.

Kubeconfig has three main sections: clusters, contexts, users. `clusters` represents configs and setups for available clusters. `users` the user accounts that have access to clusters and their certificates and pkeys. `contexts` match the other two. For example: he states which users can access a cluster or not.

Here is an example:
```
apiVersion: v1
kind: Config
current-context: dev-user@dev
clusters:
- name: sandbox
  cluster:
    certificate-authority: path.to.this.cluster.ca.crt.file
    server: https://sandbox:6443
- name: dev
  cluster:
    certificate-authority: path.to.this.other.cluster.ca.crt.file
    server: https://dev:6443
contexts:
- name: sandbox-admin@sandbox
  context:
    cluster: sandbox
    user: admin
    namespace: sandbox
- name: dev-user@dev
  (omitted for brevity)
users:
- name: sandbox-admin
  user:
    client-certificate: path.to.sandbox.admin.crt.file
    client-key: path.to.sandbox.admin.pkey.file
- name: dev-user
```
the current-context attribute tells which context is taken by default, to switch context is necessary to run `kubectl config use-contex sandbox-admin@sandbox`.

Comand `kubectl config view` shows current config.

Attribute clusters.cluster.certificate-authority can be substituted with clusters.cluster.certificate-authority-data to specify certificate itself in config file.

Commands used during test:
- echo $HOME,
- ls /root/.kube/,
- kubectl config view,
- kubectl config view --kubeconfig /root/my-kube-config,
- clear,
- kubectl config view --kubeconfig /root/my-kube-config,
- kubectl config use-context research,
- kubectl config view --kubeconfig /root/my-kube-config,
- kubectl config use-context research,
- kubectl config use-context research --kubeconfig /root/my-kube-config,
- echo $KUBECONFIG,
- cp /root/.kube/config /root/.kube/config.bkp && mv /root/my-kube-config /root/.kube/config

### Api groups
kube-apiserver is central component in cluster, he offers api to manage that cluster. Apis in kube-apiserver are organized in groups: `metrics`, `healthz`, `version`, `api`, `apis`, `logs` .. One for each resource/functionality.

`api` belongs to the core group, and exposes functionalities about:
- v1, which exposes:
  - namespaces,
  - pods,
  - rc,
  - events,
  - nodes,
  - bindings,
  - secrets...

`apis` belongs to the named group, and exposes functionalities about:
- apps, which exposes:
  - v1:
    - deployments (list, get, create, delete, update, watch),
    - replicaset,
    - statefulsets
- extensions,
- networking.k8s.io, which exposes:
  - v1:
    - networkpolicies
- storage.k8s.io,
- authentication.k8s.io,
- certificates.k8s.io, which exposes:
  - v1:
    - certificatesigningrequests

The leaves of each tree are resources. Those in brackets are verbs (commands of kubectl?).

Like rest, api and apis are browseable.

To use the api authentication by pkey/certificate is mandatory:
```
curl http://localhost:6443 -k --key admin.key --cert admin.crt --cacert ca.crt
```
Otherwise kubectl proxy may be deployed. Kubectl proxy allows to browse api on localhost:8001 and uses ~/.kube/config setup to talk with kube-apiserver.

### Authorization
There are four kind of authorization on k8s: node, ABAC, RBAC, webhooks, AlwaysAllow, AlwaysDeny.

Node authorization: kubelet and apiserver exchange informations (node status) and forward requests. Access to kube-apiserver is granted by node's group which each kubelet must be register at.

ABAC: a set of grants or permissions is defined in a policy file and bounded to a group. Updating a policy file requires a kube-apiserver restart to get changes. kube-apiserver has of course his policy enforcement point.

RBAC: see below.

Webhooks: to integrate kube-apiserver with external service who is able to tell if a request is authorizable or not.

AlwaysAllow and AlwaysDeny: as their names state.

Authorization kind can be specified by `--authorization-mode` attribute, default value is `AlwaysAllow`.

Attribute can specify multiple authorization kind. In this case each request is evaluated by each specified authorization kind in the same order are mentioned in `--authorization-mode` value. Multiple authorization kind are evaluated in an OR fashion: a request is approved if an authorization kind accept that, the request is refused when no authorization kind accept the request.

### RBAC
To create a role is mandatory to create a role object:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
```
here are again resources and verbs.

To bind a user to a certain role a RoleBinding object has to be created:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
The two section about user and role are obvious at read.

As any other component, RoleBinding can be associated to a namespace.

As any other component, kubectl provides the usual commands:
```
kubectl get roles
kubectl get rolebindings
kubectl describe role some-role-name
kubectl describe rolebinding user-role-bind-name
```
and some more:
- `kubectl auth can-i verb-name resource-name` to check if current user can perform that action,
- `kubectl auth can-i verb-name resource-name --as user-name` to check if `user-name` can perform that action.
Commands above allows to specify a namespace of course.

RBAC allow resource name granularity, user role can perform a verb on a specific resource, identified by its name:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "update"]
  resourceNames: ["pod-name1", "pod-name2"]
```

Commands used during test:
- kubectl -n kube-system describe pod kube-apiserver-controlplane | grep authorization-mode,
- kubectl get roles,
- kubectl get roles --all-namespaces,
- kubectl -n kube-system describe role kube-proxy,
- kubectl -n kube-system get rolebindings,
- kubectl -n kube-system describe rolebinding kube-proxy,
- kubectl auth can-i list pods --as dev-user,
- touch dev-user-create-list-delete-pods.yml,
- vi dev-user-create-list-delete-pods.yml,
- touch dev-user-binding.yml,
- vi dev-user-binding.yml,
- mv dev-user-create-list-delete-pods.yml developer-role.yml,
- kubectl create -f developer-role.yml,
- kubectl create -f dev-user-binding.yml,
- vi dev-user-binding.yml
- kubectl create -f dev-user-binding.yml
- cat developer-role.yml (here is the output):
  ```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "list", "delete"]
  ```
- cat dev-user-binding.yml (here is the output):
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
- kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
  ```
- controlplane $ kubectl -n blue get pods,
- controlplane $ kubectl -n blue get role -o yaml > developer-blue.yml,
- controlplane $ cat developer-blue.yml (here is the output):
  ```
  apiVersion: v1
  items:
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      creationTimestamp: "2021-04-25T09:41:04Z"
      managedFields:
      - apiVersion: rbac.authorization.k8s.io/v1
        fieldsType: FieldsV1
        fieldsV1:
          f:rules: {}
        manager: kubectl-create
        operation: Update
        time: "2021-04-25T09:41:04Z"
      name: developer
      namespace: blue
      resourceVersion: "5480"
      selfLink: /apis/rbac.authorization.k8s.io/v1/namespaces/blue/roles/developer
      uid: 24e2fba8-851e-422a-a625-fef4227c94ec
    rules:
    - apiGroups:
      - ""
      resourceNames:
      - blue-app
      resources:
      - pods
      verbs:
      - get
      - watch
      - create
      - delete
  kind: List
  metadata:
    resourceVersion: ""
    selfLink: ""
    ```
- controlplane $ vi developer-blue.yml,
- controlplane $ kubectl -n blue delete role developer,
- role.rbac.authorization.k8s.io "developer" deleted,
- controlplane $ kubectl create -f developer-blue.yml,
- role.rbac.authorization.k8s.io/developer created,
- kubectl -n blue can-i describe pod dark-blue-app --as dev-user,
- kubectl -n blue auth can-i describe pod dark-blue-app --as dev-user,
- kubectl auth can-i describe pod dark-blue-app --as dev-user,
- kubectl auth can-i describe pod --as dev-user,
- kubectl -n blue get pods,
- kubectl get roles -n blue,
- kubectl get rolebindings -n blue,
- kubectl -n blue get role -o yaml > developer-blue.yml,
- cat developer-blue.yml,
- vi developer-blue.yml,
- kubectl -n blue delete role developer,
- kubectl create -f developer-blue.yml,
- cat /var/answers/dev-user-deploy.yaml,
- vi developer-blue.yml,

(didn't do the latest)

### Cluster roles
Not all the resources or component in k8s can be organized throught namespaces, for example nodes. Those resources are organizable by 'cluster scopes'.

Resources that can be organized throught namespaces: pods, replicaset, jobs, deployments, services, secrets, roles, rolebindings, configmaps, PVC (?).

Resources that can be organized throught cluster scopes: nodes, PV (?) clusterroles, clusterrolebindings, certificatesigningrequests namespaces. These resources cannot be bounded to a namespace.

The two lists are a subset of which resources are organizable by namespaces and which are by clusterroles. To get the full list:
```
kubectl api-resources --namespaced=true (to get the namespace ones)
kubectl api-resources --namespaced=false (to get the clusterrole ones)
```
clusterroles and clusterrolebindings are the RBAC implementation about user's grant on non-namespaced resource, i.e.: create nodes.

All the resources manageable by namespaces are also manageable by clusterroles.

A clusterrole example:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```
To link a user to a clusterrole:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-adm
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

Commands used during test:
- kubectl get clusterroles,
- kubectl get clusterrolebindings,
- kubectl get clusterrolebindings cluster-admin -o yaml,
- kubectl describe role cluster-admin,
- kubectl get roles --all-namespaces,
- kubectl get clusterroles --all-namespaces,
- kubectl describe clusterrole cluster-admin,
- vi michelle.yml,
- vi michelle-clusterbind.yml,
- kubectl create -f michelle.yml,
- kubectl create -f michelle-clusterbind.yml ,
- cp michelle.yml storage-admin.yml,
- vi storage-admin.yml ,
- cp michelle-clusterbind.yml michelle-storage-admin.yml,
- vi michelle-storage-admin.yml,
- cat storage-admin.yml (here is the output):
  ```
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: storage-admin
  rules:
  - apiGroups: [""]
    resources: ["persistentvolumes", "storageclasses"]
    verbs: ["list"]
  ```
- cat michelle-storage-admin.yml (here is the output):
  ```
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: michelle-storage-admin
  subjects:
  - kind: User
    name: michelle
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: storage-admin
    apiGroup: rbac.authorization.k8s.io
  ```

### Image security
Given a simple pod definition:
```
apiVersion: v1
kind: Pod
metadata:
  name: front-end-sort-of
spec:
  containers:
  - image: nginx
    name: nginx
```
Where the image is pull from? The `image: nginx` follows docker's images name convention. If nothing has been specified images are pull from docker.io. If a private registry is needed, is sufficient to specify registry in value and use the required credentials.

Create docker private repo first by using a `docker-registry` component:
```
kubectl create secret docker-registry my-docker-registry-secret-name \
--docker-server=private-registry-url \
--docker-username=private-registry-username \
--docker-password=private-registry-password \
--docker-email=private-registry-user@email.io
```
And then use it in the pod's `spec`:
```
apiVersion: v1
kind: Pod
metadata:
  name: my-front-end-sort-of
spec:
  containers:
  - image: private-registry-url/reponame/imagename:version
    name: my-f-e-s-o
  imagePullSecrets:
  - name: my-docker-registry-secret-name
```

Commands used during test:
- kubectl get pods,
- kubectl get deployments,
- kubectl describe deployment web,
- kubectl edit deployment web,
- kubectl get deployments,
- kubectl get pods,
- kubectl create secret docker-registry private-reg-cred --docker-server=myprivateregistry.com:5000 --docker-username=dock_user --docker-password=dock_password --docker-email=dock_user@myprivateregistry.com,
- kubectl edit deployment web,
- watch "kubectl get pods".

### Security contexts
docker has its own security mechanisms k8s supports and mimic letting user to choose to get security at container level or at pod level. Security on container overrides security on pods. A couple of examples, here is securityContext at pod level:
```
apiVersion: v1
kind: Pod
metadata:
  name: my-front-end-sort-of
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - image: busybox
    name: bbox1
```
and here is securityContext at container level:
```
apiVersion: v1
kind: Pod
metadata:
  name: my-front-end-sort-of
spec:
  containers:
  - image: busybox
    name: bbox1
    securityContext:
      runAsUser: 1000
      capabilities:
        add: ["MAC_ADMIN"]
```
Capabilities are supported only at container level (don't really know what capabilities are in docker).

Commands used during test:
- kubectl describe pod ubuntu-sleeper,
- kubectl edit pod ubuntu-sleeper,
- kubectl get pod ubuntu-sleeper -o yaml > ubuntu-sleeper-secctx.yml,
- vi ubuntu-sleeper-secctx.yml,
- kubeclt delete pod ubuntu-sleeper,
- kubectl create -f ubuntu-sleeper-secctx.yml,
- kubectl exec --stdin --tty ubuntu-sleeper -- /bin/bash,
- vi ubuntu-sleeper-secctx.yml,
- kubectl delete pod ubuntu-sleeper,
- kubectl create -f ubuntu-sleeper-secctx.yml.

### Network policy
Let's divide network trafic in ingress and egress: the ingress trafic is all the requests that came into component, the egress trafic is all the requests that steam out of component.

k8s cluster is configured by default with an "All Allow" network rule: any pod can reach any other pod in cluster, no matter if they are hosted on different nodes. Cluster network can be shaped by `NetworkPolicy` components.

Pods and NetworkPolicies can be bound together by labels and selectors (as Replicasets). Let's have this:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-ingress-on-5432
  namespace:
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: app-backend
    ports:
    - protocol: TCP
      port: 5432
```
it allows ingress trafic on port 5432 coming from pods labeled 'name:app-backend' to pods labeled 'role:db'.

NetworkPolicies are not supported by all the network solutions (so network is a module or sort of in k8s).

### Developing network policies
k8s allows pods to comunicate with each other by default.

Let's say we want to prevent pods to access 'DB' pod except for 'API' pod:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    ports:
    - protocol: TCP
      port: 3306
```
Ok, the network policy above instruct pods labeled as `role: db` to allow requests that pods labeled `api-pod` do on port 3306. The `namespaceSelector` ensures that given policy applies only on a specific namespace.

Network policy has a property to specify a set of IP addresses allowed to interact the way and pods policy states:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
```
Ingress or egress criteria (indicated in the `from` array, so `podSelector` and `ipBlock` in the previous example) are read in OR way: request is allowed if satisfy just one of them. Criteria in each element of the `form` array are intended in AND: in the example, first element of `from` is satisfied fi `podSelector` and `namespaceSelector` are both satisfied.

To allow a pod to stream requests we need an egress set of rules:
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - ipBlocK:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 80
```
this case `db` pods are allowed to make requests on port 80 to a specific IP address.

When incoming traffic is allowed, reply to that traffic is allowed automatically, no need of extra rule.

Commands used during test:
- kubectl get networkpolicy --all-namespaces
- kubectl describe networkpolicy payroll-policy
- kubectl get pods --show-labels
- I create this but was not enought:
  ```
  piVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306

  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080
  ```
  I don't get why. Oh, ok, `name: internall` was correct, not the value I provide.

## Storage

### Docker storage
By default Docker stores all its data in `/var/lib/docker`:
```
+ /var/lib/docker
|- aufs
|- containers
|- image
|- volumes
```
Volumes are kept in `volumes`, images in `image`, files about containers in `containers`..

When a docker image is run, docker creates a new layer in read-write mode on top of the image (read-only) layer. If container creates a new file, it creates in its layer. If container change a file which belongs to the image layer, a copy of the updated file is created in the container layer.

When a container is removed its layer is removed too.

Volumes can be used to get some container layer's part survive while the container is removed.

Volume mount: bind a directory in `/var/lib/docker/volumes` to a specific folder in container's layer. Volume bind: bind a directory of choice to a specific folder in container's layer. Not clear about real difference between the twos...

Now Docker prefers `--mount` instead of `-v`: `docker run --mount type=bind,source=/some/place,target=/var/lib/something myapp`.

In Docker the Storage Driver is in charge to manage layers and their changes while building or running images.

### Volume driver plugins in Docker
Volume drivers are about managing container's filesystem (layer) and the host shared filesystem.

While running a container, I can choose a specific volume driver.

### Container storage interface
CSI (container storage interface) tells how k8s deals with storage runtimes, plugins implement that.

### Volumes
As said before, using volumes with containers allows to persist container's data instead of losing it when container returns and stops. The same for pods in k8s:
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers
  - image: alpine
    name: alpine
    command: ["/bin/sh", "-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
    volumeMounts:
    - mountPath: /opt
      name: app-data
  volumes:
  - name: app-data
    hostPath:
      path: /tmp/data
      type: Directory
```
This pod shares its `/opt` with host `/tmp/data`. This way `number.out` file is persisted into host and won't get wasted when pod dies.

### Persistent volumes
Having volume in pod definition file is handy but not enought, k8s allows to handle volumes in a centralized way: persistent volumes.

A persistent volume definition file:
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  hostPath:
    path: /tmp/app-data
```
Allowed `accessModes` are: `ReadWriteOnce`, `ReadWriteMany`, `ReadOnlyMany`.

Replace `hostPath.path` with some cloud one, there are many.

Persistent volume (PV) is a storage resource in the cluster provided in a static way: someone defines the PVs.

### Persistent volume claims
Persistent volume claims are component that are in charge to link volumes (can you imagine?). User defines a persistent volume claim and cluster bind that to a volume who best matches claim's constraints. If no bind is possible, claim stills in a pending state. When a volume is added, pending claims try to be bound to that new one.

Constraints can be:
- capacity,
- access mode,
- volume mode,
- storage class,
- selectors and labels

Volumes and persistent claims are boud together in a 1-to-1 relation.

Persistent volume claim definition:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvclaim1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

To revome a persistent volume claim: `kubectl delete persistentvolumeclaim pvclaim1`. Once the persistent volume is free again, attribute `peristentVolumeReclaimPolicy` tells which behavior volume will bring on:
- `Retain`: volume will exists but cannot be used by any pod,
- `Delete`: volume is automatically deleted,
- `Recycle`: volume is ready to be bound to another claim, data will be scrubbed before bounding it again.

How to use persistent volume claims (PVCs) in pods:
```
apiVersion: v1
kind: Pod
metadata:
  name: dummypod
spec:
  containers:
    - name: dummyfrontend
      image: nginx_alpine
      volumeMounts:
      - mountPath: "/var/www/html"
        name: dummyfrontendvolume
  volumes:
    - name: dummyfrontendvolume
      persistentVolumeClaim:
        claimName: pvclaim1
```
link is the `name` attribute under `volumeMounts`.

Cannot revome a PVC while is used by a running pod.

Commands used during test:
kubectl get pods
- kubectl describe pod webapp
- kubectl exec webapp -- cat /log/app.log
- kubectl get pods
- kubectl get pod webapp -o yaml > webapp.yml
- vi webapp.yml
- kubectl delete pod webapp
- kubectl create -f webapp.yml
- kubectl create -f pv-log.yml
- vi claim-log-1.yml
- kubectl create -f claim-log-1.yml
- kubectl get pvc
- kubectl get pv
- kubectl describe pv pv-log
- kubectl delete pvc claim-log-1 && kubectl create -f claim-log-1.yml
- kubectl describe pvc claim-log-1
- kubectl delete pod webapp && kubectl create -f webapp.yml
- kubectl create -f webapp.yml
- kubectl delete pvc claim-log-1
- kubectl get pvc
- kubectl get pv
- kubectl explain persistentvolume --recursive | less (handy command to get feedback about k8s components)

### Storage Class
Aims to provide storage dynamic provisioning. What we saw before is storage static provisioning. Storage classes are bound to a cloud provider (or something similar) like AWS, Azure, GCP..

A storage class definition file:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc11
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: none
  (...)
```
given this pvc:
```
apiVersion: storage.k8s.io/v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: sc11
  resources:
    requests:
      storage: 500Mi
```
can be used in pod this way:
```
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
    - name: someapp
      image: alpine
      command: ["/bin/sh",  "-c"]
      args: ["shuf -i 0-100 -n 1 >> /opt/somedata"]
      volumeMounts:
      - mountPath: /opt
        name: data-volume
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: pvc1
```
storage classes automatically build volume claims.

What is dynamic storage provisioning? I don't get it right.

Commands used during test:
- kubectl get pvc --all-namespaces
- kubectl get pvc local-pvc -o wide

## Networking
There is a lot to know here. This is one of the basic bricks to build a k8s cluster. We will go throught some Linux networking basics first.

k8s networking addresses 4 main issues:
- container-to-container communications,
- pod-to-pod,
- pod-to-service communcation,
- external-to-service communication.

To achieve that k8s imposes the following fundamentals every networking solution must implement:
- pods on a node can communicate with all pods on all nodes without NAT,
- agents on a node (kubelet, daemons, ect.) can communicate with all pods on the same node.

Every pod has a unique IP

Every pod is accessible by any other pod, regardless of what node they are on.

How containers on a pod talk to each other?
- well, containers in a pod share network namespaces, IP and MAC address, so they can talk on localhost,
- containers in a pod must  coordinate port usage (reminds me of docker-compose),
- it is all about how CNI (container network interface) is implemented (really?).

How containers on a node talk to each other?
- Node shares a root network namespace. Node `eth0` interface stays in the root networkns.
- In the same node, each pod has its own networkns with a virtual ethernet pair. One end of this pair connects pod's networkns, the other end connects node's root networkns. The end connecting pod's networkns is pod's `eth0`, a virtual interface. The end connecting node's root networkns is `vethxxx`. Pod does not know about the node, it's `eth0`.
- Node creates an ethernet bridge network and share it among all the `vethxxx` created by each hosted pod. The bridge network is the default gateway for each `veth-something` and she does routing having an ARP table with all the `vethxxx` entries. Bridge network has node's `eth0` as default gateway.

How containers on different nodes talk to each other?
- every node is assigned a unique CIDR block (a range of IP addresses) to assign to hosted pods. This way each pod has an unique IP that does not conflict with any other address.
- nodes are configured with port forwarding enabled.
- k8s require nodes to be able to route packets, but he does not bother about how.

What are services?
An abstraction about how to expose a set of pods as a network service. Service example:
```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```
service named `my-service` targets on port 9376 any pod labeled as `app=myapp`. When not using a selector service must be explicitly mapped to a network address by an ` Endpoint` object.

### Switching and routing
`ip link` shows and allows update about interfaces on board.

`ip addr add 192.168.1.99/24 dev eth0` assings an ip address to interface `eth0`.

Telling the switch about a gateway is the way to reach something out of the lan. Command `route` shows which paths a device is aware of to send packets.

A route to an external resource can be added to host this way:
```
ip route add 192.168.2.0/24 via 192.168.1.1
```
That command tells the device that to communicate with a specific external host, he must address packets to `192.168.1.1`. Usually the addressed device is a router's interface.

Device cannot be aware of all the stuff beyond the router, having a route for every device in other lans is not feasible, even impossible to map every resource on the internet. So the default gateway is the one the device asks for when he does not know to which host deliver a request or packet, a single routing table row:
```
someone@somewhere ~ $ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.1.1     0.0.0.0         UG    600    0        0 wlp4s0
link-local      *               255.255.0.0     U     1000   0        0 docker0
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0
192.168.1.0     0.0.0.0         255.255.255.0   U     600    0        0 wlp4s0
```
In the example first row tells default behavior: if no other route match send stuff to `192.168.1.1`, the gateway. Row two and thremazinga z infinitye indicate routes to connect with docker containers. Last row tells not to bother gateway to talk with anymazinga z infinity device in the same lan.
mazinga z infinity
So, given hosts A, B, C, where A is connected to B via eth0 and B by eth1 to C. In order to allow A to communicate with C we have to:
- add a route to A with the C's network ip address to send packet to B's network interface,
- add a route to C with the A's network ip address to semazinga z infinitynd packet to B's network interface,mazinga z infinitymazinga z infinitymazinga z infinitymazinga z infinityl
- update `/proc/sys/net/ipv4/ip_forward` to 1 and `/etc/sysctl.conf` entry `net.ipv4.ip_forward = 1` on host B.mazinga z infinity

### DNS
To resolve ip addresses with some labels without polluting `/etc/hosts`.

Change content of `/etc/resolv.conf` file to:
```
nameserver    192.168.1.100
```
to make the device pointing to the DNS service at `192.168.1.100`, who will resolve ip addresses by hostnames.

However, `/etc/hosts` has precedence over DNS.

Hosts that implement DNS services may point to other DNS services too.

### Domain names
A label that matches with a public ip address: www.bytebear.it.

A domain name is composed by:
- a top level domain. .it for example,
- a domain name: bytebear in our case,
- a subdomain: www.

They are also useful to group things: maps.google.it, mail.google.it... DNS services are organized as a tree, as domain names are structured. Top level DNS resolves the top domain level, the .com, lower level DNSes resolve domain names and so on. DNSes are organized in a hierarchycal structure.

By adding the following entry to `/etc/resolv.conf`:
```
nameserver    192.168.1.100
search        bytebear.it
```
a command like `ip fun` will also search for `fun.bytebear.it`, value of `search` entry will be appended.

A few type of records that are allowed in `/etc/resolv.conf` or `/etc/hosts`:
| type | name | value | note |
|:----:|:----:|:-----:|:----:|
| A | nameserver | 192.168.1.3 | stores a couple name - IPV4 address |
| AAAA | nameserver | 2001:0db8:85a3:0000:0000:8a2e:0370:7334 | stores a couple name - IPV6 address |
| CNAME | subdomain.nameserver | anothersubdomain.nameserver, againanothersubdomain.nameserver | stores domain name aliases |

To query DNS service, nslookup is the right tool:
```
$ nslookup www.google.com
Server:         127.0.1.1
Address:        127.0.1.1#53

Non-authoritative answer:
Name:   www.google.com
Address: 172.217.21.68
```
nslookup ignores `/etc/hosts` entries, it only query DNS resolution.

DNS default port is 53.

### Network namespaces
Namespace is a tool used in linux systems to isolate networks and processes.

To add a network namespace, run command `ip netns add anamespaceofchoiche`.

To create a virtual network interface, run command `ip link add veth0 type veth` (is this still true?)

To operate in a network namespace: `ip netns exec anamespaceofchoiche ip link` or `ip -n anamespaceofchoiche link`, for example. The same for `arp` or `route`:
```
ip -n anamespaceofchoiche arp
...
ip -n anamespaceofchoiche route
...
```
Namespace networks are completely separated and blind, they only aware of themselves. To link two namespace networks, use command:
```
ip link add veth0 type veth peer name veth1
```
This will creates two virtual network interfaces (veth) linked together, command `ip -n somenamespace link del veth0` deletes the link instead, automatically for both network interfaces. Then commands:
```
ip link set veth0 netns ntwname0
ip link set veth1 netns ntwname1
```
binds a virtual network interface with a namespace. Then commands:
```
ip -n ntwname0 addr add 192.168.10.1 dev veth0
ip -n ntwname1 addr add 192.168.10.2 dev veth1
```
provide specific ip addresses for each virtual network. Last but not least, commands:
```
ip -n ntwname0 link set veth0 up
ip -n ntwname1 link set veth1 up
```
turn on the two network interfaces. Both networks are aware of each other; notice that host's network is not aware of the two networks.

To set up visibility between network without getting mad, a virtual 'switch' would be useful. Linux provides a tool that solve this issue: bridge.

To create a bridge: `ip link add v-net-0 type bridge`. Of course you have to bring it up: `ip link set v-net-0 up`. To create a link between a network interface and the bridge: `ip link add veth0 type veth peer name br0`; the command creates a link between the network interfaces, creating missing interfaces automatically. Then link to the bridge the interface intendend to connect to by running:
```
ip link set veth0 netns ntwname0
ip link set br0 master v-net-0
```
Then add an ip address to bridge network: `ip addr add 192.168.10.10/24 dev v-net-0`, to make it reachable.

Now networks are configured but still blind about other networks. To make this happen the host on which network namespaces are hosted must act as gateway for each of them so, let's add some route:
```
ip -n ntwname0 route add 192.168.1.0/24 via 192.168.10.10
ip -n ntwname1 route add 192.168.1.0/24 via 192.168.10.10
```
and host must behave as NAT because returning packets about network namespaces requests must be forwarded from host eth0 to bridge interface
```
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -j MASQUERADE
```
To make host really behave as gateway respect to the network namespaces:
```
ip -n ntwname0 route add default via 192.168.10.10
ip -n ntwname1 route add default via 192.168.10.10
```

Outer networks cannot reach network namespaces. Let's say we have a web server in `ntwname1` we want to expose to the web. Then:
```
iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.10.18:2 -j DNAT
```
Now requests about port 80 hitting our host will be routed to 192.168.10.2:80, the webserver on `ntwname1`.

### Docker networking
Docker provides few options about container networking:
- `docker run --network none nginx`: container is isolated from outside world or other containers,
- `docker run --network host nginx`: container shares network interface with host. Who reach the host potentially can reach the container and container can reach anyone host can reach,
- `docker run --network bridge nginx`: it creates a network shared among containers and host. A bridge network is created by default by docker (or by docker-compose). Containers can communicate with each other and with host. It also creates a virtual network interface.

When a container is created, Docker:
- creates a namespace,
- creates a pair of interfaces, one on the bridge network and one on the container,
- creates a virtual cable between the two previous network interfaces.

Containers are not directly reachable from outside but throught host, a port mapping is mandatory.

To forward from host port to container port, Docker uses iptables with NAT. When a port mapping is specified, let's say `docker run nginx -p 8080:80`, it executes this (I think ?): `iptables -t nat -A PREROUTING -j DNAT --dport 8080 --to-destination 80`.

### CNI (Container Network Interface)
K8s uses the 'bridge' utility to create bridge network, the interfaces, pipes, virtual cables, namespace object management, assigning IP address, turning on network interfaces, NAT masquerading... all those steps required to add a container to a network and interact with it (if desired).

CNI is interface standard that software dealing with container networking can adehere to be used in K8s (or rkt or mesos...).

So CNI determines how container networking software must behave and which responsibilities must be in charge of.

Docker has its own container networking model, CNM, while k8s uses CNI. To overcome this difference, k8s creates containers in a network NONE mode, then uses CNI to attach container to bridge or whatever is the desired network.

### Cluster Networking
| k8s component | port |
|:-------------:|:----:|
| kube-api | 6443 |
| kubelet | 10250 |
| kube-scheduler | 10251 |
| kube-controllermanager | 10252 |
| services exposed by pod | 30000-32767 |
| ETCD | 2379 |
| ETCD client (in case of multi node master) | 2380 |

### Network command recap
- `ip link`,
- `ip addr`,
- `ip addr add 192.168.1.10/24 dev eth0`,
- `ip route`,
- `ip route add 192.168.1.0/24 via 192.168.2.1`,
- `cat /proc/sys/net/ipv4/ip_forward`,
- `arp`,
- `netstat -plnt`.

Commands used during test:
- kubectl get nodes,
- route,
- kubectl inspect node controlplane,
- kubectl describe node controlplane,
- ip link,
- kubectl get nodes -o wide,
- ip a,
- ip a | grep 10.14.159.3,
- arp,
- arp node1,
- arp node01,
- kubectl get nodes -o wide,
- ip link,
- ip route show default,
- ip route,
- docker ps -a,
- docker inspect 0de85d5649dc,
- netstat -nplt,
- netstat -anp | grep etcd,
- netstat -anp | grep etcd | grep 2380 | wc -l,
- netstat -anp | grep etcd | grep 2379 | wc -l.

### Pod networking
Node networking is not enought. Pod networking overcome what is missed.

k8s cluster expects that:
- each pod have an unique IP address,
- each pod is able to communicate with every other pod in the same node,
- each pod is able to communicate with every other pod on other nodes without NAT.

Suppose we have a three node cluster.

We create a bridge network for every node:
```
ip link add v-net-0 type bridge && ip link set dev v-net-0 up
```

We give an ip address on each interface we create on each node:
```
ip addr add 10.244.1.1/24 dev v-net-0 (on node1)
ip addr add 10.244.2.1/24 dev v-net-0 (on node2)
ip addr add 10.244.3.1/24 dev v-net-0 (on node3)
```

Then, every time we create a new container:
- attach a container to the network by creating a virtual network cable:
  ```
  ip link addr
  ```
- attach virtual network cable, one end to the container and one end to the bridge network:
  ```
  ip link set ... && ip link set ...
  ```
- assign IP address:
  ```
  ip -n somenetworknamespace addr add... && ip -n somenetworknamespace route add ...
  ```
- bring up the interface:
  ```
  ip -n somenetworknamespace link set...
  ```
Now all containers can communicate with others in the same node.

To let containers to communicate among nodes, adding routes does not scale. A better approach is to provide a router as default gateway for each node. Configure the router to provide connectivity among private networks.

Of course all those steps can be performed manually when a new container is created, this is what CNI is in charge of.

CNI allows to run some script on container creation, to provide it network capabilities. The script musts adhere to a specific format:
```
ADD)
# Create veth pair
# Attach veth pair
# Assign IP address
# Bring interface up
DEL)
# Delete veth pair
```
Of course the `DEL` part handle the container removal and frees his ip address.

kubelet creates containers, she looks at CNI configuration in `--cni-conf-dir=/etc/cni/net.d`, she looks for binaries in `--cni-bin-dir=/etc/cni/bin` then run the CNI script presented earlier `./net-script.sh add <container> <namespace>`.

### Container Networking Interface (CNI) in k8s
CNI defines what is in charge to the container runtime.

CNI configuration must be specified in each kubelet process that runs on each node. CNI configuration indicates where kubelet can find binaries with supported plugins, the `--cni-bin-dir=/opt/cni/bin` argument, where to find configuration files, the `--cni-conf-dir=/etc/cni/net.d` argument, indicating which plugin are in use and how.

CNI's responsibilities are:
- support arguments ADD/DEL/CHECK,
- support parameters container id, networks namespaces, etc..,
- manage IP address assignment to pods,
- returns results in a specific format.

### CNI weave works plugin
Weave works proposes an agent based solution.

Each agent stores a topology info about the entire cluster, ww solution creates its own bridge and assigns IP addresses too.

WW solution can be deployed by simply run `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(desired version | base64 | tr -d '\n')"`. This will deploy ww agents (peers) as pod, one for each node.

Commands used during test:
- ps -aux | grep kubelet | grep cni,
- ls /opt/cni/bin,
- (default cni conf dir is /etc/cni/net.d),
- ls /etc/cni/net.d (only 10-flannel.conflist was present, so flannel will be used),
- cat /etc/cni/net.d/10-flannel.conflist (check for 'plugins[].type' attribute),
- kubectl get pods
- kubectl describe pod app
- kubectl -n kube-system get pods

### IPAM (IP address management) on CNI
Node IPs can be managed by external IPAM (see weave works or calico or flannel..), here we talk about pod IPs.

Kubernetes states pod's IP management is a CNI responsibility, he not states how.

There are basically two ways/categories CNI offers to handle IP management:
- DHCP,
- host-local mode.

Let's see an example of `/etc/cni/net.d/net-script.conf`:
```
{
  "cniVersion": "0.2.0",
  "name": "mynet",
  "type": "net-script",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```
the script contains all the info a CNI IP manager needs to handle IPs about pods in a specific node.

Each CNI solution has its approach.

For example weave works CNI IPAM solution uses a predefined range of IP addresses and share them among all the pods in the cluster. Weave works uses 10.32.0.0/12 as range, that means from 10.32.0.0 to 10.47.255.255.

Commands used during test:
- kubectl get nodes,
- cd /etc/cni/net.d/,
- ls,
- cat 10-weave.conflist ,
- kubectl get pods --all-namespaces,
- kubectl get pods -n kube-system -o wide,
- kubectl get pods -n kube-system -o wide | grep weave,
- ip link (weave CNI creates a weave bridge network),
- kubectl get pods,
- kubectl get pods --all-namespace -o wide,
- kubectl get pods --all-namespace,
- kubectl get pods --all-namespaces -o wide,
- ip addr show weave,
- (to schedule a pod on a specific node)
  - kubectl describe node node03,
  - vi dummypod.yml and wrote the following:
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: busybox
  spec:
    containers:
    - name: bbox01
      image: busybox
   nodeSelector:
     kubernetes.io/hostname: node03
  ```
  the `kubernetes.io/hostname: node03` matches a specific label on node03,
  - kubectl create -f dummypod.yml,
  - kubectl get pods,
  - kubectl describe pod busybox (or kubectl exec -it busybox -- sh and run 'ip r' command).

###Service networking
Service is an abstraction to make a pod reachable from other pods, even if they are hosted on different nodes in the cluster. Services are not really a cluster component, a service does not exists in the cluster

ClusterIP: a service that make a pod reachable from other pods in cluster, but not reachable from outside.

NodePort: a service that make a pod reachable from outside the cluster.

Kubelet in cluster's node talks with kube-apiserver. When a pod is created on that node, CNI is invoked to provide new pod's networking, then kubelet tells kube-apiserver that pod has been created.

When a service is created, it lives in the cluster but not bound to any resource. The new service receives an IP address from a predefined range. The IP addresses range can be configured by --service-cluster-ip-range option while starting kube-apiserver. Once IP address is assigned, kube-proxy on each node creates a route rule allowing node's pod to reach that service.

kube-proxy can create the route rule to a service in different ways: userspace, iptables, ipvs. Iptables is the default.

`kubelet get service`: to get cluster services.
`iptables -L -t net | grep some-service-name`: to see rules kube-proxy created about that service by using iptables. kube-proxy logs the operation in its own log, `/var/log/kube-proxy.log` (path may change, it depends from installation).

Questions and commands used during test:
- how to find nodes network addresses range
  - kubectl get nodes -o wide, to see IP address values
  - ip addr, to check which network interface matches those addresses and which range she offers

- how to find pod network addresses range
  - kubectl get pods --all-namespaces
  - kubectl logs weave-net-kp74d -n kube-system
  - kubectl logs weave-net-kp74d weave -n kube-system
  - kubectl logs weave-net-kp74d weave -n kube-system | grep ipalloc

- how to find service network addresses range
  - cat /etc/kubernetes/manifests/kube-apiserver.yaml
  - cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep cluster-ip-range

- check how many kube-proxy are running in cluster
  - kubectl get nodes
  - kubectl get pods -n kube-system

- which kind of proxy kube-proxy is configured to use
  - cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep proxy
  - cat /etc/kubernetes/manifests/kube-apiserver.yaml
  - kubectl get pods -n kube-system
  - kubectl logs kube-proxy-2zc7w
  - kubectl logs kube-proxy-qsffw
  - kubectl logs kube-proxy-qsffw -n kube-system
  - kubectl logs kube-proxy-2zc7w -n kube-system

- how kube-proxy are deployed?
  - kubectl inspect pod kube-proxy-2zc7w

### Cluster DNS
Cluster DNS logs service name and its corresponding IP addres.

If service is not in the default namespace, then pod can access service by calling hostname.namespace.

hostname.namespace.svc.cluster.local: to access service from pods.

Pods are logged in DNS too. They are reachable by hitting their IP address as hostname, but dots are substituted by dashes. So pod with IP address 10.27.0.8 will have 10-27-0-8 as hostname.

### CoreDNS in k8s
How k8s implements DNS? k8s DNS logs each pod's hostname and IP address.

CoreDNS is the default implementation of k8s's DNS. CoreDNS is deployed as pods by a deployment object with a replicaset. Once deployed, a service is created to let pods reach the DNS. k8s automatically add DNS as entry in pod's `/etc/resolv.conf`, kubelet is in charge of that, kubelet's config file has an entry with the DNS server address.

Moreover, in each pod's `/etc/resolv.conf` are added `search` entries to resolve cluster.local, svc.cluster.local and default.svc.cluster.local for services only, pods aren't in `/etc/resolv.conf`.

CoreDNS uses a configuration specified in `/etc/coredns/Corefile`. Option `pods insecure` allows pods to be logged in DNS table...

`kubectl get configmap -n kube-system` to change CoreDNS configuration.

Questions and commands used during test:
- Identify the DNS solution implemented in this cluster:
  kubectl get pods -n kube-system

- What is the name of the service created for accessing CoreDNS:
  kubectl get services --all-namespaces

- How is the Corefile passed in to the CoreDNS POD:
  by a configmap object (run kubectl describe deployments coredns -n kube-system, you can see configuration file is provided by mounting a config volume)

- What is the name of the ConfigMap object created for Corefile:
  kubectl get configmap -n kube-system

- What name can be used to access the hr web server from the test Application:
  kubectl get services
  kubectl describe service web-service (inspect service to find clues about which pods are affected, usually a selector is involved)

- We just deployed a web server - webapp - that accesses a database mysql - server. However the web server is failing to connect to the database server. Troubleshoot and fix the issue. They could be in different namespaces. First locate the appliations. The web server interface can be seen by clicking the tab Web Server at the top of your terminal:
  (webapp and mysql are not in the same namespace) kubectl get pods --all-namespaces (to find in which namespace mysql pod is)
  kubectl get deployments
  kubectl get pods -n payroll
  kubectl get services -n payroll
  kubectl get deployments webapp -o yaml > webapp.yaml
  vi webapp.yaml and check environment variable (spec.containers.env part). Change the variable value with the right hostname
  kubectl apply -f webapp.yaml

- nslookup mysql.payroll > /root/nslookup.out

### Ingress
Ingress is a configurable service to configure and manage exposed pods (node-ports or load-balancers) outside the cluster.

To use Ingress you deploy an ingress controller (reverse proxy?), configured by a set of rules: the ingress resources.

Ingress controller implementations: GCP and GCE, Nginx, Contour, HAProxy, traefik, Istio.

To get an Ingress controller you create a deployment of a specific image.
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress-controller
        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
      args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/configmap-metadata-name
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
```
mandatory values:
- `args` parameter /nginx-ingress-controller must be provided,
- based on nginx, the needed configuration can be provided by a configmap component instead of modifying config files,
- `env` parts are environment variables nginx requires,
- `ports` indicates which ports are used by the ingress controller.

Of course you need a service to expose the ingress controller:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  - port: 443
    targetPort: 443
    protocol: TCP
    name: https
  selector:
    name: nginx-ingress
```

Ingress component does additional work to keep an eye on ingress resources and interact with the api-server. To do that, ingress controller needs a service account, `roles`, `clusterroles` and `rolebindings`.

What are ingress resources? Configuration that applies to ingress controller.

Ingress resource tell how ingress controller must route incoming traffic.

An ingress resource:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-something
spec:
  backend:
    serviceName: service_name
    servicePort: 80
```
that you create by the usual command `kubectl create -f ingress-resource-name-file.yml`. That resource tells ingress to route incoming traffic to service `service_name`.

Suppose we want to route traffic hitting url 'www.something.com'. The following resource will do the job:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-something
spec:
  backend:
    serviceName: service_something
    servicePort: 80
```
The following rule instead handles requests and routes them to service based on their path:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-something-somethingelse
spec:
  rules:
  - http:
      paths:
      - path: /something
        backend:
          serviceName: service_something
          servicePort: 80
      - path: /somethingelse
        backend:
          serviceName: service_somethingelse
          servicePort: 80
```
The following rule routes traffic by domain:
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-something-somethingelse
spec:
  rules:
  - host: something.my_service.com
    http:
      paths:
      - backend:
          serviceName: service_something
          servicePort: 80
  - host: somethingelse.my_service.com
    http:
      paths:
      - backend:
          serviceName: service_somethingelse
          servicePort: 80
```
You see, `host` property substitues `path` of the previous one.

If you miss `host` property, resource applies to all domains (really?).

Command `kubectl describe ingress ingress-something-somethingelse` shows resource's rules.

Ingress resource has always a default backend property. If ingress finds no matching rule for that resource, then request is forwarded to the url default backend property indicates. This means you have to deploy a service for the default backend routing. So when user asks for a non existing url because of the path part, then default backend service will serve that request.

Recap: dns drives requests to our node service, which forward requests about a specific domain to ingress controller, which read its rules and route requests to the service ingress resources told.

Questions and commands used during test:
- Which namespace is the Ingress Resource deployed in?
  `kubectl get ingress --all-namespaces`
- What is the Host configured on the ingress-resource?
  `kubectl describe ingress ingress-wear-watch -n app-space`
- You are requested to change the URLs at which the applications are made available. Make the video application available at /stream.
  `kubectl edit ingress -n app-space` and I changed the path about video app's backend. Alternative way requires to dump of ingress at app-space, `kubectl describe ingress ingress-wear-watch -o yaml > somewhere.yaml`, edit that file, remove the app-space's ingress, `kubectl delete ingress ingress-wear-watch -n app-space` and recreate that ingress, `kubectl apply -f somewhere.yaml`.
- You are requested to add a new path to your ingress to make the food delivery application available to your customers. Make the new application available at /eat.
  `kubectl get services -n app-space` to get info about 'food' service and `kubectl edit ingress -n app-space` to edit app-space's ingress accordingly.
- You are requested to make the new application (pay application) available at /pay.
  First I explored services in target namespace, `kubectl get services -n critical-space`, then I created a new ingress file to expose the new service, `vi critical-space-ingress.yml`, then I create a new ingress for critical-space namespace, `kubectl create -f critical-space-ingress.yml`.

- Let us now deploy an Ingress Controller. First, create a namespace called 'ingress-space'.
  I create a namespace using the following definition:
  ```
  apiVersion: v1
  kind: Namespace
  metadata:
    name: ingress-space
  ```
- The NGINX Ingress Controller requires a ConfigMap object. Create a ConfigMap object in the ingress-space. Spec: name = nginx-configuration.
  I create a configmap using the following definition:
  ```
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: nginx-configuration
    namespace: ingress-space
  ```
- The NGINX Ingress Controller requires a ServiceAccount. Create a ServiceAccount in the ingress-space. Specs: name = ingress-serviceaccount
  I create a serviceaccount using the following definition:
  ```
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: ingress-serviceaccount
    namespace: ingress-space
  ```
- Let us now deploy the Ingress Controller. Create a deployment using the file given.
  The given file had the following content:
  ```
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: ingress-controller
    namespace: ingress-
  spec:
    replicas: 1
    selector:
      matchLabels:
        name: nginx-ingress
    template:
      metadata:
        labels:
          name: nginx-ingress
      spec:
        serviceAccountName: ingress-serviceaccount
        containers:
          - name: nginx-ingress-controller
            image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
            args:
              - /nginx-ingress-controller
              - --configmap=$(POD_NAMESPACE)/nginx-configuration
              - --default-backend-service=app-space/default-http-backend
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            ports:
              - name: http
                containerPort: 80
              - name: https
                containerPort: 443
  ```
  I fixed it this way:
  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ingress-controller
    namespace: ingress-space
  spec:
    replicas: 1
    selector:
      matchLabels:
        name: nginx-ingress
    template:
      metadata:
        labels:
          name: nginx-ingress
      spec:
        serviceAccountName: ingress-serviceaccount
        containers:
          - name: nginx-ingress-controller
            image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
            args:
              - /nginx-ingress-controller
              - --configmap=$(POD_NAMESPACE)/configmap-metadata-name
              - --default-backend-service=app-space/default-http-backend
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            ports:
              - name: http
                containerPort: 80
              - name: https
                containerPort: 443
  ```

- Let us now create a service to make Ingress available to external users. Name: ingress, Type: NodePort, Port: 80, TargetPort: 80, NodePort: 30080, Use the right selector.
  I created a service using the following definition:
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: ingress
    namespace: ingress-space
  spec:
    type: NodePort
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 30080
      name: http
    - port: 443
      targetPort: 443
      protocol: TCP
      name: https
    selector:
      name: ingress-controller
  ```
  Cannot get how to obtain the right selector's name..
  In solution the command `kubectl -n ingress-space expose deployment ingress-controller --name ingress --port 80 --target-port 80 --type NodePort --dry-run=client -o yaml > ingress-svc.yaml` was used to obtain most of the following definition, namespace and nodePort were added later:
  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: ingress
    namespace: ingress-space
  spec:
    type: NodePort
    ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      nodePort: 30080
      name: http
    selector:
      name: nginx-ingress
  ```
  Ok now I got it: selector name's value must match with ingress controller matchlabels name's value.

- Create the ingress resource to make the applications available at /wear and /watch on the Ingress service.
  I created an ingress resource using the following definition:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-something-somethingelse
  namespace: app-space
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          serviceName: wear-service
          servicePort: 8080
      - path: /watch
        pathType: Prefix
        backend:
          serviceName: video-service
          servicePort: 8080
```

## Design and install a Kubernetes Cluster

### Design a Kubernetes Cluster

### Choosing Kubernetes Infrastructure
cloud foundry container runtime
vagrant

### Configure HA
Consider to have more than 1 master node to avoid SPOF.
Master nodes have istances of the same services: api server,controller manager, etcd, scheduler...
kubeconfig file has master node ulr..
A load balancer manage incoming requests among master nodes, kubecontrol utility must point to that load balancer.

Multiple scheduler and controller manager instances run in an active-standby mode (as they share a single thread). There is a leader election protocol that decides which instance will satisfy the next request. All the controller manager instances race to get pre-emption on the kube-controller-manager endpoint component. The first who get control of that component is the leader and will satisfy the next request.

Scheduler follows a method similar to controller manager.

ETCD can be part of each master node (this is called 'stacked topology'). It is risky, if a node goes down controlplane and cluster integrity is lost.
ETCD can be external to cluster (external etcd topology).

### ETC in HA
Etcd ensure HA by leader election. ETCD uses RAFT protocol to elect leader.

When write arrives, leader processes the write request and ensure that other instances in cluster get a copy of the updated value.

If request reach a non-leader node, he forward request to leader.

A write is considered complete if the leader gets an ack from the quorum of n/2+1 instances in cluster.

Beware of network segmentations and quorum among resulting network subsets.

### Install k8s the kubeadm way
Looks like steps I did were right. I have to provide network capabilities (POD Network) and then join nodes.

Does iptables work correctly on nodes? Check it, use as reference the install kubeadm page.

Ok lsmod | grep br_netfilter shows that br_netfilter is loaded on each node.

I have to add a k8s.conf file with the following content:
```
br_netfilter
```
in /etc/modules-load.d/. Then I have to add a k8s.conf file with the following content:
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```
in /etc/sysctl.d/. Once done both, run `sysctl --system` to update system with new configuration. This has to be done on each node. I did by hand but I have to update my ansible playbooks.

Don't remember if I installed docker's official gpg and apt repo...

I have to add a `daemon.json` file in /etc/docker with the following content:
```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```
then restart docker. Ok but what is this about? I don't get that.. To setup the docker daemon and force it to rely on systemd. On each node.
I also have to create a directory: `mkdir -p /etc/systemd/system/docker.service.d`. On each node.
Don't think run `sudo systemctl daemon-reload && sudo systemctl restart docker` is necessay once I put that in ansible. On each node.

kubeadm, kubelet and kubectl are installed on each node already.

cgroup driver is skippable because I am using docker.
