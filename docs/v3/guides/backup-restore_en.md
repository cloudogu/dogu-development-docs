# Test backup & restore of a Dogu

This guide explains how to prepare a Dogu in the Multinode EcoSystem for backup & restore and how to test it afterwards.

## Prerequisites

You need:

- a Multinode EcoSystem with an installed [backup operator](https://github.com/cloudogu/k8s-backup-operator/blob/main/docs/operations/getting_started_en.md),
- a Dogu that was installed via a Dogu CR, and
- a Dogu that stores data in PersistentVolumeClaims.

A chart installed directly with `helm install`, as in the [quick start](../getting-started/quickstart_en.md), is not managed by the Dogu operator and is not sufficient for this test.

## Label `k8s.cloudogu.com/dogu.name` on the Dogu CR

Every Dogu CR must carry the label `k8s.cloudogu.com/dogu.name`. The Dogu operator checks this with a ValidatingAdmissionPolicy when a Dogu CR is created or changed. If the label is missing or has a wrong value, the Kubernetes API server rejects the Dogu CR.

The value of the label is the name of the Dogu without the Dogu namespace. For `spec.name: official/my-dogu` it is `my-dogu`.

### With a blueprint

If the Dogu is installed via a [blueprint](../concepts/multinode-environment_en.md#blueprint-crd), the blueprint operator sets the label automatically. Nothing else needs to be done.

### Without a blueprint

If you apply the Dogu CR yourself, the label must be set in the CR:

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

Dogu CRs that were created without the label before the check was introduced can be labeled with the following command:

```shell
NAMESPACE=ecosystem kubectl -n "${NAMESPACE}" get dogus.v3beta1.k8s.cloudogu.com -o jsonpath='{range .items[*]}{.spec.name}{"\n"}{end}' \
  | xargs -I {} kubectl -n "${NAMESPACE}" label dogu {} "k8s.cloudogu.com/dogu.name={}" --overwrite
```

Check the result:

```shell
kubectl get dogus --namespace ecosystem --label-columns k8s.cloudogu.com/dogu.name
```

Every Dogu CR must show the name of the Dogu in the `DOGU.NAME` column.
