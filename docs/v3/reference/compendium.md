# Dogu V3 Artifact Compendium

This compendium provides an overview of the artifacts and APIs of Dogu V3. For an introduction to how they work together, read [Understand Dogu V3 Artifacts](../concepts/artifacts.md).

## Chart artifacts and APIs

| Artifact / API | Purpose | When needed | Owner | Reference documentation |
| --- | --- | --- | --- | --- |
| Helm chart | Leading Dogu package; contains metadata, defaults, templates and companion files | Always | Dogu developer | [Helm chart format](https://helm.sh/docs/topics/charts/) |
| `Chart.yaml` | Dogu identity and discovery metadata; `version` is the Dogu version, `appVersion` the packaged application version | Always | Dogu developer; consumed by Dogu tooling | [Helm `Chart.yaml`](https://helm.sh/docs/topics/charts/#the-chartyaml-file) |
| `values.yaml` | Safe defaults and public chart configuration interface | Always, according to the Helm chart convention | Dogu developer | [Helm values](https://helm.sh/docs/chart_template_guide/values_files/) |
| `values.schema.json` | Validates the merged values | When the Dogu exposes configurable values | Dogu developer | [Helm schema files](https://helm.sh/docs/topics/charts/#schema-files) |
| `dogu-values-metadata.yaml` | Maps CES configuration keys, such as a global log level, to chart values | For CES-wide configuration mappings; optional for a configuration-free Dogu | Dogu developer; consumed by Dogu Operator | — |
| `templates/` | Renders workloads, Services, PVCs, probes and optional CES integration resources | Always | Dogu developer | [Helm templates](https://helm.sh/docs/chart_template_guide/) and owning Kubernetes/CRD contracts |
| `chart-patch-tpl.yaml` | Lists or resolves every container image referenced by the chart for the Dogu Registry and mirroring tools | Whenever the chart references container images, which is normally every Dogu | Dogu developer; consumed by the Dogu Registry and mirroring tools | — |
| `dogu-upgrade.yaml` | Describes valid Dogu-version transitions and optional parameters for upgrade coordination | The accepted ADRs do not yet define when the file must be present | Dogu developer; consumed by the Dogu Operator | — |
| Container images | Provide the application and any sidecar and init containers referenced by the chart | For every container rendered by the chart | Image producer; references owned by Dogu developer | [Kubernetes images](https://kubernetes.io/docs/concepts/containers/images/) |
| `AuthRegistration` | Declares authentication registration when the Dogu participates in CES authentication | When the Dogu uses CES authentication | k8s-auth-registration-lib owns the API; Dogu developer declares it | [AuthRegistration v1 API](https://github.com/cloudogu/k8s-auth-registration-lib/tree/develop/api/v1) |
| `Exposition` | Connects a chart Service to CES-provided external access when needed | When the Dogu needs external access | k8s-exposition-lib owns the API; Dogu developer declares it | [Exposition v1 API](https://github.com/cloudogu/k8s-exposition-lib/tree/develop/api/v1) |
| `ServiceAccountRequest` | Requests technical credentials from a producer when another CES service is needed | When the Dogu needs technical credentials from another CES service | k8s-serviceaccount-lib owns the API; consumer declares it | [Service account v2 API](https://github.com/cloudogu/k8s-serviceaccount-lib/tree/develop/api/v2) |
| `ServiceAccountProducer` | Declares that the Dogu offers technical accounts | When the Dogu provides technical accounts | k8s-serviceaccount-lib owns the API; producer declares it | [Service account v2 API](https://github.com/cloudogu/k8s-serviceaccount-lib/tree/develop/api/v2) |
| `WarpMenuEntry` | Declares a shared-menu entry when the Dogu has a user-facing CES path | When the Dogu provides a user-facing CES path | k8s-warp-menu-entry-lib owns the API; Dogu developer declares it | [WarpMenuEntry v1 API](https://github.com/cloudogu/k8s-warp-menu-entry-lib/tree/develop/api/v1) |
| Dogu Registry data | Combines technical chart metadata with entitlement and marketing or sales data maintained outside the chart | For publication, discovery and entitlement through the Dogu Registry | Cloudogu | — |

## Complete V2 `dogu.json` disposition

The following table lists every V2 `dogu.json` field and its accepted V3 destination where one exists. These mappings are not automatic conversions and do not necessarily preserve every V2 behavior. Where the accepted architecture defines no general V3 destination, the table states that explicitly.

| V2 field | V3 destination | Disposition and status |
| --- | --- | --- |
| `Name` | `Chart.yaml` + registry namespace | Chart `name` is the simple technical name; the qualified namespace comes from registry context. |
| `Version` | `Chart.yaml` | Becomes chart `version`, the Dogu version. `appVersion` separately identifies the application. |
| `PublishedAt` | Dogu Registry v3 API | Included as publication metadata in the accepted API design. Its source and authoring workflow are not yet defined. |
| `DisplayName` | `Chart.yaml` | Dogu annotation `dogu.cloudogu.com/display-name`; optional in accepted target metadata. |
| `Description` | `Chart.yaml` | Standard `description`; required in the accepted target metadata. |
| `Category` | No concrete V3 field defined | The accepted architecture treats it as catalogue metadata but defines neither a `Chart.yaml` field nor an external source. It is independent of `WarpMenuEntry`. |
| `Tags` | `Chart.yaml` | Use Helm `keywords` for general search terms. Define menu entries separately with `WarpMenuEntry`. |
| `Logo` | `Chart.yaml` | Store the URL of the Dogu logo in the Helm `icon` field. |
| `URL` | `Chart.yaml` | Store the project or original vendor website in the Helm `home` field. |
| `Image` | Companion file + Kubernetes workloads | Image references are defined in the container specs. `chart-patch-tpl.yaml` also makes them discoverable by platform tooling. |
| `ExposedPorts` | Helm/Kubernetes + CES CR | Kubernetes Service plus `Exposition` when external access is needed. |
| `ExposedCommands` | Removed / purpose-specific V3 mechanisms | There is no generic V3 `ExposedCommands` API. Implement upgrade migrations with init containers or Helm hook Jobs. Assess other lifecycle or custom commands separately. |
| `Volumes` | Helm/Kubernetes resources | PVCs, volumes and volume mounts. Backup and retention must be declared/documented separately; no automatic field conversion. |
| `HealthCheck` | Helm/Kubernetes resources | Deprecated V2 single check. Map TCP or HTTP checks to an appropriate Kubernetes probe where the signal is equivalent; the accepted architecture defines no direct V3 equivalent for the V2 `state` check. |
| `HealthChecks` | Helm/Kubernetes resources | Model applicable checks as startup, readiness or liveness probes according to their purpose. This does not automatically preserve every V2 check type or consumer behavior. |
| `ServiceAccounts` | CES CRs | `ServiceAccountRequest` and, for offered accounts, `ServiceAccountProducer`. |
| `Privileged` | Removed | There is no direct V3 equivalent. The V2 field mounted the Docker socket; Kubernetes privileged mode or a security context is not an equivalent replacement. Define only the pod and container permissions the application requires. |
| `Security` | Helm/Kubernetes resources | Map supported controls to pod or container security contexts and capabilities. The accepted architecture does not define a shared V3 security baseline. |
| `Configuration` | Chart values, schema and companion metadata | `values.yaml`, `values.schema.json` and, for CES mappings, `dogu-values-metadata.yaml`. |
| `Properties` | Removed | The generic V2 field has no general V3 replacement. Use a concrete Helm, Kubernetes or CES API only when that API defines the required behavior. |
| `EnvironmentVariables` | Helm/Kubernetes resources | Explicit container `env`/`envFrom`, normally sourced from values, ConfigMaps or Secrets as appropriate. |
| `Dependencies` | Helm/Kubernetes resources or CES service-account CRs | Model service dependencies with `ServiceAccountRequest` where appropriate and other requirements explicitly in the chart. The accepted architecture defines no generic replacement for all V2 client, package or version checks. |
| `OptionalDependencies` | Optional `ServiceAccountRequest` or purpose-specific chart resources | Use an optional request for optional service-account dependencies. The accepted architecture defines no generic contract that preserves every V2 optional-dependency and version-check behavior. |

## Related concept

Return to [Understand Dogu V3 Artifacts](../concepts/artifacts.md), or continue with [the Multinode runtime environment](../concepts/multinode-environment.md).
