# Release a Dogu V3

This guide explains how to publish a Dogu V3 to the Cloudogu OCI registry. The Helm chart is the release artifact. Pushing the chart starts the import into the Dogu Registry.

## Before you start

You need:

- a Dogu V3 Helm chart that runs in a Multinode EcoSystem;
- Helm 3.8 or newer;
- Docker, if you publish non-public container images; and
- registry access provided by Cloudogu.

Cloudogu provides the following access information:

- a Harbor robot account;
- the assigned Dogu namespace; and
- permission to pull and push charts and, if required, images in that namespace.

The robot account is a technical account for `registry.cloudogu.com`.

## Request and manage registry access

There is currently no self-service for partner registry accounts. Contact your Cloudogu contact person to request or change access.

### Request access

Include this information in the request:

- your company and technical contact;
- the Dogu name.

### Remove access

Ask your Cloudogu contact person to disable the robot account when it is no longer needed. Include the account name, namespace and requested removal date. Renew an expired account only when another release is planned.

## Log in

Log in with the robot account. Helm asks for the username and password:

```shell
helm registry login registry.cloudogu.com
```

If you also publish images, log in with Docker:

```shell
docker login registry.cloudogu.com
```

Do not store the password in the chart, source repository or a shell script. Use the credential store of your CI system for automated releases.

## Prepare the release

Update `Chart.yaml` before every release:

- `version` is the Dogu version and must follow semantic versioning;
- `appVersion` is the version of the packaged application; and
- `annotations.dogu.cloudogu.com/api-version` must be `v3`.

`version` and `appVersion` are two independent versions and can therefore differ.

The chart name and version identify the published artifact. Do not reuse a version that was already released.

Download dependencies, validate the chart and create the package:

```shell
helm dependency build .
helm lint .

mkdir -p dist
helm package . --destination dist
```

The chart should also be installed and tested in the supported Multinode EcoSystem before it is published. The [Dogu V3 release checklist](../concepts/multinode-environment_en.md#checklist-before-a-release) lists the relevant tests.

For a public source repository, we recommend tagging the commit that contains the release:

```shell
git tag -a "release/chart/<version>" -m "<version>

chart-version: <version>
app-version: <app-version>

this is a Dogu API v3 release"

git push origin "release/chart/<version>"
```

## Publish Dogu-specific container images

List every image used by the chart in `chart-patch-tpl.yaml`. This file describes the original image addresses and where they are used in `values.yaml`, so the images can be mirrored for an offline EcoSystem and the addresses can be adjusted. [Prepare images for later mirroring](../getting-started/advanced-guide_en.md#prepare-images-for-later-mirroring) explains the structure of the file.

If the chart uses only public images from other registries, you can skip the following push steps.

Publish all Dogu-specific container images before the chart. Use the assigned namespace and the image path for Dogu V3:

```shell
docker tag "<local-image>:<tag>" \
  "registry.cloudogu.com/<namespace>/dogu/v3/images/<image>:<tag>"

docker push \
  "registry.cloudogu.com/<namespace>/dogu/v3/images/<image>:<tag>"
```

The image reference in the chart must use the same address and tag.

## Publish the chart

Push the chart only after all required images are available:

```shell
helm push "dist/<dogu>-<version>.tgz" \
  "oci://registry.cloudogu.com/<namespace>/dogu/v3/charts"
```

This push is the Dogu V3 release. The DCC automatically imports the chart and reads the metadata from `Chart.yaml` and the other files in the chart. A separate `dogu.json` upload or DCC request is not required.

## Verify the release

Read the chart from the OCI registry:

```shell
helm show chart \
  "oci://registry.cloudogu.com/<namespace>/dogu/v3/charts/<dogu>" \
  --version "<version>"
```

## Possible errors

- `401 Unauthorized`: the username or password is incorrect;
- `403 Forbidden`: the account does not have the required permission for the selected namespace; and
- an existing version cannot be replaced: choose a new chart version.
