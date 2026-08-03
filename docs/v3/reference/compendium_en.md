# Dogu V3 Artifact Compendium

This compendium provides an overview of the artifacts and APIs of Dogu V3. For an introduction to how they work together, read [Dogu Artifacts](../concepts/artifacts_en.md).

## Chart artifacts and APIs

| Artifact / API | Purpose | Reference documentation |
| --- | --- | --- |
| Helm chart | Leading Dogu package; contains metadata, defaults, templates and companion files | [Helm chart format](https://helm.sh/docs/topics/charts/) |
| `Chart.yaml` | Dogu identity and discovery metadata; `version` is the Dogu version, `appVersion` the packaged application version | [Helm `Chart.yaml`](https://helm.sh/docs/topics/charts/#the-chartyaml-file) |
| `values.yaml` | Default values and public chart configuration interface | [Helm values](https://helm.sh/docs/chart_template_guide/values_files/) |
| `values.schema.json` | Validates the merged values | [Helm schema files](https://helm.sh/docs/topics/charts/#schema-files) |
| `dogu-values-metadata.yaml` | Maps CES configuration keys, such as a global log level, to chart values | [dogu-values-metadata.yaml](../concepts/artifacts_en.md#dogu-values-metadatayaml) |
| `templates/` | Renders workloads, Services, PVCs, probes and optional CES integration resources | [Helm templates](https://helm.sh/docs/chart_template_guide/) and owning Kubernetes/CRD contracts |
| `chart-patch-tpl.yaml` | Lists every container image referenced by the chart and resolves the references through a mirroring tool for air-gapped environments | [chart-patch-tpl.yaml](../concepts/artifacts_en.md#chart-patch-tplyaml) |
| `dogu-upgrade.yaml` | Describes valid Dogu-version transitions and optional parameters for upgrade coordination | [dogu-upgrade.yaml](../concepts/artifacts_en.md#dogu-upgradeyaml) |
| Container images | Provide the application and any sidecar and init containers referenced by the chart | [Kubernetes images](https://kubernetes.io/docs/concepts/containers/images/) |
| `AuthRegistration` | Declares authentication registration when the Dogu participates in CES authentication | [AuthRegistration](../concepts/artifacts_en.md#authregistration) |
| `Exposition` | Connects a chart Service to CES-provided external access when needed | [Exposition](../concepts/artifacts_en.md#exposition) |
| `ServiceAccountRequest` | Requests technical credentials from a producer when another CES service is needed | [ServiceAccountRequest](../concepts/artifacts_en.md#serviceaccountrequest--serviceaccountproducer) |
| `ServiceAccountProducer` | Declares that the Dogu offers technical accounts | [ServiceAccountProducer](../concepts/artifacts_en.md#serviceaccountrequest--serviceaccountproducer) |
| `WarpMenuEntry` | Declares a Warp menu entry when the Dogu has a user-facing CES path | [WarpMenuEntry](../concepts/artifacts_en.md#warpmenuentry) |
| Dogu Registry data | Combines technical chart metadata with entitlement and marketing or sales data maintained outside the chart | — |
