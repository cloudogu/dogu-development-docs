# Chart paketieren und Fehler untersuchen

Diese Seite ergänzt die Anleitung [Vom Helm-Chart zum Dogu V3](quickstart_de.md). Sie zeigt den lokalen OCI-Workflow und einige Diagnosebefehle.

## Chart in die lokale OCI-Registry pushen

Bei einem neuen Terminal zuerst in das Verzeichnis `xwiki-dogu` wechseln. Danach die Chart-Version setzen:

```shell
export CHART_VERSION='1.7.1-1'
```

Die Abhängigkeiten laden und das Chart paketieren:

```shell
helm dependency build .
helm lint .

mkdir -p dist
helm package . --destination dist
```

Das Paket in die lokale Registry pushen:

```shell
helm push "dist/xwiki-${CHART_VERSION}.tgz" \
  oci://localhost:5001/quickstart/charts \
  --plain-http
```

Das Chart zur Kontrolle wieder herunterladen:

```shell
mkdir -p /tmp/xwiki-chart

helm pull oci://localhost:5001/quickstart/charts/xwiki \
  --version "${CHART_VERSION}" \
  --plain-http \
  --destination /tmp/xwiki-chart
```

Damit ist geprüft, dass die lokale Registry das Helm-Chart speichern kann. Ein Release in die Cloudogu-Registry gehört nicht zu dieser Anleitung.

## Aus der OCI-Registry installieren

Das gepushte Chart lässt sich direkt mit Helm installieren:

```shell
export KUBECONFIG="${HOME}/.kube/quickstart.k3ces.localdomain"

helm upgrade --install xwiki \
  oci://localhost:5001/quickstart/charts/xwiki \
  --version "${CHART_VERSION}" \
  --plain-http \
  --namespace ecosystem \
  --wait \
  --timeout 15m
```

Das Paket enthält das Parent-Chart, seine Werte und das XWiki-Chart als Abhängigkeit.

Auch dieser Befehl installiert das Chart direkt mit Helm. Der Dogu-Operator verwaltet dieses Release noch nicht.

## Fehler untersuchen

Zuerst den Status des Helm-Releases und der Pods prüfen:

```shell
helm status xwiki --namespace ecosystem
helm get values xwiki --namespace ecosystem

kubectl get pods --namespace ecosystem \
  --selector app.kubernetes.io/instance=xwiki
kubectl get events --namespace ecosystem --sort-by=.lastTimestamp
```

### XWiki-Image kann nicht geladen werden

```shell
kubectl describe pod xwiki-0 --namespace ecosystem
```

Das Parent-Chart lädt das öffentliche Image direkt von Docker Hub:

```text
docker.io/library/xwiki:18.6.0-mysql-tomcat
```

Der Cluster benötigt dafür Zugriff auf Docker Hub.

### MySQL oder der Datenbank-Check startet nicht

```shell
kubectl describe pod xwiki-mysql-0 --namespace ecosystem
kubectl logs xwiki-mysql-0 --namespace ecosystem
kubectl logs xwiki-0 --container wait-for-db --namespace ecosystem
```

Der Pod `xwiki-mysql-0` muss bereit sein, bevor XWiki startet.

Die zufällig erzeugten Datenbankpasswörter liegen im Secret `xwiki-mysql`. Das Benutzerpasswort lässt sich zu Diagnosezwecken so anzeigen:

```shell
kubectl get secret xwiki-mysql \
  --namespace ecosystem \
  --output jsonpath='{.data.mysql-password}' | base64 --decode
echo
```

### Ein PVC wird nicht gebunden

```shell
kubectl get persistentvolumeclaim --namespace ecosystem
kubectl describe persistentvolumeclaim xwiki-data-xwiki-0 --namespace ecosystem
kubectl describe persistentvolumeclaim data-xwiki-mysql-0 --namespace ecosystem
```

Beide PVCs müssen den Status `Bound` haben. Die lokale Quickstart-Umgebung stellt den benötigten Speicher über die StorageClass `local-path` bereit.

### Die URL liefert `502 Bad Gateway`

Service, Endpunkte und NetworkPolicy prüfen:

```shell
kubectl get service,endpointslice --namespace ecosystem \
  --selector app.kubernetes.io/instance=xwiki

kubectl get networkpolicy xwiki-gateway \
  --namespace ecosystem \
  --output yaml
```

Das CES-Gateway benötigt Zugriff auf Port `8080` des XWiki-Pods.

### XWiki wird nicht bereit

```shell
kubectl describe pod xwiki-0 --namespace ecosystem
kubectl logs xwiki-0 --namespace ecosystem
```

Die Pfade der drei Kubernetes-Prüfungen müssen mit `/xwiki` beginnen. Außerdem muss im Pod `CONTEXT_PATH=xwiki` gesetzt sein:

```shell
kubectl get statefulset xwiki --namespace ecosystem \
  --output jsonpath='{.spec.template.spec.containers[0].env}'
echo
```

### Die Exposition wird nicht verarbeitet

```shell
kubectl describe exposition xwiki --namespace ecosystem
kubectl logs deployment/k8s-service-discovery-controller-manager \
  --namespace ecosystem
```

Die Bedingungen `Valid` und `IngressesReady` müssen `True` sein.

## Aufräumen

Nur XWiki und MySQL entfernen:

```shell
helm uninstall xwiki --namespace ecosystem
```

Die PVCs bleiben dabei erhalten. Sie müssen nur gelöscht werden, wenn auch die gespeicherten Daten entfernt werden sollen:

```shell
kubectl delete persistentvolumeclaim \
  xwiki-data-xwiki-0 \
  data-xwiki-mysql-0 \
  --namespace ecosystem
```

Den gesamten Cluster im Verzeichnis `k8s-ecosystem` entfernen:

```shell
./k3d/ces-k3d delete quickstart
```

Die Registry-Container und der Registry-Speicher bleiben bestehen. Sie können von weiteren lokalen CES-Umgebungen verwendet werden.
