# Test backup & restore of a Dogu

This guide explains which resources of a Dogu V3 are backed up in the Multinode EcoSystem, how to prepare a Dogu for
this and how to test the backup.

## Prerequisites

You need:

- a Multinode EcoSystem with an installed
  [backup operator](https://github.com/cloudogu/k8s-backup-operator/blob/main/docs/operations/getting_started_en.md) and
  Velero,
- a storage provisioner that supports CSI volume snapshots,
  e.g. [Longhorn](https://github.com/cloudogu/k8s-backup-operator/blob/main/docs/operations/use_longhorn_storage_provisioner_en.md),
- a Dogu that was installed via a Dogu CR, and
- a Dogu that stores data in PersistentVolumeClaims.

A chart installed directly with `helm install`, as in the [quick start](../getting-started/quickstart_en.md), is not
managed by the Dogu operator and is not sufficient for this test. The local development cluster from
the [prerequisites](../getting-started/prerequisites_en.md) does not support backups either.

## What is backed up

A backup of the EcoSystem consists of several backup units. Each unit is a separate Velero backup:

| Backup unit     | Content                                                                |
|-----------------|------------------------------------------------------------------------|
| `global-config` | the ConfigMap `global-config`                                          |
| `dogu`          | the resources of a Dogu V3, one unit per Dogu                          |
| `doguv2`        | all Dogus V2 together                                                  |
| `components`    | resources of components with the label `k8s.cloudogu.com/backup-scope` |

If the backup of one Dogu fails, the other units remain usable.

### Resources of a Dogu V3

The unit of a Dogu V3 contains all resources of the following types with the label
`k8s.cloudogu.com/dogu.name: <Dogu name>`:

- the Dogu CR,
- PersistentVolumeClaims including the data of their volumes,
- ConfigMaps and
- Secrets.

Other resources such as Deployments, StatefulSets, Services or Jobs are not backed up. On a restore, the Dogu operator
recreates them from the Helm chart.

The chart does not have to set the label itself. The Dogu operator sets it on all resources the chart renders. PVCs from
the `volumeClaimTemplates` of a StatefulSet are not part of the chart. The Dogu operator maps them to the Dogu via the
StatefulSet and sets the label afterwards.

### Generated Secrets and ConfigMaps

ConfigMaps and Secrets are backed up although they are part of the chart. A chart can generate values during templating
that cannot be reproduced from the chart, e.g. a database password:

```gotemplate
{{- $existing := lookup "v1" "Secret" .Release.Namespace "my-dogu-db" }}
apiVersion: v1
kind: Secret
metadata:
name: my-dogu-db
type: Opaque
data:
{{- if $existing }}
password: {{ index $existing.data "password" }}
{{- else }}
password: {{ randAlphaNum 32 | b64enc }}
{{- end }}
```

Without a backup, a reinstallation would generate a new password that no longer matches the database in the restored
volume. On a restore, ConfigMaps and Secrets are restored before the Dogu CR. The Helm release adopts them during the
reinstallation because they still carry the Helm metadata of the release.

For generated values this means:

- Keep values with `lookup` if the resource already exists. Otherwise the chart overwrites the restored values.
- Generate values in the chart, not at runtime. A Secret that a container or Job creates via the Kubernetes API does not
  get the label from the Dogu operator and is only backed up if it carries the label `k8s.cloudogu.com/dogu.name`itself.

### What is not backed up

| Resource                                                                           | Behavior                                                                                                                                                |
|------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| `global-config`                                                                    | Not part of the Dogu unit. It is backed up in its own `global-config` unit. When only the Dogu is restored, it uses the current global-config.          |
| Secrets and ConfigMaps that do not come from the chart (e.g. via `existingSecret`) | Are not backed up. Creating and backing them up is up to the operator of the system. The label `k8s.cloudogu.com/backup-scope` has no effect for Dogus. |
| `ServiceAccountRequest` CRs                                                        | Are not backed up and not deleted on a restore, see [Service accounts](#service-accounts).                                                              |
| Secrets of service accounts                                                        | Are created by the service account operator, do not carry the Dogu label and are therefore not backed up.                                               |
| Data outside of PVCs (e.g. `emptyDir`, container file system)                      | Is lost on a restore.                                                                                                                                   |

### Service accounts

Service accounts of a Dogu V3 are requested via `ServiceAccountRequest` CRs in the chart
(see [ServiceAccountRequest / ServiceAccountProducer](../concepts/artifacts_en.md#serviceaccountrequest--serviceaccountproducer)).
Neither the CRs nor their Secrets are backed up. They are kept on a restore, so account and Secret still match.

If the producer is restored, e.g. a Dogu from an older backup, account and Secret no longer match. The service account
operator then rotates the credentials of all related accounts.

For Dogus this means:

- As a consumer: Always read credentials from the Secret and do not copy them into volumes. The Dogu must be able to
  handle a rotation, e.g. by a restart.
- As a producer: The producer must be able to set new credentials for an existing account and to recreate a missing
  account.

## Label `k8s.cloudogu.com/dogu.name` on the Dogu CR

Every Dogu CR must carry the label `k8s.cloudogu.com/dogu.name`. The Dogu operator checks this with a
ValidatingAdmissionPolicy when a Dogu CR is created or changed. If the label is missing or has a wrong value, the
Kubernetes API server rejects the Dogu CR.

The value of the label is the name of the Dogu without the Dogu namespace. For `spec.name: official/my-dogu` it is
`my-dogu`.

### With a blueprint

If the Dogu is installed via a [blueprint](../concepts/multinode-environment_en.md#blueprint-crd), the blueprint
operator sets the label automatically. Nothing else needs to be done.

### Without a blueprint

If you apply the Dogu CR yourself, you must set the label in the CR:

```yaml
apiVersion: k8s.cloudogu.com/v3beta1
kind: Dogu
metadata:
  name: my-dogu
  labels:
    k8s.cloudogu.com/dogu.name: my-dogu
spec:
  name: my-dogu
  doguNamespace: official
  version: 1.2.3-4
```

```shell
kubectl apply --namespace ecosystem -f my-dogu.yaml
```

### Label existing Dogu CRs

Dogu CRs that were created in the cluster without the label before the check was introduced can be labeled with the
following command:

```shell

NAMESPACE=ecosystem kubectl -n "${NAMESPACE}" get dogus.v3beta1.k8s.cloudogu.com -o jsonpath='{range .items[*]}{.spec.name}{"\n"}{end}' \
  | xargs -I {} kubectl -n "${NAMESPACE}" label dogu {} "k8s.cloudogu.com/dogu.name={}" --overwrite
```

Check the result:

```shell
kubectl get dogus --namespace ecosystem --label-columns k8s.cloudogu.com/dogu.name
```

Every Dogu CR must show the name of the Dogu in the `DOGU.NAME` column.

## Test the backup

The following steps use the Dogu `my-dogu` in the namespace `ecosystem`.

### 1. Check the backed-up resources

Before the backup, check which resources end up in the unit of the Dogu:

```shell
kubectl get dogus,persistentvolumeclaims,configmaps,secrets \
  --namespace ecosystem \
  --selector k8s.cloudogu.com/dogu.name=my-dogu
```

The list must contain:

- the Dogu CR,
- all PVCs with data of the Dogu, including those from `volumeClaimTemplates`, and
- all Secrets and ConfigMaps with generated values.

If a resource is missing, it is not backed up. Common causes are resources that are created at runtime instead of in the
chart, or PVCs whose StatefulSet is not part of the chart.

### 2. Create test data

Create data in the Dogu that can be clearly recognized after a restore, e.g. a project or an entry with a timestamp.
Also note the generated values:

```shell
kubectl get secret my-dogu-db --namespace ecosystem -o jsonpath='{.data.password}'
```

### 3. Create a backup

A backup without further settings backs up the whole EcoSystem. Use `excludedDogus` to exclude other Dogus V3 and
`options.ignoreV2Dogus` to exclude all Dogus V2. Dogus V2 cannot be excluded individually.

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: Backup
metadata:
  name: backup-my-dogu
spec:
  provider: velero
  excludedDogus:
    - other-dogu
  options:
    ignoreV2Dogus: true
```

```shell
kubectl apply --namespace ecosystem -f backup-my-dogu.yaml
```

An excluded Dogu must be installed as a Dogu V3, otherwise the backup fails.

### 4. Check the result

Wait for the backup to finish:

```shell
kubectl wait backups.k8s.cloudogu.com/backup-my-dogu \
  --namespace ecosystem --for=condition=Succeeded --timeout=60m
```

If the command times out, the backup as a whole failed or was canceled. The cause is shown in the conditions:

```shell
kubectl get backups.k8s.cloudogu.com backup-my-dogu --namespace ecosystem -o jsonpath='{.status.conditions}'
```

Then check the status of the Dogu's unit:

```shell
kubectl get backups.k8s.cloudogu.com backup-my-dogu --namespace ecosystem \
  -o jsonpath='{.status.dogus[?(@.name=="my-dogu")]}'
```

The status must contain `state: Succeeded` and a `providerRef`:

```json
{
  "name": "my-dogu",
  "version": "1.2.3-4",
  "state": "Succeeded",
  "providerRef": {
    "name": "backup-my-dogu-dogu-my-dogu"
  }
}
```

`Succeeded=True` means that all units have completed successfully.
If any units fail, you can also check the unit's `state`.
In this case, details are provided in `reason` and `message`.

### 5. Check the content of the Velero backup

The Velero backups of a backup carry the label `k8s.cloudogu.com/backup`:

```shell
kubectl get backups.velero.io --namespace ecosystem \
  --selector k8s.cloudogu.com/backup=backup-my-dogu \
  --label-columns k8s.cloudogu.com/backup-type,k8s.cloudogu.com/dogu.name
```

The [Velero CLI](https://velero.io/docs/main/basic-install/#install-the-cli) lists the backed-up resources and volume
snapshots:

```shell
velero backup describe backup-my-dogu-dogu-my-dogu --namespace ecosystem --details
```

`Resource List` must list all resources from step 1, and `CSI Snapshots` must contain one snapshot per PVC.
