# Dogu Artifacts

A Dogu is a ready-to-run application within the Cloudogu EcoSystem (CES) – such as SCM-Manager, Jenkins or SonarQube – that is seamlessly integrated into the platform.

With the generational change starting from version 3 (V3), the CES takes a step toward cloud-native standards: the Helm chart becomes the central release artifact and the primary format for packaging, versioning and deploying a Dogu. By deploying through Helm, an additional, platform-specific descriptor format (such as the previous `dogu.json`) becomes obsolete. This simplifies artifact management and embeds the application seamlessly into the Kubernetes ecosystem. Container images remain separate artifacts that the Helm chart references.

## The Chart as the Dogu Descriptor

Within the Helm chart, the `Chart.yaml` serves as the leading manifest for providing central metadata. The file provides both the identification of the chart and its package dependencies. In addition, there are further manifests that play an important role in the integration into the CES. The following sections describe the most important artifacts and their significance for a Dogu V3 in detail. The top-level files in the Helm chart include:

```
k8s/
└─ helm/
   ├─ templates/
   │  └─ ...
   ├─ Chart.yaml
   ├─ chart-patch-tpl.yaml
   ├─ dogu-upgrade.yaml
   ├─ dogu-values-metadata.yaml
   ├─ values.yaml
   └─ values.schema.json
```

These artifacts are described in order of relevance.

### `Chart.yaml`

The `Chart.yaml` contains metadata that describe a Dogu. Two versions are distinguished within the chart.
The Dogu version corresponds to the version of the chart (`version`), while the app version (`appVersion`) describes the version of the actual application contained within the Dogu. Both versions are independent of each other and may differ. For example, Redmine could be published as a Dogu in version `45.7.0`, while the contained Redmine application has the app version `6.1.2`.

In addition to the [Helm standard fields](https://helm.sh/docs/topics/charts/#the-chartyaml-file), the `Chart.yaml` uses platform-specific annotations that are marked with the prefix `dogu.cloudogu.com/`. This prefix identifies Dogu metadata within the Helm chart and distinguishes it from general Kubernetes or Helm annotations. The following platform-specific annotations are currently supported:

- `api-version`: The Dogu API version, e.g. `v3`
- `display-name`: The display name of a Dogu if the `name` attribute of the `Chart.yaml` should not be used for display
- `application.<name>`: Defines additional applications contained in the chart. `<name>` corresponds to the technical name of the contained application, whereby the annotation value contains a non-empty version specification as a string

Unknown annotations with the prefix `dogu.cloudogu.com/` are permitted but are ignored by platform-specific components, as long they are not part of the defined mandatory metadata. The following fields are considered mandatory in the `Chart.yaml`:

- `name`
- `version`
- `appVersion`
- `description`
- `annotations.dogu.cloudogu.com/api-version`

An example of a valid `Chart.yaml` for a Dogu could look as follows:

```yaml
apiVersion: v2
name: redmine
description: Redmine als Cloudogu EcoSystem Dogu
type: application
version: 45.7.0
appVersion: 6.1.2
home: https://cloudogu.com/ecosystem
sources:
  - https://github.com/cloudogu/redmine
icon: https://dogu.cloudogu.com/api/v3/dogus/official/redmine/icon.svg
keywords:
  - projectmanagement
  - issue
  - development
maintainers:
  - name: Cloudogu GmbH
    email: hello@cloudogu.com
annotations:
  dogu.cloudogu.com/api-version: v3
  dogu.cloudogu.com/display-name: Redmine
  dogu.cloudogu.com/application.redmine: "6.1.2"
  dogu.cloudogu.com/application.postgresql: "16.8"
```

### `values.yaml`

The `values.yaml` serves as the public interface for configuring Helm charts. It contains default values that are used for the Helm chart's templates. During installation, Helm allows the values from the `values.yaml` to be overridden by merging the supplied values with the default values from the Helm chart.

### `values.schema.json`

For the `values.yaml`, a schema can be defined in the form of a JSON schema using the `values.schema.json` to validate the resulting values of the Helm chart. Since the developers of an application know best what input is required and its correct structure, this schema file should always be an integral part of the Dogu Helm chart. The Dogu operator uses it to perform automated validation of the values during installation or update and to catch faulty configurations early.

### `dogu-values-metadata.yaml`

For the platform there can be global configuration parameters that cannot always be represented by existing values from a Helm chart's `values.yaml`. For example, a global log level can be set for the platform that is applied to all Dogus. To account this, the `dogu-values-metadata.yaml` was introduced. Through it, platform-specific configurations can be applied to the values of a Dogu's Helm chart by defining the mapping between individual values within it. Platform-specific configurations are defined in the Dogu CR and applied by the Dogu operator. The following example shows the mapping of the log level onto two applications of a Helm chart:

```yaml
apiVersion: v1
metavalues:
  # Platform-specific configuration value
  mainLogLevel:
    keys:
      # without a mapping the configuration value is passed through to the values.yaml unchanged
      - path: controllerManager.env.loglevel
      # with a mapping the configuration value is first translated (e.g. panic -> error) and then passed on
      # path to the value in the values.yaml for the main application
      - path: app.env.loglevel
        mapping:
          DEBUG: debug
          INFO: info
          WARN: error
          ERROR: error
      # path to the value in the values.yaml for the second application
      - path: loglevel
        mapping:
          DEBUG: 1
          INFO: 2
          WARN: 3
          ERROR: 4      
```

**Dogu CR that sets the configuration value:**

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: Dogu
metadata:
  name: my-dogu
spec:
  name: official/my-dogu
  version: 1.2.3-4
  mappedValues:
    mainLogLevel: ERROR
```

To be able to make platform-specific configurations, a Dogu Helm chart should always ship the `dogu-values-metadata.yaml`. For Dogus without configuration this file is optional.

### `dogu-upgrade.yaml`

An essential part of the CES is the execution of Dogu upgrades, which under normal circumstances require manual actions by the administrator. In the CES, upgrade processes are to be executed automatically by the Dogu operator. An important part for this are migration paths between individual Dogu versions. These are represented in the `dogu-upgrade.yaml` and validated by the Dogu operator during an upgrade. In the `dogu-upgrade.yaml`, the fundamentally permitted version jumps are defined:

```yaml
upgrades:
  # Upgrades within the path are allowed without migration.
  - from: ">=1.0.0 <=1.7.0" 
    to: "1.8.0"
  # Upgrade includes a migration.
  - from: ">=1.8.0 <2.0.0"
    to: "2.0.0"
    isMigration: true
    helmTimeout: 15m
    scaleSelectors: 
    - matchLabels:
        dogu.name: nexus
```

### `chart-patch-tpl.yaml`

A particular challenge for operating the platform are so-called air-gapped environments, which are run in isolation from other environments – in particular the internet. For such environments, both the required Helm charts and container images are mirrored into an internal OCI registry within the isolated environment, which changes the sources from which the images are obtained in the Helm chart. With the help of the `chart-patch-tpl.yaml`, the image references can be resolved and overwritten for the target environment by a mirroring tool.

**`chart-patch-tpl.yaml` using the Nexus Dogu as an example**

```yaml
apiVersion: v1
values:
  images:
    nexus: registry.cloudogu.com/official/nexus:3.86.2-6
    nexusSaManager: registry.cloudogu.com/k8s/service-account-producer-sidecar:0.1.2
    postgresql: docker.io/library/postgres:14.18
patches:
  values.yaml:
    nexus:
      image:
        registry: "{{ registryFrom .images.nexus }}"
        repository: "{{ repositoryFrom .images.nexus }}"
        tag: "{{ tagFrom .images.nexus }}"
      saManager:
        image:
          registry: "{{ registryFrom .images.nexusSaManager }}"
          repository: "{{ repositoryFrom .images.nexusSaManager }}"
          tag: "{{ tagFrom .images.nexusSaManager }}"
    postgresql:
      image:
        registry: "{{ registryFrom .images.postgresql }}"
        repository: "{{ repositoryFrom .images.postgresql }}"
        tag: "{{ tagFrom .images.postgresql }}"
```

The `chart-patch-tpl.yaml` should always be part of the Dogu Helm chart in order to be able to mirror the container images referenced in the Helm chart into an air-gapped environment.


## Templates and the Connection to the Platform

The Helm chart's templates provide all Kubernetes resources required to make the application runnable on the platform. These resources also include Custom Resource Definitions (CRDs) that are provided by Cloudogu and serve as the API for the platform. This section describes the individual CRDs for the CES integration in more detail.

### AuthRegistration

The CES offers its users the option of Single Sign-On (SSO). For a Dogu to be able to use SSO, it must register at the platform's identity provider (IdP) using the [`AuthRegistration-CR`](https://github.com/cloudogu/k8s-auth-registration-lib/blob/main/docs/operations/auth_registration_en.md). Currently the authentication protocols `CAS`, `OAUTH` and `OIDC` are supported. Registration with the IdP simultaneously provides credentials in a Secret that can be used by the Dogu for requests against the IdP.

An `AuthRegistration` only provisions the server-side integration at the IdP. The application must still implement the selected authentication protocol itself.

### WarpMenuEntry

The Warp menu is used for central navigation on the platform. If a Dogu should be shown in the Warp menu, a [`WarpMenuEntry-CR`](https://github.com/cloudogu/k8s-warp-menu-entry-lib/blob/main/docs/operations/warp_menu_entry_en.md) must be provided for each internal entry point that is visible to users. Depending on the need, a Dogu can define no entries, one entry, or multiple entries.

Each entry contains a German and an English display name, a category, and a relative path to the Dogu. The Warp menu does not make the application reachable by itself. The relative path must match an independently functioning HTTP exposition.

### Exposition

If a Dogu is supposed to be reachable from the outside, one or more [`Exposition-CR`](https://github.com/cloudogu/k8s-exposition-lib/blob/main/docs/operations/exposition_cr_en.md) must be declared for this purpose. The `Exposition` defines how a service is made reachable from outside the platform. It supports HTTP routes (Layer 7) as well as raw TCP and UDP ports (Layer 4). For HTTP routes, path rewrites can additionally be defined. At the Layer 4 level, collisions can occur if two or more `Expositions` request the same port for the same protocol. In this case, none of the affected `Expositions` are applied as long as the conflict persists. The administrator is informed about this in the status of the CR.

### ServiceAccountRequest / ServiceAccountProducer

In the CES it is possible for different Dogus/Components to interact with each other. For this, an appropriate ServiceAccount must be created at the target Dogu/Component (Producer) for an accessing Dogu/Component (Consumer). This service-account relationship can therefore be freely chosen and is not limited to Dogus only. This mechanism also allows cyclical dependencies between producers and consumers.

ServiceAccount creation can be controlled declaratively via the CRDs [`ServiceAccountRequest`](https://github.com/cloudogu/k8s-serviceaccount-lib/blob/main/docs/operations/serviceaccountrequest_cr_en.md) and [`ServiceAccountProducer`](https://github.com/cloudogu/k8s-serviceaccount-lib/blob/main/docs/operations/serviceaccountproducer_cr_en.md).

**`ServiceAccountProducer`**

If a Producer offers an interface that can be used by other Dogus, it must provide a `ServiceAccountProducer-CR`. The `ServiceAccountProducer`-CR defines how service accounts are created for the Consumer and which parameters are supported. Furthermore, the CR describes values that the Producer returns after creating a service account. Each returned value is written as a key into the Secret referenced by the requesting the Consumer.

**`ServiceAccountRequest`**

If a Consumer needs a service account for a Producer, it must request this via the `ServiceAccountRequest-CR` by naming a producer and optionally passing parameters. The resulting credentials of the request are written by the operator into a referenced Kubernetes Secret of the Consumer. If no reference to a Secret is declared in the `ServiceAccountRequest`, a Secret with the name of the ServiceAccountRequest resource is created. The CR is the counterpart to the `ServiceAccountProducer`.

## Distinction from Dogu V2

In Dogu V2, a Dogu is described using the platform-specific descriptor format `dogu.json`, in which both the configuration and the use of the platform API are described at the same time. In Dogu V3, a switch to [HELM](https://helm.sh/), the de-facto standard of Kubernetes for package management, takes place. As a result, much of the content of the previous `dogu.json` is moved either into Kubernetes resources or into separate Dogu-specific files within the Helm chart. The following table shows how the individual fields from the `dogu.json` are represented in the new Dogu Helm chart.

| Field in `dogu.json`                                               | Handling in Dogu V3                                                                     |
|--------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| `Name`, `Version`, `DisplayName`, `Description`, `URL`, `Logo`     | Metadata of the `Chart.yaml`                                                            |
| `Image`                                                            | Container image references in `chart-patch-tpl.yaml`                                    |
| `Dependencies`                                                     | Additional workloads or external dependencies via ServiceAccount CRs                    |
| `ServiceAccounts`                                                  | ServiceAccount CRs                                                                      |
| `Volumes`                                                          | Kubernetes PVCs and volume definitions                                                  |
| `ExposedCommands`                                                  | No longer exists as an API                                                              |
| `ExposedPorts`                                                     | Exposition CRs                                                                          |
| `Tags`, `Category`                                                 | Catalog metadata in `Chart.yaml`; Warp menu via WarpMenu CR                             |
| `Configuration`                                                    | `values.yaml`, `dogu-values-metadata.yaml` and `values.schema.json`                    |
| `HealthChecks`                                                     | Kubernetes probes in the pod specification                                              |
| `Capabilities`, `EnvironmentVariables`, `Properties`, `Privileged` | no longer exist or are represented as Kubernetes/Helm concepts                          |
