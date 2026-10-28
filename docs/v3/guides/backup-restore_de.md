# Backup & Restore eines Dogus testen

Diese Anleitung erklärt, wie ein Dogu im Multinode EcoSystem für Backup & Restore vorbereitet und anschließend getestet wird.

## Voraussetzungen

Benötigt werden:

- ein Multinode EcoSystem mit installiertem [Backup-Operator](https://github.com/cloudogu/k8s-backup-operator/blob/main/docs/operations/getting_started_de.md),
- ein Dogu, das über eine Dogu-CR installiert wurde, und
- ein Dogu, das Daten in PersistentVolumeClaims ablegt.

Ein direkt mit `helm install` installiertes Chart, wie im [Quickstart](../getting-started/quickstart_de.md), wird nicht vom Dogu-Operator verwaltet und reicht für diesen Test nicht aus.

## Label `k8s.cloudogu.com/dogu.name` an der Dogu-CR

Jede Dogu-CR muss das Label `k8s.cloudogu.com/dogu.name` tragen. Der Dogu-Operator prüft dies beim Anlegen und Ändern einer Dogu-CR mit einer ValidatingAdmissionPolicy. Fehlt das Label oder hat es einen falschen Wert, lehnt der Kubernetes-API-Server die Dogu-CR ab.

Der Wert des Labels ist der Name des Dogus ohne Dogu-Namespace. Für `spec.name: official/my-dogu` lautet er also `my-dogu`.

### Mit Blueprint

Wird das Dogu über einen [Blueprint](../concepts/multinode-environment_de.md#blueprint-crd) installiert, setzt der Blueprint-Operator das Label automatisch. Es ist nichts weiter zu tun.

### Ohne Blueprint

Wird die Dogu-CR selbst angewendet, muss das Label in der CR gesetzt werden:

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

### Bestehende Dogu-CRs nachträglich labeln

Dogu-CRs, die bereits vor Einführung der Prüfung ohne Label im Cluster angelegt wurden, können mit folgendem Befehl gelabelt werden:

```shell
NAMESPACE=ecosystem kubectl -n "${NAMESPACE}" get dogus.v3beta1.k8s.cloudogu.com -o jsonpath='{range .items[*]}{.spec.name}{"\n"}{end}' \
  | xargs -I {} kubectl -n "${NAMESPACE}" label dogu {} "k8s.cloudogu.com/dogu.name={}" --overwrite
```

Das Ergebnis prüfen:

```shell
kubectl get dogus --namespace ecosystem --label-columns k8s.cloudogu.com/dogu.name
```

Jede Dogu-CR muss in der Spalte `DOGU.NAME` den Namen des Dogus anzeigen.