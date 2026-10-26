# Vom Helm-Chart zum Dogu V3

Diese Anleitung erstellt ein Dogu-V3-Chart für XWiki. Das vorhandene [XWiki-Helm-Chart](https://github.com/xwiki-contrib/xwiki-helm) wird dabei als Abhängigkeit eingebunden.

Am Ende ist XWiki unter `https://quickstart.k3ces.localdomain/xwiki/` erreichbar.

Anmeldung am CES, Warp-Menü, Backups und Updates werden hier nicht behandelt.

Das Ergebnis dieses Guides (inklusive des [Advanced-Guides](advanced-guide_de.md)) kann im [xwiki-dogu-Ordner](xwiki-dogu) gefunden werden.

## Voraussetzungen

Zuerst die [lokale Entwicklungsumgebung](prerequisites_de.md) einrichten. Danach die Kubeconfig setzen:

```shell
export KUBECONFIG="${HOME}/.kube/quickstart.k3ces.localdomain"
kubectl cluster-info
```

## Parent-Chart erstellen

Ein neues Verzeichnis für das Dogu-Chart erstellen:

```shell
mkdir xwiki-dogu
cd xwiki-dogu
mkdir templates
```

Die Datei `Chart.yaml` erstellen:

```yaml
apiVersion: v2
name: xwiki
description: XWiki als Dogu V3
type: application
# Version des Dogu-Charts
version: 1.7.1-1
# Version von XWiki
appVersion: "18.6.0"
annotations:
  # Dogu-API-Version
  dogu.cloudogu.com/api-version: v3
dependencies:
  - name: xwiki
    version: 1.7.1
    repository: https://xwiki-contrib.github.io/xwiki-helm
```

Das neue Chart ist das Parent-Chart. Es enthält die Dogu-V3-Metadaten und bindet das XWiki-Chart `1.7.1` als Abhängigkeit ein.

Für Dogu V3 werden diese Angaben in `Chart.yaml` gepflegt. Eine eigene `dogu.json` wird nicht verwendet.

## XWiki konfigurieren

Im selben Verzeichnis wie `Chart.yaml` die Datei `values.yaml` erstellen:

```yaml
# Werte für das eingebundene XWiki-Chart
xwiki:
  fullnameOverride: xwiki

  # Öffentliches XWiki-Image direkt von Docker Hub laden
  image:
    name: docker.io/library/xwiki
    tag: 18.6.0-mysql-tomcat
    pullPolicy: IfNotPresent

  # XWiki unter /xwiki bereitstellen
  extraEnvVars:
    - name: CONTEXT_PATH
      value: xwiki

  # XWiki erst starten, wenn MySQL erreichbar ist
  initContainers:
    database:
      enabled: true

  # Kubernetes-Prüfungen an den Pfad /xwiki anpassen
  probes:
    startup:
      httpGet:
        path: /xwiki/rest
    liveness:
      httpGet:
        path: /xwiki/rest
    readiness:
      httpGet:
        path: /xwiki/rest/wikis/xwiki/spaces

  mysql:
    auth:
      rootPassword: ""
      username: xwiki
      password: ""
      database: xwiki
```

Das XWiki-Chart aktiviert MySQL bereits als Standarddatenbank. Benutzer und Datenbank heißen weiterhin `xwiki`. Wenn die beiden Passwortfelder leer sind, erzeugt das eingebundene MySQL-Chart beim ersten Installieren zufällige Passwörter. Es speichert sie im Kubernetes-Secret `xwiki-mysql`. XWiki liest das Benutzerpasswort aus demselben Secret.

Bei einem erneuten `helm upgrade --install` bleibt das vorhandene Passwort erhalten.

## Exposition hinzufügen

Eine `Exposition` macht XWiki über das CES-Gateway erreichbar. Die Datei `templates/exposition.yaml` erstellen:

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: Exposition
metadata:
  name: {{ .Release.Name }}
  labels:
    app.kubernetes.io/name: xwiki
    app.kubernetes.io/instance: {{ .Release.Name }}
spec:
  http:
    - name: web
      service: {{ .Values.xwiki.fullnameOverride }}
      port: {{ .Values.xwiki.service.externalPort }}
      path: /xwiki
```

## NetworkPolicies

Die CES-Umgebung sperrt eingehenden Verkehr im Namespace `ecosystem` standardmäßig über NetworkPolicies.
Dementsprechend müssen NetworkPolicies hinzugefügt werden, wenn eingehender Verkehr zwischen Pods des Dogus erlaubt werden soll.
In diesem Beispiel erstellt das mitgelieferte MySQL-Chart bereits eine eigene NetworkPolicy für die Datenbank.

NetworkPolicies für die Exposition / das Gateway müssen nicht hinzugefügt werden, da diese automatisch für jede Exposition erstellt werden.

## Abhängigkeit laden

```shell
helm dependency build .
```

Helm lädt das XWiki-Chart in das Verzeichnis `charts/`. Dessen MySQL-Abhängigkeit ist bereits im Paket enthalten. Die erzeugte Datei `Chart.lock` hält die gewählte XWiki-Chart-Version fest.

## Chart prüfen und installieren

```shell
helm lint .

helm upgrade --install xwiki . \
  --namespace ecosystem \
  --wait \
  --timeout 15m
```

`helm upgrade --install` installiert das Release beim ersten Aufruf. Bei weiteren Aufrufen aktualisiert Helm die vorhandene Installation. Der erste Start kann mehrere Minuten dauern.

## Installation prüfen

```shell
helm status xwiki --namespace ecosystem

kubectl get statefulset,pod,service,exposition,networkpolicy \
  --namespace ecosystem \
  --selector app.kubernetes.io/instance=xwiki

kubectl get persistentvolumeclaim \
  xwiki-data-xwiki-0 \
  data-xwiki-mysql-0 \
  --namespace ecosystem
```

Die StatefulSets `xwiki` und `xwiki-mysql` müssen jeweils `1/1` bereit sein. Die PVCs `xwiki-data-xwiki-0` und `data-xwiki-mysql-0` müssen den Status `Bound` haben.

Im Browser öffnen:

```text
https://quickstart.k3ces.localdomain/xwiki/
```

Die Zertifikatswarnung ist bei der lokalen, selbst signierten Umgebung normal. Danach erscheint der Einrichtungsassistent von XWiki.

## Chart wieder entfernen

```shell
helm uninstall xwiki --namespace ecosystem
```

Helm entfernt die Anwendung, aber die PVCs mit den Daten bleiben erhalten. Für einen vollständigen Neustart ohne alte Daten können sie zusätzlich gelöscht werden:

```shell
kubectl delete persistentvolumeclaim \
  xwiki-data-xwiki-0 \
  data-xwiki-mysql-0 \
  --namespace ecosystem
```
