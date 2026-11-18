# From a Helm chart to a Dogu V3

This guide creates a Dogu V3 chart for XWiki. It includes the existing [XWiki Helm chart](https://github.com/xwiki-contrib/xwiki-helm) as a dependency.

At the end, XWiki is available at `https://quickstart.k3ces.localdomain/xwiki/`.

CES authentication, the Warp menu, backups, and upgrades are outside the scope of this guide.

The result of this guide (including the [advanced guide](advanced-guide_en.md)) can be found in the [xwiki-dogu folder](xwiki-dogu).

## Prerequisites

First, set up the [local development environment](prerequisites_en.md) including `cldk`. Then select its kubeconfig:

```shell
export KUBECONFIG="${HOME}/.kube/quickstart.k3ces.localdomain"
kubectl cluster-info
```

## Create the parent chart

Create a directory for the Dogu chart:

```shell
mkdir xwiki-dogu
cd xwiki-dogu
mkdir templates
```

Create `Chart.yaml`:

```yaml
apiVersion: v2
name: xwiki
description: XWiki as a Dogu V3
type: application
# Version of the Dogu chart
version: 1.7.1-1
# XWiki version
appVersion: "18.6.0"
annotations:
  # Dogu API version
  dogu.cloudogu.com/api-version: v3
dependencies:
  - name: xwiki
    version: 1.7.1
    repository: https://xwiki-contrib.github.io/xwiki-helm
```

The new chart is the parent chart. It contains the Dogu V3 metadata and includes the XWiki chart `1.7.1` as a dependency.

Dogu V3 stores this information in `Chart.yaml`. It does not use a separate `dogu.json`.

## Configure XWiki

Create `values.yaml` in the same directory as `Chart.yaml`:

```yaml
# Values for the included XWiki chart
xwiki:
  fullnameOverride: xwiki

  # Pull the public XWiki image directly from Docker Hub
  image:
    name: docker.io/library/xwiki
    tag: 18.6.0-mysql-tomcat
    pullPolicy: IfNotPresent

  # Make XWiki available below /xwiki
  extraEnvVars:
    - name: CONTEXT_PATH
      value: xwiki

  # Start XWiki only after MySQL is available
  initContainers:
    database:
      enabled: true

  # Adjust the Kubernetes probes to the /xwiki path
  probes:
    startup:
      httpGet:
        path: /xwiki/rest
    liveness:
      httpGet:
        path: /xwiki/rest
    readiness:
      httpGet:
        path: /xwiki/rest/wikis/xwiki/spaces

  mysql:
    auth:
      rootPassword: ""
      username: xwiki
      password: ""
      database: xwiki
```

The XWiki chart already enables MySQL as its default database. The user and database are still named `xwiki`. When both password fields are empty, the included MySQL chart generates random passwords during the first installation. It stores them in the `xwiki-mysql` Kubernetes Secret. XWiki reads the user password from the same Secret.

The existing password is retained when `cldk dogu dev-push` is run again.

## Add an Exposition

An `Exposition` makes XWiki available through the CES gateway. Create `templates/exposition.yaml`:

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: Exposition
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: xwiki
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  http:
    - name: web
      service: {{ .Values.xwiki.fullnameOverride }}
      port: {{ .Values.xwiki.service.externalPort }}
      path: /xwiki
```

## NetworkPolicies

By default, the CES environment blocks incoming traffic in the `ecosystem` namespace with NetworkPolicies.
So, add NetworkPolicies where needed to allow incoming traffic between pods of the Dogu.
In this example, the bundled MySQL chart already creates its own NetworkPolicy for the database.

NetworkPolicies for the Exposition/Gateway are not necessary, since these are generated automatically for each Exposition.

## Download the dependency

```shell
helm dependency update .
```

Helm downloads the XWiki chart into the `charts/` directory. Its MySQL dependency is already included in the package. The generated `Chart.lock` file records the selected XWiki chart version.

## Set up `cldk`

`cldk dogu dev-push` deploys the working state of the chart as a Dogu to the cluster. It pushes the chart under a new dev version to the local registry and creates the Dogu CR. The Dogu operator then installs the Dogu from this chart.

Create the `cldk.yaml` file:

```yaml
# Dogu namespace in the registry
namespace: quickstart
# The chart is in the same directory
chart: .
```

The settings of the local environment go into the `.env` file:

```shell
cat > .env <<EOF
KUBECONFIG=${HOME}/.kube/quickstart.k3ces.localdomain
CLDK_DEV_KUBE_NAMESPACE=ecosystem
CLDK_DEV_REG=localhost:5001
CLDK_DEV_CLUSTER_REG=k3d-registry-proxy.localhost:5000
CLDK_DEV_PLAIN_HTTP=true
EOF
```

`cldk` pushes to `localhost:5001`; the cluster pulls the chart via `k3d-registry-proxy.localhost:5000`. Both addresses belong to the [local registry](prerequisites_en.md#local-registry).

To keep `cldk.yaml` and `.env` out of the chart, create the `.helmignore` file:

```text
cldk.yaml
.env
```

All options are described in [Deploying a dogu with `cldk dogu dev-push`](https://github.com/cloudogu/cldk/blob/develop/docs/operations/deploying_a_dogu_with_dev_push_en.md).

## Install the Dogu

First check what `cldk` would do. The dry run shows the target cluster, the chart address, and the Dogu CR without changing anything:

```shell
cldk dogu dev-push --dry-run
```

Then push and install:

```shell
cldk dogu dev-push
```

Every run creates a new dev version, e.g. `1.7.1-1-dev.20261117103428`. After changing the chart, run `cldk dogu dev-push` again.

## Check the installation

`cldk` does not wait for the installation. Wait until the Dogu shows `True` in the `READY` column:

```shell
kubectl get dogu xwiki --namespace ecosystem --watch
```

The first startup can take several minutes. Then check the resources:

```shell
kubectl get statefulset,pod,service,exposition,networkpolicy \
  --namespace ecosystem \
  --selector app.kubernetes.io/instance=xwiki

kubectl get persistentvolumeclaim \
  xwiki-data-xwiki-0 \
  data-xwiki-mysql-0 \
  --namespace ecosystem
```

The `xwiki` and `xwiki-mysql` StatefulSets must both be `1/1` ready. The `xwiki-data-xwiki-0` and `data-xwiki-mysql-0` PVCs must have the `Bound` status.

Open this URL in a browser:

```text
https://quickstart.k3ces.localdomain/xwiki/
```

The certificate warning is normal for the local, self-signed environment. XWiki then opens its setup wizard.

## Remove the Dogu

```shell
kubectl delete dogu xwiki --namespace ecosystem
```

The Dogu operator removes the application, but the PVCs containing its data remain. To start again without the old data, delete them as well:

```shell
kubectl delete persistentvolumeclaim \
  xwiki-data-xwiki-0 \
  data-xwiki-mysql-0 \
  --namespace ecosystem
```
