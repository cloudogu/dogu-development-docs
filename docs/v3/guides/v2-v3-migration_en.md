# Prepare a V2 Dogu as a V3 chart that supports migration

A V3 Dogu is delivered as a Helm chart. The chart describes the application's resources and contains the jobs that transfer existing V2 data.
The Dogu operator orchestrates the migration and provides the jobs with the existing volumes and source credentials.

For Dogu developers, the main changes are:

- **Resources:** The contents of `dogu.json` are moved into chart files and Kubernetes resources.
- **Configuration:** Helm values become the new source of truth. The existing V2 Dogu configuration is not transferred automatically.
- **Data:** Migration jobs defined by the chart transfer persistent files and, where needed, data from previously shared services.

Each section first describes the requirements for your own Dogu, followed by an example from the [Nexus chart][nexus-chart].
The [Dogu operator's operations guide][operations] describes how administrators perform a migration.

## Contents

- [Prerequisites and limitations](#prerequisites-and-limitations): starting point, downtime, storage requirements and rollback
- [Prepare the V3 chart](#prepare-the-v3-chart): artifacts, workloads and CES integrations
- [Provide configuration](#provide-configuration): Helm values and how they differ from persistent data
- [Create migration jobs](#create-migration-jobs): flag, labels, source resources and Nexus examples
- [Check the chart](#check-the-chart): render templates for a fresh installation and a migration
- [Troubleshooting](#troubleshooting): conditions, events and common chart errors

## Prerequisites and limitations

### Prerequisites

- The existing V2 Dogu and its `dogu.json` are available.
- You are familiar with Helm chart structure and templates; see [From a Helm chart to a Dogu V3](../getting-started/quickstart_en.md).
- The Dogu's persistent data is known: files in its own volume and data in shared services such as PostgreSQL.

The subsequent migration requires a Dogu managed by the Dogu operator. A direct Helm installation does not perform it.

### Limitations

- **Downtime:** The application remains stopped while data is transferred.
- **Storage requirements:** Source and target volumes require storage at the same time during migration.
- **No automatic rollback to V2:** If migration fails, the application remains stopped. Any required recovery uses a backup.
- **One-time transition:** V3-to-V3 upgrades are a separate process. Migration support remains in the chart as long as V2 installations still need to be migrated.

**Example: Nexus**

Nexus needs both the files in its own volume and the data in the previously shared PostgreSQL instance.
Both data sets must be considered when preparing the chart.

## Prepare the V3 chart

The existing `dogu.json` is the starting point for the chart.
The complete [mapping of dogu.json fields to V3 artifacts](../concepts/artifacts_en.md#distinction-from-dogu-v2) describes where each part belongs.

### Chart files and resources

| File or resource | Purpose |
| --- | --- |
| `Chart.yaml` | Name, description and versions of the Dogu. The annotation `dogu.cloudogu.com/api-version: v3` identifies the V3 Dogu API. |
| Workload templates | Deployment or StatefulSet with containers, probes and resource requirements. |
| Service templates | Network access to workloads within the cluster. |
| PVCs and volume definitions | Persistent storage and how it is mounted in containers. |
| `mirror-patch-tpl.yaml` | Image references and their target keys in the values for mirroring to other registries. |

In `Chart.yaml`, `version` denotes the chart version and `appVersion` the application version.
Its `apiVersion: v2` denotes the Helm chart format; the Dogu API version is specified in the annotation.
The configuration files are described in the [next section](#provide-configuration).
For your own Dogu, decide which previously shared services will become part of the chart and which external integrations will remain.

**Example: Nexus**

Nexus uses a StatefulSet and bundles its own PostgreSQL instance.
The [Nexus Chart.yaml][nexus-chart] contains the metadata and V3 annotation; the templates define the workloads and their PVCs.

### CES integrations

The required integrations are declared as resources in the chart templates:

| Resource | Purpose |
| --- | --- |
| `AuthRegistration` | Registers the application with the identity provider. |
| `Exposition` | Makes a service accessible from outside the platform. |
| `WarpMenuEntry` | Adds an application entry point to the Warp menu. |
| `ServiceAccountRequest` | Requests a technical account from an external producer. |
| `ServiceAccountProducer` | Describes how the Dogu itself provides accounts for other applications. |

The documentation for these APIs is linked under [Templates and platform integration](../concepts/artifacts_en.md#templates-and-the-connection-to-the-platform).

**Example: Nexus**

Nexus registers its CAS integration through an `AuthRegistration`.
A `ServiceAccountProducer` describes the accounts Nexus provides to other applications.

### Service accounts still needed after migration

If the V3 Dogu needs an account from an external producer for normal operation, its chart must contain a `ServiceAccountRequest`.
The request is also deployed outside a migration. The operators handle the transfer of existing accounts.

**Example**

```yaml
apiVersion: k8s.cloudogu.com/v2
kind: ServiceAccountRequest
metadata:
  name: example-external-account
spec:
  consumer: example
  consumerType: Dogu
  producer: example-producer
  secretRef:
    name: example-external-credentials
```

The producer must exist. The producer defines which additional parameters are required.
The Dogu uses the credentials from `example-external-credentials`; see the [ServiceAccountRequest reference][sar] for details.

**Example: Nexus**

The Nexus chart contains no `ServiceAccountRequest`; its bundled database does not require an external account.
Temporary access to the old PostgreSQL instance uses the [migration Secret](#source-resources).
Nexus itself provides accounts through a `ServiceAccountProducer`, which represents the opposite side of a request.

## Provide configuration

The `values.yaml` is the new source of truth for the Dogu's configuration.
It contains defaults that can be overridden by Helm values supplied during installation.
The templates generate the application configuration from the merged values.

| File | Purpose |
| --- | --- |
| `values.yaml` | Configurable values and their defaults. |
| `values.schema.json` | Types, required values and allowed values for validation. |
| `dogu-values-metadata.yaml` | Mapping of platform-specific configuration values to chart values. |

The existing V2 Dogu configuration is not automatically transferred into these values.
The desired settings must be supplied through the new configuration interface.
Values, schema and their use in the workload must match.

**Example: Nexus**

For example, Nexus maps `logging/root` in its [values.yaml][nexus-values] as follows:

```yaml
nexus:
  configuration:
    normal:
      logging:
        root: "WARN"
```

The [ConfigMap template][nexus-config] renders `nexus.configuration.normal` as configuration for the application.

### Configuration and persistent data

The Dogu configuration described in `dogu.json` is not migrated automatically. The desired settings must be supplied through Helm values.
Persistent configuration in the Dogu PVC is transferred along with the other persistent data.

| Area | Handling |
| --- | --- |
| Dogu configuration from `dogu.json` | Not migrated; supply the desired settings through Helm values. |
| Persistent configuration and data in the Dogu PVC | Transfer through the migration PVC into the target PVCs. |
| Data in a shared service | Transfer into the target instance through a dedicated job. |

For your own Dogu, identify the required files and their target directories.
Data in shared services must also be considered; it resides outside the Dogu's own PV.

**Example: Nexus**

Nexus stores persistent local configuration in `localConfig`, including whether initial configuration has already completed.
The PVC migration job transfers this directory along with `data`.
It does not translate the Dogu configuration described in `dogu.json` into Helm values.
The log level `logging/root` is supplied through values. A separate job transfers the database from the previous PostgreSQL Dogu.

## Create migration jobs

The chart defines the required jobs as regular resources under `templates/`.
The operator enables them during migration and waits for them to complete.
The jobs are not implemented as Helm hooks.
The number and tasks of the jobs depend on the data to be transferred.
Even a Dogu without a database may need a job to transfer files.

**Example: Nexus**

Nexus needs two jobs:

| Job | Purpose |
| --- | --- |
| [PVC migration job][nexus-pvc] | Copies `data` and `localConfig` from the existing volume into the target PVC. |
| [PostgreSQL migration job][nexus-db-job] | Transfers the database from the shared PostgreSQL instance into the bundled instance. |

The complete templates are linked in the table.

### Activation

The following flag must be present in `values.yaml`:

```yaml
migrationJob:
  enabled: false
```

For a fresh installation it remains `false`.
For migration the operator sets it to `true`; after successful completion it resets it to `false`.
The job templates are rendered only when the flag is enabled.
The chart itself defines images, commands, mounts and resource requirements. No additional generic `migration` schema is required for this.

**Example: Nexus**

The Nexus values contain `migrationJob.enabled: false`.
Both the PVC job and the database job are wrapped in `{{- if .Values.migrationJob.enabled }}` in their respective templates.

### Labels

The chart labels the jobs for migration detection and their pods for access to shared source services:

| Label | Placement | Purpose |
| --- | --- | --- |
| `k8s.cloudogu.com/migration-job: "true"` | The job's `metadata.labels` | Identifies a migration job. Setting it only on the pod is not sufficient for job detection. |
| `k8s.cloudogu.com/dogu.name: <dogu>` | Added to chart resources by the operator; also set in the pod template by the chart | Associates the job and pods with the Dogu. |
| `k8s.cloudogu.com/migration-<name>: "true"` | The job's `spec.template.metadata.labels` | Together with the Dogu label, allows access to the shared source service through the temporary NetworkPolicy. |

`<dogu>` is the Dogu's name without its Dogu namespace.
`<name>` in `k8s.cloudogu.com/migration-<name>` identifies the **source service**.
This label belongs only on pods that need access to that service.

The operator detects jobs using both the migration label and the Dogu label.
Completed jobs must remain observable until the migration finishes. A TTL or other early cleanup must not remove them beforehand.

**Example: Nexus**

The [Nexus database job][nexus-db-job] needs access to the shared PostgreSQL service.
Its pods therefore carry `migration-postgresql` and the Dogu label with the value `nexus`:

```yaml
{{- if .Values.migrationJob.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "nexus.fullname" . }}-postgres-migration
  labels:
    k8s.cloudogu.com/migration-job: "true"
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        k8s.cloudogu.com/migration-postgresql: "true"
        k8s.cloudogu.com/dogu.name: nexus
    # See the complete template for the remaining pod settings.
{{- end }}
```

The Nexus PVC copy job does not need a PostgreSQL access label.

### Source resources

The operator provides the existing data and credentials in the Dogu's namespace:

| Resource | Usage |
| --- | --- |
| PVC `<dogu>-migration` | Source for transferring persistent files. |
| Secret `<dogu>-migration` | Source credentials for previously shared services. |

The chart references these resources and creates its own target PVCs and target credentials.
The operator creates the source resources and removes them after successful migration.
Secret and PVC can have the same name because they are different resource kinds.

**Example: Nexus**

The PVC `nexus-migration` contains the Dogu's existing files.
The Secret of the same name contains, among other things, the credentials for the previous PostgreSQL database.

### Transfer files

The copy job mounts the migration PVC as its source and one or more PVCs belonging to the chart as its targets.
Target PVCs, mounts and directories depend on the Dogu's data layout.
The application must later use exactly the PVCs into which the job wrote the data.
These must remain available after migration is disabled.

The name `<dogu>-migration` is the fixed interface to the source and is not derived from an arbitrary, different Helm fullname.

**Example: Nexus**

The Nexus copy job mounts the source PVC at `/source` and the target PVC at `/target`.
It copies the directories `data` and `localConfig` into the target volume.

The target name is defined in the [template][nexus-pvc] to match the PVC later used by the StatefulSet:

```gotemplate
{{- $pvcName := printf "%s-%s-0" "nexus-data" (include "nexus.fullname" .) }}
```

The job pod's volumes reference source and target:

```yaml
volumes:
  - name: source-data
    persistentVolumeClaim:
      claimName: nexus-migration
  - name: target-data
    persistentVolumeClaim:
      claimName: {{ $pvcName }}
```

The target PVC is named `nexus-data-<StatefulSet-Name>-0`.
The template creates it beforehand with `helm.sh/resource-policy: keep`, so it is retained when the migration template is disabled.
The Nexus StatefulSet then uses this PVC for its application data.

### Transfer data from shared services

The migration job accesses the previous service using credentials from `<dogu>-migration`.
The source address and Secret keys must match the requirements of that service.
The chart itself defines the target instance and its credentials.
The Dogu implements the transfer of the required data into this target instance.

**Example: Nexus**

The Nexus database job reads its source credentials from `nexus-migration`.
The keys for PostgreSQL are `postgresql_database`, `postgresql_username` and `postgresql_password`:

```yaml
env:
  - name: SOURCE_POSTGRESQL_DATABASE
    valueFrom:
      secretKeyRef:
        name: nexus-migration
        key: postgresql_database
  - name: SOURCE_POSTGRESQL_USERNAME
    valueFrom:
      secretKeyRef:
        name: nexus-migration
        key: postgresql_username
  - name: SOURCE_POSTGRESQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: nexus-migration
        key: postgresql_password
```

The source is `postgresql.<Namespace>.svc.cluster.local`; the target is the bundled PostgreSQL instance.
The [complete job template][nexus-db-job] and the [Nexus migration script][nexus-db-script] show the existing implementation.

### Stop the application during migration

The application must not start until its data has been transferred.
Its workload must therefore be set to zero replicas while `migrationJob.enabled` is enabled.
Target services required for the transfer must remain available during this time.

**Example: Nexus**

The [Nexus StatefulSet][nexus-statefulset] sets its replicas to zero during migration:

```yaml
replicas: {{ if .Values.migrationJob.enabled }} 0 {{ else }} {{ .Values.nexus.replicas | default 1 }} {{ end }}
```

The bundled PostgreSQL instance remains available so the job can import data.

### Job requirements

| Requirement | Meaning |
| --- | --- |
| Idempotence | Retrying after an interruption must not create inconsistent or duplicate data. A retry limit alone does not ensure this. |
| Version compatibility | Source version, migration tools and target version must be compatible. |
| Completeness | All required files and data from shared services must arrive at the target used later. |
| Observable completion | A successful job exits successfully; errors are reported as failures. |
| Stable resource names | Job names, target PVCs and references remain the same when rendered again. Random values or version numbers must not change them. |

For more information on names and references, see [Take care when renaming Kubernetes resources](../best-practices/helm-chart_en.md#take-care-when-renaming-kubernetes-resources).

**Example: Nexus**

Nexus uses the stable job names `<fullname>-pvc-migration` and `<fullname>-postgres-migration`.
For its database migration, the source PostgreSQL, the export/import tools and the bundled target version must be compatible.

## Check the chart

Render the chart with migration disabled and enabled.
Check the results for the following properties:

| Check | Expected result |
| --- | --- |
| `migrationJob.enabled=false` | No migration jobs; the application workload uses its normal replica count. |
| `migrationJob.enabled=true` | All required jobs are rendered; the application stays at zero replicas and the required target services are available. |
| Labels | The migration label is on the job; service-specific access labels and the Dogu label are on the pod template. |
| Source references | PVC and Secret names follow `<dogu>-migration`; the Secret keys match the source service. |
| Target references | The jobs write into the same PVCs and directories that the application workload uses later. |

Local rendering does not perform a migration or add labels supplied by the operator.
The managed job resources must later carry both the migration label and the Dogu label.

**Example: Nexus**

In the Nexus repository, the chart is located under `k8s/helm`:

```shell
helm template nexus ./k8s/helm --set migrationJob.enabled=false
helm template nexus ./k8s/helm --set migrationJob.enabled=true
```

With the flag enabled, the PVC and PostgreSQL migration jobs are rendered.
The Nexus StatefulSet then has zero replicas, while the target PostgreSQL remains available.

## Troubleshooting

The migration state is shown in the `Ready` condition of the V3 Dogu resource.
The reason identifies the state, the message describes the current step or error. Events show the history.

| Ready | Reason | Meaning |
| --- | --- | --- |
| `False` | `MigrationPending` | Migration is running or waiting. The message describes the current progress. |
| `False` | `MigrationFailed` | Migration has failed. The message and Warning events identify the cause. |
| `True` | Regular success reason | Migration is complete. The Dogu continues to operate as a normal V3 Dogu. |

`MigrationPending` and `MigrationFailed` are reasons of `Ready`, not separate condition types.

**Example: Nexus**

The following commands read the state of the Nexus migration in the `ecosystem` namespace:

```shell
# Ready condition
kubectl -n ecosystem get dogus.v3beta1.k8s.cloudogu.com nexus \
  -o jsonpath='{range .status.conditions[?(@.type=="Ready")]}{.status}: {.reason}: {.message}{"\n"}{end}'

# Events
kubectl -n ecosystem get events \
  --field-selector involvedObject.kind=Dogu,involvedObject.name=nexus
```

### Common errors

| Symptom | What to check in the chart |
| --- | --- |
| No job is detected within the timeout; `MigrationFailed`. | Is the job rendered when the flag is enabled? Is the migration label on the job and the Dogu label on the managed resource? |
| Job pod does not start because a Secret or Secret key is missing. | Check `<dogu>-migration` and the expected keys. If the source resource itself is missing, check the corresponding events on the Dogu resource. |
| Database job cannot reach the shared source service. | Check the source address, `migration-<name>` and the Dogu label on the pod; also consult pod events and logs. |
| Copy job waits for a PVC or the application later cannot find the transferred files. | Compare claim names, mounts and directories with the application workload that uses them later. |
| Job fails permanently; `MigrationFailed`. | Check job logs for copy or import errors, permissions, storage requirements and version compatibility. |

The [operations guide][operations] describes how to resume and what to do after a failed migration.

**Example: Nexus**

If the database job cannot reach the previous PostgreSQL instance, first check the source address and pod labels.
The pod must carry `k8s.cloudogu.com/migration-postgresql: "true"` and `k8s.cloudogu.com/dogu.name: nexus`.
The Dogu resource's message and the job logs help narrow down the error.

## References

[nexus-chart]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/Chart.yaml
[nexus-values]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/values.yaml
[nexus-config]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/config.yaml
[nexus-pvc]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/pvc-migration.yaml
[nexus-db-job]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/postgres-migration-job.yaml
[nexus-db-script]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/files/postgres-migration.sh
[nexus-statefulset]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/statefulset.yaml
[operations]: https://github.com/cloudogu/k8s-dogu-operator/blob/a38702ddd26d4c19681915e2ff8858330a852fa6/docs/operations/v3/migrating_from_v2_to_v3_en.md
[sar]: https://github.com/cloudogu/k8s-serviceaccount-lib/blob/develop/docs/operations/serviceaccountrequest_cr_en.md
