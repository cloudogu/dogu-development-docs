# Ein Dogu V3 releasen

Diese Anleitung erklärt, wie ein Dogu V3 in der Cloudogu-OCI-Registry veröffentlicht wird. Das Helm-Chart ist das Release-Artefakt. Der Push des Charts startet die Übernahme in die Dogu-Registry.

## Voraussetzungen

Benötigt werden:

- ein Dogu-V3-Helm-Chart, das in einem Multinode EcoSystem lauffähig ist,
- Helm 3.8 oder neuer,
- Docker, wenn nicht öffentliche Container-Images veröffentlicht werden, und
- ein Registry-Zugang von Cloudogu.

Cloudogu stellt folgende Zugangsdaten bereit:

- einen Harbor-Robot-Account,
- den zugewiesenen Dogu-Namespace und
- die Berechtigung, Charts und bei Bedarf Images in diesem Namespace zu pullen und zu pushen.

Der Robot-Account ist ein technischer Account für `registry.cloudogu.com`.

## Registry-Zugang beantragen und verwalten

Aktuell gibt es keine Selbstverwaltung für Registry-Accounts von Partnern. Der Zugang wird über die zuständige Kontaktperson bei Cloudogu beantragt und geändert.

### Zugang beantragen

Die Anfrage sollte folgende Informationen enthalten:

- Unternehmen und technische Kontaktperson,
- Name des Dogus.

### Zugang entfernen

Wenn der Zugang nicht mehr benötigt wird, muss die Cloudogu-Kontaktperson gebeten werden, den Robot-Account zu deaktivieren. Die Anfrage sollte Accountname, Namespace und gewünschtes Löschdatum enthalten. Ein abgelaufener Account sollte nur verlängert werden, wenn ein weiteres Release geplant ist.

## Anmelden

Mit dem Robot-Account anmelden. Helm fragt Benutzername und Passwort ab:

```shell
helm registry login registry.cloudogu.com
```

Wenn auch Images veröffentlicht werden, zusätzlich mit Docker anmelden:

```shell
docker login registry.cloudogu.com
```

Das Passwort darf nicht im Chart, Quellcode-Repository oder in einem Shell-Skript gespeichert werden. Für automatische Releases muss die Zugangsdatenverwaltung des CI-Systems verwendet werden.

## Release vorbereiten

Vor jedem Release muss die `Chart.yaml` aktualisiert werden:

- `version` ist die Dogu-Version und muss der semantischen Versionierung folgen,
- `appVersion` ist die Version der enthaltenen Anwendung und
- `annotations.dogu.cloudogu.com/api-version` muss `v3` sein.

Chartname und Version identifizieren das veröffentlichte Artefakt. Eine bereits veröffentlichte Version darf nicht erneut verwendet werden.

Abhängigkeiten laden, Chart prüfen und Paket erstellen:

```shell
helm dependency build .
helm lint .

mkdir -p dist
helm package . --destination dist
```

Vor der Veröffentlichung sollte das Chart außerdem im unterstützten Multinode EcoSystem installiert und getestet werden. Die [Dogu-V3-Release-Checkliste](../concepts/multinode-environment_de.md#checkliste-vor-einem-release) nennt die wichtigen Tests.

Bei einem öffentlichen Quellcode-Repository muss der Commit des Releases getaggt werden:

```shell
git tag -a "release/chart/<version>" -m "<version>

chart-version: <version>
app-version: <app-version>

this is a Dogu API v3 release"

git push origin "release/chart/<version>"
```

Wenn das Quellcode-Repository einen eigenen Release-Ablauf vorgibt, gelten dessen Regeln. Falls der Push des Tags ein automatisches Release startet, dürfen dieselben Artefakte nicht zusätzlich manuell veröffentlicht werden.

## Dogu-eigene Container-Images veröffentlichen

Alle vom Chart verwendeten Images müssen in der `chart-patch-tpl.yaml` aufgeführt sein. Sie beschreibt die ursprünglichen Image-Adressen und deren Verwendung in der `values.yaml`, damit die Images für ein Offline EcoSystem gespiegelt und die Adressen angepasst werden können. Der Abschnitt [Images für die spätere Spiegelung vorbereiten](../getting-started/advanced-guide_de.md#images-für-die-spätere-spiegelung-vorbereiten) erklärt den Aufbau der Datei.

Wenn das Chart ausschließlich öffentliche Images aus anderen Registries verwendet, können die folgenden Push-Schritte übersprungen werden.

Alle Dogu-eigenen Container-Images müssen vor dem Chart veröffentlicht werden. Dafür werden der zugewiesene Namespace und der Image-Pfad für Dogu V3 verwendet:

```shell
docker tag "<lokales-image>:<tag>" \
  "registry.cloudogu.com/<namespace>/dogu/v3/images/<image>:<tag>"

docker push \
  "registry.cloudogu.com/<namespace>/dogu/v3/images/<image>:<tag>"
```

Die Image-Referenz im Chart muss dieselbe Adresse und denselben Tag verwenden.

## Chart veröffentlichen

Das Chart darf erst gepusht werden, wenn alle benötigten Images verfügbar sind:

```shell
helm push "dist/<dogu>-<version>.tgz" \
  "oci://registry.cloudogu.com/<namespace>/dogu/v3/charts"
```

Dieser Push ist das Dogu-V3-Release. Der DCC übernimmt das Chart automatisch und liest die Metadaten aus der `Chart.yaml` und den weiteren Dateien im Chart. Ein zusätzlicher Upload einer `dogu.json` oder ein separater Aufruf des DCC ist nicht erforderlich.

## Release prüfen

Das Chart aus der OCI-Registry lesen:

```shell
helm show chart \
  "oci://registry.cloudogu.com/<namespace>/dogu/v3/charts/<dogu>" \
  --version "<version>"
```

## Mögliche Fehler

- `401 Unauthorized`: Benutzername oder Passwort sind falsch,
- `403 Forbidden`: Der Account besitzt nicht die erforderliche Berechtigung für den gewählten Namespace, und
- eine vorhandene Version kann nicht ersetzt werden: Es muss eine neue Chart-Version gewählt werden.
