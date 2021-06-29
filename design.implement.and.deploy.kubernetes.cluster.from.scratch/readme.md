# Design implement and deploy kubernetes cluster from scratch
Maybe course could be out of date.. Well, whatever. To build my master node I need:
-[x] kubectl,
-[x] virtualbox (not really),
-[x] minikube,
-[x] helm,
-[] visual studio code,
-[] kubectx and kubens (?),
-[] kubetail (?).

I already have kubectl and minkube. I need no virtualbox because my machines are RPis. Now let's focus on components to put into master node. Of course this will go into an ansible playbook.

### kubectl
A good to have but boring to translate in ansible's idiom apt installation:
```
sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2 curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```
Or a trivial download:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" > /tmp/kubectl && chmod +x /tmp/kubectl && sudo mv /tmp/kubectl /usr/local/bin/kubectl
```

### minikube
Do I need minikube? Should I install `kubeadm` instead?

### helm
Helm can be downloaded and installed by:
```
curl -L https://get.helm.sh/helm-v3.5.2-linux-amd64.tar.gz > /tmp/helm.tar.gz && tar -xzvf /tmp/helm.tar.gz -C /tmp && sudo mv /tmp/linux-amd64/helm /usr/local/bin/ && rm -r /tmp/linux-amd64
```

### kubectx and kubens
kubectx is a tool to manage and switch between kubectl contexts. So it should be useful. kubens is an utility to switch between k8s namespaces, it should be useful too.
```
curl -L https://github.com/ahmetb/kubectx/releases/download/v0.9.1/kubectx_v0.9.1_linux_x86_64.tar.gz > /tmp/kubectx.tar.gz && tar -xzvf /tmp/kubectx.tar.gz -C /tmp && sudo mv /tmp/kubectx /usr/local/bin/kubectx
```
Download one (kubectx) and get two (kubectx, kubens).

### kubetail
From the github project page:

Bash script that enables you to aggregate (tail/follow) logs from multiple pods into one stream. This is the same as running "kubectl logs -f " but for multiple pods.

ok. Let's download this too. It's only a file..
```
curl https://raw.githubusercontent.com/johanhaleby/kubetail/master/kubetail > /tmp/kubetail && chmod +x /tmp/kubetail && sudo mv /tmp/kubetail /usr/local/bin/kubetail
```

Ok. Let's get feet wet. `minikube start` will start a cluster; because I already have one, I have to ignore that (default started by minikube) and create a new empty one. Command `kubectl config view` shows all existing clusters. For example, I have a cluster named 'minikube' I previously create:
```
fpezzati@oem-OMEN-by-HP-Laptop-15-ce0xx ~ $ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/fpezzati/.minikube/ca.crt
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/fpezzati/.minikube/profiles/minikube/client.crt
    client-key: /home/fpezzati/.minikube/profiles/minikube/client.key
```
ok, let's stop the cluster, `minikube stop -p minikube`, and start a new one: `minikube start -p dummycluster`. Now, kubectl gives me:
```
fpezzati@oem-OMEN-by-HP-Laptop-15-ce0xx ~ $ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/fpezzati/.minikube/ca.crt
    server: https://192.168.59.2:8443
  name: dummycluster
contexts:
- context:
    cluster: dummycluster
    user: dummycluster
  name: dummycluster
current-context: dummycluster
kind: Config
preferences: {}
users:
- name: dummycluster
  user:
    client-certificate: /home/fpezzati/.minikube/profiles/dummycluster/client.crt
    client-key: /home/fpezzati/.minikube/profiles/dummycluster/client.key
```
And now in ~/.minikube/profiles/ I have both dummycluster and minikube. I can choose which one to start by specifying the right name after the `-p` option when starting minikube. Uh oh minikube always starts the default profile... To run my new dummycluster I have to run `minikube start -p dummycluster` to really start dummycluster, otherwise default cluster will be started (the one named minikube).

Minikube came out with a bunch of addons ready to be used, I only have to enable them:
```
minikube addons enable ingress
minikube addons enable registry
minikube addons enable heapster
```
What are addons and what are those I have (not yet to be honest) enabled? `ingress` manages external http/https access to the cluster (do I access cluster throught service too? boh..). `registry` should be a credentials manager to pull docker images from private registries.. I think. `heapster` has been retired, I guess it was a monitoring tool.. Well, I don't need it anymore.

So minikube is my cluster (single node, lots of pods) for now. Let's deploy:
- create an image of something (see ./dummy.nginx.app/Dockerfile) by running `docker build -t dummy_nginx .`,
- create deploy docs for kubernetes (see ./dummy.nginx.app/deployment/deployment.yaml and service.yaml).
To make it VERY simple: `deployment.yaml` tells which images deployed containers will run, `service.yaml` tells which port cluster expose outside and which port it pair inside the cluster. By running:
```
fpezzati@oem-OMEN-by-HP-Laptop-15-ce0xx dummy.ngnix.app (master) $ kubectl create -f deployment/
Error from server (Invalid): error when creating "deployment/service.yaml": Service "dummy_nginx" is invalid: metadata.name: Invalid value: "dummy_nginx": a DNS-1035 label must consist of lower case alphanumeric characters or '-', start with an alphabetic character, and end with an alphanumeric character (e.g. 'my-name',  or 'abc-123', regex used for validation is '[a-z]([-a-z0-9]*[a-z0-9])?')
unable to recognize "deployment/deployment.yaml": no matches for kind "Deployment" in version "v1"
```
you get some errors. You can't use `_` in names. Moreover `kind` is case sensitive and `apiVersion` in the deployment file should be `apps/v1`. You have to be careful. And you eventually have some errors because of already created services/pods/whatever in the previous attempts.

Uh oh I did something wrong my cluster is not going up again... I'll destroy it and start again. I run `minikube delete -p dummycluster` to remove existing dummycluster cluster.

Still wrong. Never a win at first try.. Image was wrong and now I am stuck with a cluster hosting a wrong image, that's why dashboard shows everything red. Now I can throw away the cluster or update the image specified in the deployment document.

Don't want to trash my cluster, I'll update images. Easiest way: build again my image with a different tag (to see that the cool way: I will deliver a newer version):
```
kubectl set image deployment/dummy-nginx dummy-nginx=dummy-nginx:1.0
```
No better. Pod's log tells he was unable to pull the image... It is because minikube searches for images on docker hub by default..

Interesting solution (I have to investigate) about using a private registry here: https://stackoverflow.com/questions/42564058/how-to-use-local-docker-images-with-minikube (see Fahad answer).

Btw I use command `minikube cache add dummy-nginx:1.0` to put minikube aware of my image. It worked, but I struggle some time to find which ip:port I have to use to access my service (and so my app) in the cluster. For example, command `kubectl get service -o yaml` gives me back a lot of info but not the ip I need (another ip instead). Using `minikube service list -p dummycluster` instead prints a nice view of services indicating which ip and port are exposed by whom. There I find the ip I need to reach my dummy app.

Ok, my dummycluster works.

## k8s architecture
```
+master node(s)
|-kubecontroller-manager
|-+kube-apiserver: here resources are published (resources may have the form label - url)
| |-etcd: an in-memory map used to store api-server resources
|-kube-scheduler: is in charge to deliver requests to workers or resources
|-cloud-controller-manager: a connector to provider hosted clusters (say AWS for example)

+worker node(s)
|-kubelet: starts containers in node
|-kube-proxy: keeps track of containers among pods or other components
```

### kube-proxy
It is a pod that runs on each node which is in charge to manage network traffic. kube-proxy handles services.

### Services and Pods
Services are usually used to allow comunication among Pods. Services host selector while Pods have labels, so Services applies something that are bound to Pods matching by labels. For example, service
```
apiVersion: v1
kind: Service
metadata:
  name: dummy-nginx
spec:
  selector:
    app: dummy-nginx
...
```
applies to:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dummy-nginx
spec:
  selector:
    matchLabels:
      app: dummy-nginx
...
```
because labels match. Pods came in three flavours:
- init container,
- sidecar container (have a secondary role, bring feature to app pods, i.e: a log collector),
- app container (a main actor in application architecture, i.e.: a web server).

Pods are organized in replicaset and daemonset. Replicaset is in charge to keep a certain number of pods hosting an application (or a piece of that application) at any time. Daemonset is in charge to keep a certain number of copies of a pod on every worker node.

Statefulset pods: are pods who have to satisfy some constraints like startup or teardown order, having a specific pod name, stay bound to a specific storage.

### Job and Cronjob
Well, guess what they do, they do on pods.

### Ingress
Ingress is mainly the component implementing service discovery and loadbalancing.

### ConfigMap
Stores non confidential data in a dictionary. Allows to separate configuration from pods or other components.

### Secret

### Persistent Volumes & Persistent Volume Claims

### NameSpaces

### Manifest files
In ./deployment folder there are two manifest files. A manifest file tells characteristics that a k8s component (a service, a set of pods, whatever) must offer in a cluster. A manifest ususally has the following attributes:
- apiVersion: indicates group and version of the manifest (i.e.: apps/v1). You can get a full list of available values by running `kubectl api-versions`,
- kind
- metadata
- spec

### Kubectl
The main tool to interact with k8s clusters. Kubectl behavior is configured by its config file (~/.kube/config).

I can switch context by running `kubectl config use-context some context` to switch kubectl on another cluster.

## Helm package manager
Helm is an advanced deployment tools that collects and define interaction between cluster components by aggregating manifest files in a chart. The wording 'package manager for kubernetes' is quite misleading I think. Helm helps in managing the cluster. You can also download existing charts. Helm bundles manifest files and charts in a chart to make deployment easy. Having a lot of manifests or existing charts Helm is a must-to-have tool.
