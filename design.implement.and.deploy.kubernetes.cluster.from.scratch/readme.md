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
