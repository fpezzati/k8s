# Kubernetes (from now on k8s)
It provides an easy way to orchestrate containers.
k8s has `object` entities as main actors. Objects can be of different kind, each one is defined by its own .yml file which contains an entry, `kind`, telling which type of object this one is about.

`minikube start` starts a VM on your pc. This VM hosts a node and a node hosts one or more objects. Each object is defined by a config file, each object has its own configfile. A config file has a `metadata` attribute which includes a `name` one, which tells us who the object is.

A k8s node matches with a VM minikube starts. A node contains a kube-proxy and one or more objects.

k8s allows a declarative approach to deployment. Also imperative is allowed of course.

## Service config file
Threre are four types of services: ClusterIP, NodePort, LoadBalancer, Ingress.

### NodePort
Exposes a node object port to the outside. It tells kube-proxy to pass him some requests, for examples all the http requests on a specific port.
```
apiVersion: v1
kind: Service
metadata:
  name: client-node-port
spec:
  type: NodePort
  ports:
    - port: 3050
      targetPort: 3000
      nodePort: 31515
  selector:
    component: web
```
This configuration is saying that we want to pass traffic to component marked with label web. k8s uses a label system to identify objects in node. The labels entry is used to specify a key value map to identify objects. Requests will be pass throught port 3050 and moved to port 3000 of object labeled component web. So port is what the NodePort service exposes to other node objects. targetPort tells which port of target object the service should reach. nodePort tells wich port will be exposed to the outside world. If none is specified, minikube (?) assigns a random one between 30000 and 32767.

## Pod
Pod is a group of one or more containers. Pod should hosts a set of homogeneus containers or containers that have a strong relation. Pod can be seen as a composition. Pod is the smallest unit of deployment in k8s.

## kubectl
`kubectl apply -f object.configuration.file.we.choose.to.use.yml` runs an object configuring it as specified in -f file.

`kubectl get pods`: shows all the pods of your cluster.

`kubectl get services`: shows all the services of your cluster.

`kubectl delete -f object.configuration.file.we.previously.choose.to.use.yml` removes all the k8s items indicated in the file.

A k8s cluster is made by a bunch of k8s nodes. One of them is the master, that is in charge to deal with user's commands and apply them to the cluster.
You pass to master node your deployment file. I each node there is a docker ready to host your containers. Containers in different pods are isolated from container hosted in other pods.
Once you ask master to run some instances of an image, master spread the requested number of instances throught all the available pods.

Probably the declarative approach will be the easiest and safer one: you don't run commands against your k8s cluster but you update your deploy file and tell master to handle that. But how master node is aware that the file you provide is an update of a previous one rather than a brand new one? By file's methadata name attribute and its kind (pod, service...). Name and kind provide a unique identifier in k8s.

You can't update everything in your pod k8s file but only the following properties:
- `spec.containers[*].image`,
- `spec.initContainers[*].image`,
- `spec.activeDeadlineSeconds` or `spec.tolerations`.

## Deployment
It is a collection of identical pods monitoring their status in order to keep them solid with the config file. It is an object more suitable in production than the `pod`. `deployment` has a configuration file having a `pod template` which tells how should any pod handled by this deployment look like.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
    component: web
  template:
    metadata:
    labels:
      component: web
    spec:
      containers:
        - name: client
          image: stephengrider/multi-client
          ports:
            - containerPort: 3000
```
That is a `deployment` config file:
- attributes metadata and name need no explanation
- replica tells how many pods of the type indicated in the template section deployment should handle,
- selector matchLabels marks the created pods with this label in order to link them to the deployment. What selector indicates must be kept in the pod's label attribute,

Notice: as far as I `kubectl delete -f my-pod-config-file.yml` the service I deployed at the same time was untouched. As I deploy a `deployment`, the service was still proxying calls, that reach the deployment's pod instead of the pod this time. So, it looks `pod`, `service`, `deployment` share the same network by default. My service found the pod in deployment because has the same selector key:value property that the single pod.

Something does not sound right in port mappings..

Beware! k8s is not aware of image changes! If you left the same image name in your .yml file, k8s does not check if image was updated so you can't update your pods. You have to change the image name in the file, maybe changing its version. This is a big inconvenient you can overcome in a few ways:
- delete all the pods hosting containers that run the obsolete image. This is a prone-error approach that can cause serious harm. Not recomended at all,
- provide a real version tag value to image name and update the .yml file accordingly to have the latest one. Looks ok to me but, it is a non-trivial step to do (no variables or command params allowed in k8s config files),
- provide a real version tag value to image name and use an imperative command to tell k8s to update the pods.

So looks like the last is the best of the three. Use this command:
```
kubectl set image deployment/client-deployment client=fpezzati/fibsequence_client:1.0.0
```
The `set` param tells to update a certain property in the cluster. `deployment/client-deployment` tells which kind and which object of that kind we'll update. The value after the `=` will replace the value in the `client` container hosted by our `client-deployment` deployment object. Pods looks recreated (short `AGE` value).

It turns out I have two docker: the one that I installed once ago and the one that is shipped with minikube. So my docker CLI is pointing to the docker server I installed. By typing `eval $(minikube docker-env)` I can temporary tell my docker CLI to match with the k8s (minikube) server, so I can see how many and which one containers it is running. Command affects the current terminal window, change it or shut it down and you'll be back to normal setup.

kubectl has commands that partially mimic what docker cli offers you. For example `kubectl logs the-pod-id-you-want-to-read-logs` or `kubectl exec -it the-pod-id-you-want-to-connect-to sh`.

## ClusterIP
What is a ClusterIP? It is a Service that allows pods to comunicate inside the node. ClusterIP does not expose ports outside the node.
