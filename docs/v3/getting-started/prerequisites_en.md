# Set up the local development environment

This guide creates a complete Cloudogu EcoSystem in a local k3d cluster. The other examples use this environment.

The cluster is intended only for development and testing. It has one node and uses local storage. Volume expansion and backups are not supported.

## What you need

- a Linux computer with enough free resources;
- credentials for the Cloudogu registries;
- Docker;
- Git;
- Go 1.26 or newer;
- [k3d](https://k3d.io/);
- [kubectl](https://kubernetes.io/docs/tasks/tools/);
- [Helm](https://helm.sh/docs/intro/install/);
- curl;
- a text editor such as nano;
- jq; and
- [mikefarah/yq](https://github.com/mikefarah/yq) version 4 or newer.

Partners receive the credentials from their Cloudogu contact. The installation downloads protected CES components, so the public repositories alone are not enough.

On its first run, the installation script builds a small helper program. This requires Go.

Check the tools:

```shell
docker version
git --version
go version
k3d version
kubectl version --client
helm version
curl --version
jq --version
yq --version
```

This guide was tested with Docker `29.7.2`, Go `1.26.1`, k3d `5.9.0`, kubectl `1.36.3`, Helm `3.20.2`, jq `1.7`, and mikefarah/yq `4.53.3`.

## Get the installation repository

```shell
git clone https://github.com/cloudogu/k8s-ecosystem.git
cd k8s-ecosystem
git checkout 252bddc62b36537e1caadefeff21c14c9bd7f7bc
```

The checkout selects the revision tested with this guide. It prevents the installation scripts from changing unnoticed.

## Add the credentials

Copy the configuration template:

```shell
cp k3d/config.env.template k3d/config.env
chmod 600 k3d/config.env
```

Passwords in this file use Base64 encoding. Base64 is not encryption. Do not share or commit this file. The repository already excludes it through `.gitignore`.

Enter the password and encode it with Base64:

```shell
read -rsp 'Registry password: ' REGISTRY_PASSWORD
printf '\n'
printf '%s' "${REGISTRY_PASSWORD}" | base64 -w0
printf '\n'
unset REGISTRY_PASSWORD
```

The command prints the encoded password. Copy this value.

Then open `k3d/config.env`:

```shell
nano k3d/config.env
```

Change these eight fields. Enter the same username and encoded password in all four sections:

```dotenv
LOCAL_REGISTRY_PROXY_USERNAME="<username>"
LOCAL_REGISTRY_PROXY_PASSWORD="<Base64-encoded password>"

DOGU_REGISTRY_USERNAME="<username>"
DOGU_REGISTRY_PASSWORD="<Base64-encoded password>"

IMAGE_REGISTRY_USERNAME="<username>"
IMAGE_REGISTRY_PASSWORD="<Base64-encoded password>"

HELM_REGISTRY_USERNAME="<username>"
HELM_REGISTRY_PASSWORD="<Base64-encoded password>"
```

The credentials are now stored in `k3d/config.env`. The remaining commands do not have to run in the same terminal.

## Get the Dogu V3 configuration

Download the two prepared files to the root of `k8s-ecosystem`:

```shell
curl -fsSLo .blueprint-override.yaml \
  https://raw.githubusercontent.com/cloudogu/dogu-development-docs/main/docs/v3/getting-started/code/prerequisites/.blueprint-override.yaml

curl -fsSLo .ecosystem-core-values-patch.yaml \
  https://raw.githubusercontent.com/cloudogu/dogu-development-docs/main/docs/v3/getting-started/code/prerequisites/.ecosystem-core-values-patch.yaml
```

These files enable LOP-IDP and the processing of `Exposition` resources. They also set the fixed local address `quickstart.k3ces.localdomain`. The relaxed password policy and self-signed certificate are for local development only.

## Create the EcoSystem

```shell
./k3d/ces-k3d create quickstart
```

The installation can take several minutes. The name `quickstart` is fixed because it must match the address in the two configuration files.

After a successful installation, the command prints values like these:

```text
URL:
  https://quickstart.k3ces.localdomain

Dedicated kubeconfig:
  /home/<user>/.kube/quickstart.k3ces.localdomain

Registry stack:
  push:    localhost:5001
  consume: k3d-registry-proxy.localhost:5000
```

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
./k3d/ces-k3d stop quickstart
./k3d/ces-k3d start quickstart
./k3d/ces-k3d list
```

Remove the environment:

```shell
./k3d/ces-k3d delete quickstart
```

Then remove the `quickstart.k3ces.localdomain` entry from `/etc/hosts`. The shared registry containers and their storage remain in place.

Next: [Turn an existing Helm chart into a Dogu V3](quickstart_en.md).
