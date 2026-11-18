# Release a Dogu V3

This guide publishes a Dogu with `cldk dogu release` to the Cloudogu OCI registry `registry.cloudogu.com`. The Helm chart is the release artifact: as soon it is pushed to the registry, the dogu will be released.

## Prerequisites

- a Dogu repository with a Helm chart and a `cldk.yaml` that has been tested in the Multinode EcoSystem, e.g. with `cldk dogu dev-push` in the [local EcoSystem](../getting-started/quickstart_en.md)
- [`cldk`](https://github.com/cloudogu/cldk/blob/develop/README.md)
- Docker or another Docker-compatible builder if the Dogu builds its own container images
- registry access from Cloudogu

## Steps

### 1. Request registry access

There is currently no self-service for partner registry accounts. Access is requested and changed through the responsible Cloudogu contact person. The request should contain:

- company and technical contact person;
- name of the Dogu.

Cloudogu then provides a registry account for `registry.cloudogu.com`, the assigned Dogu namespace, and permission to pull and push charts and images in this namespace.

When the access is no longer needed, the registry account should be deactivated. The request for it should contain the account name, namespace, and desired deletion date. An expired account is only renewed if another release is planned.

### 2. Prepare the release

In `Chart.yaml`:

- `version` is the Dogu version and follows semantic versioning. A version that has already been published must not be reused.
- `appVersion` is the version of the included application. It is independent of `version`.
- `annotations.dogu.cloudogu.com/api-version` is `v3`.

In `cldk.yaml`, set the assigned Dogu namespace and the release tags of the images, see [Release tags in `cldk.yaml`](https://github.com/cloudogu/cldk/blob/develop/docs/operations/releasing_a_dogu_en.md#release-tags-in-cldkyaml).

All images used by the chart are listed in `mirror-patch-tpl.yaml` so that they can be mirrored for an offline EcoSystem. The section [Prepare images for later mirroring](../getting-started/advanced-guide_en.md#prepare-images-for-later-mirroring) explains its structure.

Load the chart dependencies if `Chart.yaml` declares any. The chart repositories of the dependencies must be added with `helm repo add` beforehand:

```shell
helm dependency build <chart-directory>
```

### 3. Log in

Log in with the robot account:

```shell
docker login registry.cloudogu.com
```

`cldk` and the builder use this login. The password does not belong in the chart, the source code repository, or a shell script. Automated releases use the credential management of the CI system.

### 4. Check and publish the release

In the root directory of the Dogu repository, start with a dry run. It checks `cldk.yaml`, the chart, the release tags, and the targets and shows what would be published:

```shell
cldk dogu release --dry-run
```

Then publish:

```shell
cldk dogu release
```

The DCC automatically imports the chart and reads the metadata from `Chart.yaml` and the other files in the chart. A `dogu.json` or a separate DCC request is not required.

All options, releasing an already packaged chart, and releases with images from other registries are described in [Releasing a dogu with `cldk dogu release`](https://github.com/cloudogu/cldk/blob/develop/docs/operations/releasing_a_dogu_en.md).

### 5. Tag the release commit

For a public source code repository, it is recommended to tag the release commit:

```shell
git tag -a "release/chart/<version>" -m "<version>

chart-version: <version>
app-version: <app-version>

this is a Dogu API v3 release"

git push origin "release/chart/<version>"
```

## Check the result

```shell
helm show chart \
  "oci://registry.cloudogu.com/<namespace>/dogu/v3/charts/<dogu>" \
  --version "<version>"
```

The output shows the `Chart.yaml` of the published chart with the expected `version`.

## Common errors

| Symptom | Cause and solution |
| --- | --- |
| `401 Unauthorized` | Username or password are wrong. Log in again with `docker login registry.cloudogu.com`. |
| `403 Forbidden` | The account may not push to the selected namespace. Check `namespace` in `cldk.yaml`. |

More error patterns are described in the [troubleshooting of `cldk dogu release`](https://github.com/cloudogu/cldk/blob/develop/docs/operations/releasing_a_dogu_en.md#troubleshooting).

## Further information

- [Understand Dogu V3 artifacts](../concepts/artifacts_en.md)
- [`cldk` README](https://github.com/cloudogu/cldk/blob/develop/README.md#dogu-release)
