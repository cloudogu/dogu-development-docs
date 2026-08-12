# Package the chart and investigate problems

This page extends [From a Helm chart to a Dogu V3](quickstart_en.md). It covers the local OCI workflow and some diagnostic commands.

## Push the chart to the local OCI registry

In a new terminal, first change to the `xwiki-dogu` directory. Then set the chart version:

```shell
export CHART_VERSION='1.7.1-1'
```

Download the dependencies and package the chart:

```shell
helm dependency build .
helm lint .

mkdir -p dist
helm package . --destination dist
```

Push the package to the local registry:

```shell
helm push "dist/xwiki-${CHART_VERSION}.tgz" \
  oci://localhost:5001/quickstart/charts \
  --plain-http
```

Pull the chart again to check the result:

```shell
mkdir -p /tmp/xwiki-chart

helm pull oci://localhost:5001/quickstart/charts/xwiki \
  --version "${CHART_VERSION}" \
  --plain-http \
  --destination /tmp/xwiki-chart
```

This verifies that the local registry can store the Helm chart. Releasing it to the Cloudogu registry is outside the scope of this guide.

## Install from the OCI registry

The pushed chart can be installed directly with Helm:

```shell
export KUBECONFIG="${HOME}/.kube/quickstart.k3ces.localdomain"

helm upgrade --install xwiki \
  oci://localhost:5001/quickstart/charts/xwiki \
  --version "${CHART_VERSION}" \
  --plain-http \
  --namespace ecosystem \
  --wait \
  --timeout 15m
```

The package contains the parent chart, its values, and the XWiki chart dependency.

This command also installs the chart directly with Helm. The Dogu operator does not manage this release yet.

## Investigate problems

Start with the Helm release and pod status:

```shell
helm status xwiki --namespace ecosystem
helm get values xwiki --namespace ecosystem

kubectl get pods --namespace ecosystem \
  --selector app.kubernetes.io/instance=xwiki
kubectl get events --namespace ecosystem --sort-by=.lastTimestamp
```

### The XWiki image cannot be pulled

```shell
kubectl describe pod xwiki-0 --namespace ecosystem
```

The parent chart pulls the public image directly from Docker Hub:

```text
docker.io/library/xwiki:18.6.0-mysql-tomcat
```

The cluster needs access to Docker Hub.

### MySQL or the database check does not start

```shell
kubectl describe pod xwiki-mysql-0 --namespace ecosystem
kubectl logs xwiki-mysql-0 --namespace ecosystem
kubectl logs xwiki-0 --container wait-for-db --namespace ecosystem
```

The `xwiki-mysql-0` pod must be ready before XWiki can start.

The randomly generated database passwords are stored in the `xwiki-mysql` Secret. To show the user password for diagnostics:

```shell
kubectl get secret xwiki-mysql \
  --namespace ecosystem \
  --output jsonpath='{.data.mysql-password}' | base64 --decode
echo
```

### A PVC is not bound

```shell
kubectl get persistentvolumeclaim --namespace ecosystem
kubectl describe persistentvolumeclaim xwiki-data-xwiki-0 --namespace ecosystem
kubectl describe persistentvolumeclaim data-xwiki-mysql-0 --namespace ecosystem
```

Both PVCs must have the `Bound` status. The local quickstart environment provides the required storage through the `local-path` StorageClass.

### The URL returns `502 Bad Gateway`

Check the Service, endpoints, and NetworkPolicy:

```shell
kubectl get service,endpointslice --namespace ecosystem \
  --selector app.kubernetes.io/instance=xwiki

kubectl get networkpolicy xwiki-gateway \
  --namespace ecosystem \
  --output yaml
```

The CES gateway needs access to port `8080` of the XWiki pod.

### XWiki does not become ready

```shell
kubectl describe pod xwiki-0 --namespace ecosystem
kubectl logs xwiki-0 --namespace ecosystem
```

The startup, readiness, and liveness probe paths must start with `/xwiki`. The pod must also have `CONTEXT_PATH=xwiki`:

```shell
kubectl get statefulset xwiki --namespace ecosystem \
  --output jsonpath='{.spec.template.spec.containers[0].env}'
echo
```

### The Exposition is not processed

```shell
kubectl describe exposition xwiki --namespace ecosystem
kubectl logs deployment/k8s-service-discovery-controller-manager \
  --namespace ecosystem
```

The `Valid` and `IngressesReady` conditions must be `True`.

## Clean up

Remove only XWiki and MySQL:

```shell
helm uninstall xwiki --namespace ecosystem
```

The PVCs remain. Delete them only when the stored data should be removed as well:

```shell
kubectl delete persistentvolumeclaim \
  xwiki-data-xwiki-0 \
  data-xwiki-mysql-0 \
  --namespace ecosystem
```

Remove the complete cluster from the `k8s-ecosystem` directory:

```shell
./k3d/ces-k3d delete quickstart
```

The registry containers and registry storage remain. Other local CES environments can reuse them.
