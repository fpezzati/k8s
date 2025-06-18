# CRDs
They look cool. You know what resources are in kubernetes, pods, deployments, configmaps.. CRDs are not part of kubernetes, they are blueprints about new types, hand crafted. They are used to create custom resources, kubernetes non-native objects that fulfill specific tasks.

CRDs are replacing helm about resource management.

##

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
