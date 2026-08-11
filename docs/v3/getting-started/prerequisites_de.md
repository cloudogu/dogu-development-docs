# Lokale Entwicklungsumgebung einrichten

Diese Anleitung richtet ein vollständiges Cloudogu EcoSystem in einem lokalen k3d-Cluster ein. Diese Umgebung wird für die weiteren Beispiele benötigt.

Der Cluster ist nur für Entwicklung und Tests gedacht. Er besteht aus einem Knoten und verwendet lokalen Speicher. Mehrere Knoten, Backups und das Vergrößern von Volumes werden nicht unterstützt.

## Was wird benötigt?

- ein Linux-Rechner mit ausreichend freien Ressourcen
- Zugangsdaten für die Cloudogu-Registries
- Docker
- Git
- Go ab Version 1.26
- [k3d](https://k3d.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- jq
- [mikefarah/yq](https://github.com/mikefarah/yq) ab Version 4

Partner erhalten die Zugangsdaten von ihrer zuständigen Kontaktperson bei Cloudogu. Die Installation lädt geschützte CES-Komponenten. Deshalb reichen die öffentlichen Repositories allein nicht aus.

## Cloudogu EcoSystem erstellen

```shell
./code/init_ecosystem
```

Die Installation kann einige Minuten dauern. Der Name `quickstart` ist fest, weil er zur Adresse in den beiden Konfigurationsdateien passen muss.

## Cluster verwenden

Die Kubeconfig der Umgebung aktivieren (kann z. B. auch in die `.bashrc` oder `.zshenv` eingetragen werden):

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
# Cluster anhalten und starten
./code/k8s-ecosystem/k3d/ces-k3d stop quickstart
./code/k8s-ecosystem/k3d/ces-k3d start quickstart
# alle Cluster auflisten
./code/k8s-ecosystem/k3d/ces-k3d list
# Cluster entfernen
./code/k8s-ecosystem/k3d/ces-k3d delete quickstart
# Cluster (neu) erstellen
./code/k8s-ecosystem/k3d/ces-k3d create quickstart
```

Nach dem Löschen den Eintrag für `quickstart.k3ces.localdomain` aus `/etc/hosts` entfernen. Die gemeinsam verwendeten Registry-Container und deren Speicher bleiben bestehen.

Als Nächstes: [Ein bestehendes Helm-Chart in ein Dogu V3 überführen](quickstart_de.md).
