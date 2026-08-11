# Lokale Entwicklungsumgebung einrichten

Diese Anleitung richtet ein vollständiges Cloudogu EcoSystem in einem lokalen k3d-Cluster ein. Diese Umgebung wird für die weiteren Beispiele benötigt.

Der Cluster ist nur für Entwicklung und Tests gedacht. Er besteht aus einem Knoten und verwendet lokalen Speicher. Backups und das Vergrößern von Volumes werden nicht unterstützt.

## Was wird benötigt?

- ein Linux-Rechner mit ausreichend freien Ressourcen,
- Zugangsdaten für die Cloudogu-Registries,
- Docker,
- Git,
- Go ab Version 1.26,
- [k3d](https://k3d.io/),
- [kubectl](https://kubernetes.io/docs/tasks/tools/),
- [Helm](https://helm.sh/docs/intro/install/),
- curl,
- einen Texteditor wie nano,
- jq und
- [mikefarah/yq](https://github.com/mikefarah/yq) ab Version 4.

Partner erhalten die Zugangsdaten von ihrer zuständigen Kontaktperson bei Cloudogu. Die Installation lädt geschützte CES-Komponenten. Deshalb reichen die öffentlichen Repositories allein nicht aus.

Das Installationsskript erstellt beim ersten Aufruf ein kleines Hilfsprogramm. Dafür wird Go benötigt.

Die Werkzeuge prüfen:

```shell
docker version
git --version
go version
k3d version
kubectl version --client
helm version
curl --version
jq --version
yq --version
```

Die Anleitung wurde mit Docker `29.7.2`, Go `1.26.1`, k3d `5.9.0`, kubectl `1.36.3`, Helm `3.20.2`, jq `1.7` und mikefarah/yq `4.53.3` getestet.

## Installations-Repository laden

```shell
git clone https://github.com/cloudogu/k8s-ecosystem.git
cd k8s-ecosystem
git checkout 252bddc62b36537e1caadefeff21c14c9bd7f7bc
```

Der Checkout wählt den Stand aus, mit dem diese Anleitung getestet wurde. Dadurch ändern sich die Installationsskripte nicht unbemerkt.

## Zugangsdaten eintragen

Die Konfigurationsvorlage kopieren:

```shell
cp k3d/config.env.template k3d/config.env
chmod 600 k3d/config.env
```

Passwörter werden in dieser Datei Base64-kodiert eingetragen. Base64 ist keine Verschlüsselung. Die Datei darf nicht weitergegeben oder eingecheckt werden. Sie wird vom Repository bereits über `.gitignore` ausgeschlossen.

Das Passwort eingeben und Base64-kodieren:

```shell
read -rsp 'Registry-Passwort: ' REGISTRY_PASSWORD
printf '\n'
printf '%s' "${REGISTRY_PASSWORD}" | base64 -w0
printf '\n'
unset REGISTRY_PASSWORD
```

Der Befehl gibt das kodierte Passwort aus. Diesen Wert kopieren.

Anschließend `k3d/config.env` öffnen:

```shell
nano k3d/config.env
```

Diese acht Felder anpassen. Dabei überall denselben Benutzernamen und dasselbe kodierte Passwort eintragen:

```dotenv
LOCAL_REGISTRY_PROXY_USERNAME="<Benutzername>"
LOCAL_REGISTRY_PROXY_PASSWORD="<Base64-kodiertes Passwort>"

DOGU_REGISTRY_USERNAME="<Benutzername>"
DOGU_REGISTRY_PASSWORD="<Base64-kodiertes Passwort>"

IMAGE_REGISTRY_USERNAME="<Benutzername>"
IMAGE_REGISTRY_PASSWORD="<Base64-kodiertes Passwort>"

HELM_REGISTRY_USERNAME="<Benutzername>"
HELM_REGISTRY_PASSWORD="<Base64-kodiertes Passwort>"
```

Die Zugangsdaten stehen nun in `k3d/config.env`. Die weiteren Befehle müssen deshalb nicht im selben Terminal ausgeführt werden.

## Konfiguration für Dogu V3 herunterladen

Die beiden vorbereiteten Dateien in das Hauptverzeichnis von `k8s-ecosystem` laden:

```shell
curl -fsSLo .blueprint-override.yaml \
  https://raw.githubusercontent.com/cloudogu/dogu-development-docs/main/docs/v3/getting-started/code/prerequisites/.blueprint-override.yaml

curl -fsSLo .ecosystem-core-values-patch.yaml \
  https://raw.githubusercontent.com/cloudogu/dogu-development-docs/main/docs/v3/getting-started/code/prerequisites/.ecosystem-core-values-patch.yaml
```

Die Dateien aktivieren die LOP-IDP und die Verarbeitung von `Exposition`-Ressourcen. Außerdem setzen sie die feste lokale Adresse `quickstart.k3ces.localdomain`. Die vereinfachte Passwortrichtlinie und das selbst signierte Zertifikat sind nur für die lokale Entwicklung gedacht.

## Cloudogu EcoSystem erstellen

```shell
./k3d/ces-k3d create quickstart
```

Die Installation kann einige Minuten dauern. Der Name `quickstart` ist fest, weil er zur Adresse in den beiden Konfigurationsdateien passen muss.

Nach erfolgreicher Installation zeigt der Befehl unter anderem diese Werte an:

```text
URL:
  https://quickstart.k3ces.localdomain

Dedicated kubeconfig:
  /home/<user>/.kube/quickstart.k3ces.localdomain

Registry stack:
  push:    localhost:5001
  consume: k3d-registry-proxy.localhost:5000
```

## Cluster verwenden

Die Kubeconfig der Umgebung aktivieren:

```shell
export KUBECONFIG="${HOME}/.kube/quickstart.k3ces.localdomain"
kubectl cluster-info
```

Die Adresse in `/etc/hosts` eintragen:

```shell
sudo sh -c 'echo "127.0.0.2 quickstart.k3ces.localdomain" >> /etc/hosts'
```

Falls die CLI eine andere IP ausgibt, muss diese IP verwendet werden.

Die Installation prüfen:

```shell
kubectl get pods
kubectl get components
kubectl get crd expositions.k8s.cloudogu.com
```

Erst fortfahren, wenn die benötigten Komponenten und Pods verfügbar sind. Insbesondere `k8s-service-discovery` und `lop-idp` müssen den Zustand `available` erreichen.

Beim ersten Aufruf im Browser erscheint eine Warnung, weil das lokale Zertifikat selbst signiert ist. Danach ist das CES unter folgender Adresse erreichbar:

```text
https://quickstart.k3ces.localdomain
```

## Lokale Registry

Die lokale Registry hat zwei Adressen:

- `localhost:5001` wird für Pushes vom lokalen Rechner verwendet.
- `k3d-registry-proxy.localhost:5000` wird für Images im Cluster verwendet.

Verwende in einem Helm-Chart nicht `localhost:5001`. Innerhalb eines Pods bezeichnet `localhost` den Pod selbst.

## Umgebung anhalten oder entfernen

Die Befehle werden im Verzeichnis `k8s-ecosystem` ausgeführt:

```shell
./k3d/ces-k3d stop quickstart
./k3d/ces-k3d start quickstart
./k3d/ces-k3d list
```

Die Umgebung vollständig entfernen:

```shell
./k3d/ces-k3d delete quickstart
```

Danach den Eintrag für `quickstart.k3ces.localdomain` aus `/etc/hosts` entfernen. Die gemeinsam verwendeten Registry-Container und deren Speicher bleiben bestehen.

Als Nächstes: [Ein bestehendes Helm-Chart in ein Dogu V3 überführen](quickstart_de.md).
