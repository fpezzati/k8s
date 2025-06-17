# Helm!
I thought I was confident with helm but I was wrong.

Rearrangin stuff.

Helm is a package manager: instead of a bunch of .yaml manifests, with helm you can manage them as a single resource.

Helm does a lot of things so he offers a lot of commands.

## Core concepts
Helm is built around a few concepts.

### Chart
Collection of yml manifests that define an application. Helm charts are organized into values and templates yaml.

### Release
An instance of a chart running on cluster.

### Repo
A place where charts are stored, more or less a repo in a docker hub.

helm repo list shows repos.

`helm repo list`: shows all known repos.

`helm repo update <repo-name>`: update repo's metadata.

There is no bound between repo and chart. A repo can host multiple charts, a chart can came from anywhere. Command `helm search repo <keyword>` finds repos by keyword, could be the only way to point chart to repo.. So, there is no way to find original url of a chart, you guess and hope for best (that's incredible to me) unless chart was signed.

### Hub
What the hell is a hub? Where repo are stored. Ok, but how to get a repo, which has its own url, from a hub?

## helm install chart
There are a few ways of installing charts:
- by chart reference: helm install mymaria example/mariadb,
- by path to a packaged chart: helm install mynginx ./nginx-1.2.3.tgz,
- by path to an unpacked chart directory: helm install mynginx ./nginx,
- by absolute URL: helm install mynginx https://example.com/charts/nginx-1.2.3.tgz,
- by chart reference and repo url: helm install --repo https://example.com/charts/ mynginx nginx,
- by OCI registries: helm install mynginx --version 1.2.3 oci://example.com/charts/nginx.

## Things I am bad at with helm

### Given a local chart repo, do some maintenance, check if there are problems during installation and fix them before install the chart.

### Given a local chart repo, install a new version of specific chart.
Straightforward: `helm repo update repo-name` will pull latest from that repo, including chart updates.

### Perform a rollback of a deployed local chart.

### Uninstall a release.
This is straightforward:
- find the release you want to uninstall by running `helm list`,
- then execute `helm uninstall release-you-find-earlier` to remove that release.

### Find url of specific chart.
There is no reference between charts and repos, even if you get that chart from that repo. The only way to match repos and charts is to perform a search by keywords related to the chart: `helm search repo some-word --list-repo-url` and cross your fingers, amongs all the matches (they're not that much tbh) there is the one you're looking for and its url.

### Differences between repo and hub
Command `helm search hub some-word` finds across all charts helm's ArtifactHub hosts. Repo is a remote collection of charts you add to your helm client, `helm search repo some-word` will search among repos the client knows.

### Update a local helm chart with newer version recently released.
Straightforward: `helm repo update repo-name` will pull latest from that repo, including chart updates.

### Which relation bounds repo to release?
Release is the instance of a chart you download from a repo. In a few steps:
- `helm list -A` and you'll find out what releases are running in your cluster throught all namespaces,
- `helm search repo release-name-you-found-earlier` to get from which repo the release's chart came from.
Remember: helm resources are namespaced.

helm repo list
helm repo update lvm-crystal-apd
helm search repo lvm-crystal-apd (find where chart is)
helm show chart lvm-crystal-apd/fluent-bit (gives you info about the chart)
helm upgrade lvm-crystal-apd lvm-crystal-apd/fluent-bit --version=0.48.5 --set replicaCount=3 --set kind=Deployment -n crystal-apd-ns

Stuff from readme.md..
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

### Add a repo and get template from a chart in there
To find a repo about kafka run `helm find hub kafka`, it gives you a lot of results, but they all are useless:
```
https://artifacthub.io/packages/helm/bitnami/kafka	32.2.15                    	4.0.0                   	Apache Kafka..
```
Try instead run `helm find hub kafka --list-repo-url` to get useful info:
```
https://artifacthub.io/packages/helm/bitnami/kafka	32.2.15                    	4.0.0                   	Apache Kafka is a distributed streaming platfor...	https://charts.bitnami.com/bitnami
...
```
Stupid helm, first url he provides is the hub url, useless if you want to install charts. To do so you have to get the right property adding flag `--list-repo-url`, which makes the second url to appear. That is the one you have to use to install a chart!

Add it as repo:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Repos are stored in `~/.config/helm/repositories.yaml`.

Now let's get the chart. Command `helm install` allows you to install a chart, but how to do that from a repo? Command should be `helm install kafka-chart-name-of-choice kafka-chart-name --repo https://charts.bitnami.com/bitnami`, but you have to know `kafka-chart-name` first.

To get the chart name you can run `helm search repo bitnami`. Remember? `bitnami` was how I labeled the repo I added earlier. Sad thing is, all this didn't help, still getting a bulk of charts in return
```
bitnami/airflow                             	24.1.2       	3.0.1        	Apache Airflow is a tool to express and execute...
bitnami/apache                              	11.3.15      	2.4.63       	Apache HTTP Server is an open-source HTTP serve...
bitnami/apisix                              	5.0.3        	3.12.0       	Apache APISIX is high-performance, real-time AP...
bitnami/appsmith                            	6.0.10       	1.76.0       	Appsmith is an open source platform for buildin...
bitnami/argo-cd                             	9.0.20       	3.0.6        	Argo CD is a continuous delivery tool for Kuber...
bitnami/argo-workflows                      	12.0.6       	3.6.10       	Argo Workflows is meant to orchestrate Kubernet...
bitnami/aspnet-core                         	7.0.7        	9.0.6        	ASP.NET Core is an open-source framework for we...
....
bitnami/wildfly                             	24.0.8       	36.0.1       	Wildfly is a lightweight, open source applicati...
bitnami/wordpress                           	24.2.10      	6.8.1        	WordPress is the world's most popular blogging ...
bitnami/wordpress-intel                     	2.1.31       	6.1.1        	DEPRECATED WordPress for Intel is the most popu...
bitnami/zipkin                              	1.3.5        	3.5.1        	Zipkin is a distributed tracing system that hel...
bitnami/zookeeper                           	13.8.3       	3.9.3        	Apache ZooKeeper provides a reliable, centraliz...
```

better grep that. Running `helm search repo bitnami | grep kafka` I get the entry I was looking for
```
bitnami/kafka                               	32.2.15      	4.0.0        	Apache Kafka is a distributed streaming platfor...
```

Running `helm pull bitnami/kafka` dowloads the chart as .tar.gz.

Running `helm install mykafka kafka --repo bitnami --dry-run > /tmp/kafka-manifest.yaml` generates the manifest, and some note at bottom, about the chart.
