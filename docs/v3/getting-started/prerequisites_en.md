# Set up the local development environment

This guide creates a complete Cloudogu EcoSystem in a local k3d cluster. The other examples use this environment.

The cluster is intended only for development and testing. It has one node and uses local storage. Multiple nodes, volume expansion and backups are not supported.

## What you need

- a Linux computer with enough free resources
- credentials for the Cloudogu registries
- Docker
- Git
- Go 1.26 or newer
- [k3d](https://k3d.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- jq
- [mikefarah/yq](https://github.com/mikefarah/yq) version 4 or newer

Partners receive the credentials from their Cloudogu contact. The installation downloads protected CES components, so the public repositories alone are not enough.

## Create the EcoSystem

```shell
./code/init_ecosystem
```

The installation can take several minutes. The name `quickstart` is fixed because it must match the address in the two configuration files.

## Use the cluster

Select the environment's dedicated kubeconfig:

```shell
export KUBECONFIG="${HOME}/.kube/quickstart.k3ces.localdomain"
kubectl cluster-info
```

Make the address known on your computer:

```shell
sudo sh -c 'echo "127.0.0.2 quickstart.k3ces.localdomain" >> /etc/hosts'
```

Use the IP printed by the CLI if it differs from this example.

Check the installation:

```shell
kubectl get pods
kubectl get components
kubectl get crd expositions.k8s.cloudogu.com
```

Wait for the required components and pods to become available. In particular, `k8s-service-discovery` and `lop-idp` must reach the `available` state.

Your browser shows a warning on the first request because the local certificate is self-signed. You can then open the CES at:

```text
https://quickstart.k3ces.localdomain
```

## Local registry

The local registry has two addresses:

- Use `localhost:5001` to push from your computer.
- Use `k3d-registry-proxy.localhost:5000` for images inside the cluster.

Do not use `localhost:5001` in a Helm chart. Inside a pod, `localhost` refers to the pod itself.

## Stop or remove the environment

Run these commands in the `k8s-ecosystem` directory:

```shell
# stop and start the cluster
./code/k8s-ecosystem/k3d/ces-k3d stop quickstart
./code/k8s-ecosystem/k3d/ces-k3d start quickstart
# list all clusters
./code/k8s-ecosystem/k3d/ces-k3d list
# remove the cluster
./code/k8s-ecosystem/k3d/ces-k3d delete quickstart
# (re)create the cluster
./code/k8s-ecosystem/k3d/ces-k3d create quickstart
```

After deletion, remove the `quickstart.k3ces.localdomain` entry from `/etc/hosts`. The shared registry containers and their storage remain in place.

Next: [Turn an existing Helm chart into a Dogu V3](quickstart_en.md).
