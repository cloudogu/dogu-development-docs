# From a Helm chart to a Dogu V3

This guide creates a Dogu V3 chart for XWiki. It includes the existing [XWiki Helm chart](https://github.com/xwiki-contrib/xwiki-helm) as a dependency.

At the end, XWiki is available at `https://quickstart.k3ces.localdomain/xwiki/`.

CES authentication, the Warp menu, backups, and upgrades are outside the scope of this guide.

The result of this guide (including the [advanced guide](advanced-guide_en.md)) can be found in the [xwiki-dogu folder](xwiki-dogu).

## Prerequisites

First, set up the [local development environment](prerequisites_en.md). Then select its kubeconfig:

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

The existing password is retained when `helm upgrade --install` is run again.

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

## Allow access from the gateway

The CES environment blocks incoming traffic in the `ecosystem` namespace by default. The gateway therefore needs permission to reach the XWiki pod.

Create `templates/networkpolicy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ .Release.Name }}-gateway
  labels:
    app.kubernetes.io/name: xwiki
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  # Apply the NetworkPolicy to the XWiki pod
  podSelector:
    matchLabels:
      app.kubernetes.io/name: xwiki
      app.kubernetes.io/instance: {{ .Release.Name }}
  # Control incoming connections
  policyTypes:
    - Ingress
  ingress:
    # Allow access from the CES gateway
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Release.Namespace }}
          podSelector:
            matchLabels:
              k8s.cloudogu.com/component.name: k8s-ces-gateway
      # Allow the internal XWiki port
      ports:
        - port: {{ .Values.xwiki.service.internalPort }}
          protocol: TCP
```

The bundled MySQL chart already creates its own NetworkPolicy for the database.

## Download the dependency

```shell
helm dependency build .
```

Helm downloads the XWiki chart into the `charts/` directory. Its MySQL dependency is already included in the package. The generated `Chart.lock` file records the selected XWiki chart version.

## Validate and install the chart

```shell
helm lint .

helm upgrade --install xwiki . \
  --namespace ecosystem \
  --wait \
  --timeout 15m
```

`helm upgrade --install` installs the release on its first run. On subsequent runs, Helm updates the existing installation. The first startup can take several minutes.

## Check the installation

```shell
helm status xwiki --namespace ecosystem

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

## Remove the chart

```shell
helm uninstall xwiki --namespace ecosystem
```

Helm removes the application, but the PVCs containing its data remain. To start again without the old data, delete them as well:

```shell
kubectl delete persistentvolumeclaim \
  xwiki-data-xwiki-0 \
  data-xwiki-mysql-0 \
  --namespace ecosystem
```
