# Helm!
I thought I was confident with helm but I was wrong.

Rearrangin stuff.

## Things I am bad at with helm
Given a local chart repo, do some maintenance, check if there are problems during installation and fix them before install the chart.

Given a local chart repo, install a new version of that chart.

Perform a rollback of a deployed local chart.

Uninstall a release.

Find url of specific chart.

Difference between chart and hub.

Update a local helm chart with newer version recently released.

Update a given repo.

Which relation bounds repo to release?

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
