# Backup & Restore eines Dogus testen

Diese Anleitung erklärt, welche Ressourcen eines Dogus V3 im Multinode EcoSystem gesichert werden, wie ein Dogu darauf
vorbereitet wird und wie das Backup getestet wird.

## Voraussetzungen

Benötigt werden:

- ein Multinode EcoSystem mit
  installiertem [Backup-Operator](https://github.com/cloudogu/k8s-backup-operator/blob/main/docs/operations/getting_started_de.md)
  und Velero,
- ein Storage-Provisioner mit Unterstützung für CSI-Volume-Snapshots, z.
  B. [Longhorn](https://github.com/cloudogu/k8s-backup-operator/blob/main/docs/operations/use_longhorn_storage_provisioner_de.md),
- ein Dogu, das über eine Dogu-CR installiert wurde, und
- ein Dogu, das Daten in PersistentVolumeClaims ablegt.

Ein direkt mit `helm install` installiertes Chart, wie im [Quickstart](../getting-started/quickstart_de.md), wird nicht
vom Dogu-Operator verwaltet und reicht für diesen Test nicht aus. Auch der lokale Entwicklungscluster aus
den [Voraussetzungen](../getting-started/prerequisites_de.md) unterstützt keine Backups.

## Was gesichert wird

Ein Backup des EcoSystems besteht aus mehreren Backup-Units. Jede Unit ist ein eigenes Velero-Backup:

| Backup-Unit     | Inhalt                                                                  |
|-----------------|-------------------------------------------------------------------------|
| `global-config` | die ConfigMap `global-config`                                           |
| `dogu`          | die Ressourcen eines Dogus V3, eine Unit je Dogu                        |
| `doguv2`        | alle V2-Dogus gemeinsam                                                 |
| `components`    | Ressourcen von Components mit dem Label `k8s.cloudogu.com/backup-scope` |

Schlägt die Sicherung eines Dogus fehl, bleiben die übrigen Units nutzbar.

### Ressourcen eines Dogus V3

Die Unit eines Dogus V3 enthält alle Ressourcen folgender Typen mit dem Label `k8s.cloudogu.com/dogu.name: <Dogu-Name>`:

- die Dogu-CR,
- PersistentVolumeClaims einschließlich der Daten ihrer Volumes,
- ConfigMaps und
- Secrets.

Andere Ressourcen wie Deployments, StatefulSets, Services oder Jobs werden nicht gesichert. Der Dogu-Operator erzeugt
sie bei einem Restore neu aus dem Helm-Chart.

Das Label muss im Chart nicht selbst gesetzt werden. Der Dogu-Operator setzt es an alle Ressourcen, die das Chart
rendert. PVCs aus den `volumeClaimTemplates` eines StatefulSets sind nicht Teil des Charts. Der Dogu-Operator ordnet sie
über das StatefulSet dem Dogu zu und setzt das Label nachträglich.

### Generierte Secrets und ConfigMaps

ConfigMaps und Secrets werden gesichert, obwohl sie Teil des Charts sind. Ein Chart kann beim Templating Werte erzeugen,
die sich nicht aus dem Chart reproduzieren lassen, z. B. ein Datenbankpasswort:

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

Ohne Sicherung würde eine Neuinstallation ein neues Passwort erzeugen, das nicht mehr zur Datenbank im
wiederhergestellten Volume passt. Bei einem Restore werden ConfigMaps und Secrets vor der Dogu-CR wiederhergestellt. Das
Helm-Release übernimmt sie bei der Neuinstallation, weil sie weiterhin die Helm-Metadaten des Releases tragen.

Für generierte Werte gilt deshalb:

- Werte mit `lookup` beibehalten, wenn die Ressource bereits existiert. Andernfalls überschreibt das Chart die
  wiederhergestellten Werte.
- Werte im Chart erzeugen, nicht zur Laufzeit. Ein Secret, das ein Container oder Job über die Kubernetes-API anlegt,
  erhält das Label nicht vom Dogu-Operator und wird nur gesichert, wenn es das Label `k8s.cloudogu.com/dogu.name` selbst
  trägt.

### Was nicht gesichert wird

| Ressource                                                                             | Verhalten                                                                                                                                                  |
|---------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `global-config`                                                                       | Nicht Teil der Dogu-Unit. Sie wird in der eigenen Unit `global-config` gesichert. Wird ausschließlich das Dogu wiederhergestellt, verwendet es die aktuelle global-config. |
| Secrets und ConfigMaps, die nicht aus dem Chart stammen (z. B. über `existingSecret`) | Werden nicht gesichert. Ihre Erstellung und Sicherung liegt beim Betreiber. Das Label `k8s.cloudogu.com/backup-scope` ist für Dogus wirkungslos.           |
| `ServiceAccountRequest`-CRs                                                           | Werden nicht gesichert und bei einem Restore nicht gelöscht, siehe [Service-Accounts](#service-accounts).                                                  |
| Secrets von Service-Accounts                                                          | Werden vom Service-Account-Operator erstellt, tragen das Dogu-Label nicht und werden daher nicht gesichert.                                                |
| Daten außerhalb von PVCs (z. B. `emptyDir`, Container-Dateisystem)                    | Gehen bei einem Restore verloren.                                                                                                                          |

### Service-Accounts

Service-Accounts eines Dogus V3 werden über `ServiceAccountRequest`-CRs im Chart angefordert
(siehe [ServiceAccountRequest / ServiceAccountProducer](../concepts/artifacts_de.md#serviceaccountrequest--serviceaccountproducer)).
Weder die CRs noch ihre Secrets werden gesichert. Bei einem Restore bleiben sie erhalten, sodass Account und Secret
weiterhin zueinander passen.

Wird der Producer wiederhergestellt, z. B. ein Dogu aus einem älteren Backup, passen Account und Secret eventuell nicht mehr
zueinander. Der Service-Account-Operator rotiert dann die Credentials aller zugehörigen Accounts.

Daraus folgt für Dogus:

- Als Consumer: Credentials immer aus dem Secret lesen und nicht in Volumes kopieren. Eine Rotation muss das Dogu
  übernehmen können, z. B. durch einen Neustart.
- Als Producer: Der Producer muss für einen bestehenden Account neue Credentials setzen und einen fehlenden Account neu
  anlegen können.

## Label `k8s.cloudogu.com/dogu.name` an der Dogu-CR

Jede Dogu-CR muss das Label `k8s.cloudogu.com/dogu.name` tragen. Der Dogu-Operator prüft dies beim Anlegen und Ändern
einer Dogu-CR mit einer ValidatingAdmissionPolicy. Fehlt das Label oder hat es einen falschen Wert, lehnt der
Kubernetes-API-Server die Dogu-CR ab.

Der Wert des Labels ist der Name des Dogus ohne Dogu-Namespace. Für `spec.name: official/my-dogu` lautet er also
`my-dogu`.

### Mit Blueprint

Wird das Dogu über einen [Blueprint](../concepts/multinode-environment_de.md#blueprint-crd) installiert, setzt der
Blueprint-Operator das Label automatisch. Es ist nichts weiter zu tun.

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

Dogu-CRs, die bereits vor Einführung der Prüfung ohne Label im Cluster angelegt wurden, können mit folgendem Befehl
gelabelt werden:

```shell

NAMESPACE=ecosystem kubectl -n "${NAMESPACE}" get dogus.v3beta1.k8s.cloudogu.com -o jsonpath='{range .items[*]}{.spec.name}{"\n"}{end}' \
  | xargs -I {} kubectl -n "${NAMESPACE}" label dogu {} "k8s.cloudogu.com/dogu.name={}" --overwrite
```

Das Ergebnis prüfen:

```shell
kubectl get dogus --namespace ecosystem --label-columns k8s.cloudogu.com/dogu.name
```

Jede Dogu-CR muss in der Spalte `DOGU.NAME` den Namen des Dogus anzeigen.

## Backup testen

Die folgenden Schritte verwenden das Dogu `my-dogu` im Namespace `ecosystem`.

### 1. Gesicherte Ressourcen prüfen

Vor dem Backup prüfen, welche Ressourcen in der Unit des Dogus landen:

```shell
kubectl get dogus,persistentvolumeclaims,configmaps,secrets \
  --namespace ecosystem \
  --selector k8s.cloudogu.com/dogu.name=my-dogu
```

Die Liste muss enthalten:

- die Dogu-CR,
- alle PVCs mit Daten des Dogus, auch die aus `volumeClaimTemplates`, und
- alle Secrets und ConfigMaps mit generierten Werten.

Fehlt eine Ressource, wird sie nicht gesichert. Häufige Ursachen sind Ressourcen, die zur Laufzeit statt im Chart
angelegt werden, oder PVCs, deren StatefulSet nicht Teil des Charts ist.

### 2. Testdaten anlegen

Im Dogu Daten erzeugen, die nach einem Restore eindeutig wiedererkennbar sind, z. B. ein Projekt oder einen Eintrag mit
Zeitstempel. Zusätzlich die generierten Werte notieren:

```shell
kubectl get secret my-dogu-db --namespace ecosystem -o jsonpath='{.data.password}'
```

### 3. Backup erstellen

Ein Backup ohne weitere Angaben sichert das gesamte EcoSystem. Mit `excludedDogus` lassen sich andere Dogus V3
ausschließen, mit `options.ignoreV2Dogus` alle Dogus V2. Dogus V2 lassen sich nicht einzeln ausschließen.

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

Ein ausgeschlossenes Dogu muss als Dogu V3 installiert sein, sonst schlägt das Backup fehl.

### 4. Ergebnis prüfen

Auf das Ende des Backups warten:

```shell
kubectl wait backups.k8s.cloudogu.com/backup-my-dogu \
  --namespace ecosystem --for=condition=Succeeded --timeout=60m
```

Läuft der Befehl in den Timeout, ist das Backup als Ganzes fehlgeschlagen oder abgebrochen. Die Ursache steht in den
Conditions:

```shell
kubectl get backups.k8s.cloudogu.com backup-my-dogu --namespace ecosystem -o jsonpath='{.status.conditions}'
```

Anschließend den Status der Unit des Dogus prüfen:

```shell
kubectl get backups.k8s.cloudogu.com backup-my-dogu --namespace ecosystem \
  -o jsonpath='{.status.dogus[?(@.name=="my-dogu")]}'
```

Der Status muss `state: Succeeded` und eine `providerRef` enthalten:

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

`Succeeded=True` bedeutet nur, dass alle Units abgeschlossen sind. Ein Backup mit fehlgeschlagenen Units gilt trotzdem
als erfolgreich. Deshalb immer den `state` der Unit prüfen. Bei `Failed` oder `Missing` stehen Details in `reason` und
`message`.

### 5. Inhalt des Velero-Backups prüfen

Die Velero-Backups eines Backups tragen das Label `k8s.cloudogu.com/backup`:

```shell
kubectl get backups.velero.io --namespace ecosystem \
  --selector k8s.cloudogu.com/backup=backup-my-dogu \
  --label-columns k8s.cloudogu.com/backup-type,k8s.cloudogu.com/dogu.name
```

Mit der [Velero-CLI](https://velero.io/docs/main/basic-install/#install-the-cli) lassen sich die gesicherten Ressourcen
und Volume-Snapshots auflisten:

```shell
velero backup describe backup-my-dogu-dogu-my-dogu --namespace ecosystem --details
```

Unter `Resource List` müssen alle Ressourcen aus Schritt 1 aufgeführt sein, unter `CSI Snapshots` je PVC ein Snapshot.
