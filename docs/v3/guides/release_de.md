# Ein Dogu V3 releasen

Diese Anleitung veröffentlicht ein Dogu mit `cldk dogu release` in der Cloudogu-OCI-Registry `registry.cloudogu.com`. Das Helm-Chart ist das Release-Artefakt, sein Push ist das Release.

## Voraussetzungen

- ein Dogu-Repository mit Helm-Chart und `cldk.yaml`, das im Multinode EcoSystem getestet wurde, z. B. mit `cldk dogu dev-push` im [lokalen EcoSystem](../getting-started/quickstart_de.md),
- [`cldk`](https://github.com/cloudogu/cldk/blob/develop/README.md),
- Docker oder ein anderer Docker-kompatibler Builder, wenn das Dogu eigene Container-Images baut, und
- ein Registry-Zugang von Cloudogu.

## Schritte

### 1. Registry-Zugang beantragen

Aktuell gibt es keine Selbstverwaltung für Registry-Accounts von Partnern. Der Zugang wird über die zuständige Kontaktperson bei Cloudogu beantragt und geändert. Die Anfrage sollte enthalten:

- Unternehmen und technische Kontaktperson,
- Name des Dogus.

Cloudogu stellt dann einen Harbor-Robot-Account für `registry.cloudogu.com` bereit, den zugewiesenen Dogu-Namespace und die Berechtigung, in diesem Namespace Charts und Images zu pullen und zu pushen.

Wird der Zugang nicht mehr benötigt, wird die Cloudogu-Kontaktperson gebeten, den Robot-Account zu deaktivieren. Die Anfrage enthält Accountname, Namespace und gewünschtes Löschdatum. Ein abgelaufener Account wird nur verlängert, wenn ein weiteres Release geplant ist.

### 2. Release vorbereiten

In der `Chart.yaml`:

- `version` ist die Dogu-Version und folgt der semantischen Versionierung. Eine bereits veröffentlichte Version darf nicht erneut verwendet werden.
- `appVersion` ist die Version der enthaltenen Anwendung. Sie ist unabhängig von `version`.
- `annotations.dogu.cloudogu.com/api-version` ist `v3`.

In der `cldk.yaml` den zugewiesenen Dogu-Namespace und die Release-Tags der Images setzen, siehe [Release-Tags in der `cldk.yaml`](https://github.com/cloudogu/cldk/blob/develop/docs/operations/releasing_a_dogu_de.md#release-tags-in-der-cldkyaml).

Alle vom Chart verwendeten Images stehen in der `mirror-patch-tpl.yaml`, damit sie für ein Offline EcoSystem gespiegelt werden können. Der Abschnitt [Images für die spätere Spiegelung vorbereiten](../getting-started/advanced-guide_de.md#images-für-die-spätere-spiegelung-vorbereiten) erklärt den Aufbau.

Abhängigkeiten des Charts laden, falls die `Chart.yaml` welche deklariert. Die Chart-Repositories der Abhängigkeiten müssen dafür mit `helm repo add` eingetragen sein:

```shell
helm dependency build <chart-verzeichnis>
```

### 3. Anmelden

Mit dem Robot-Account anmelden:

```shell
docker login registry.cloudogu.com
```

`cldk` und der Builder verwenden diese Anmeldung. Das Passwort gehört nicht in das Chart, das Quellcode-Repository oder ein Shell-Skript. Für automatische Releases wird die Zugangsdatenverwaltung des CI-Systems verwendet.

### 4. Release prüfen und veröffentlichen

Im Wurzelverzeichnis des Dogu-Repositorys zuerst einen Probelauf starten. Er prüft `cldk.yaml`, Chart, Release-Tags und Ziele und zeigt, was veröffentlicht werden würde:

```shell
cldk dogu release --dry-run
```

Danach veröffentlichen:

```shell
cldk dogu release
```

Der DCC übernimmt das Chart automatisch und liest die Metadaten aus der `Chart.yaml` und den weiteren Dateien im Chart. Eine `dogu.json` oder ein separater Aufruf des DCC ist nicht nötig.

Alle Optionen, das Release eines bereits gepackten Charts und Releases mit Images aus anderen Registries beschreibt [Ein Dogu mit `cldk dogu release` veröffentlichen](https://github.com/cloudogu/cldk/blob/develop/docs/operations/releasing_a_dogu_de.md).

### 5. Release-Commit taggen

Bei einem öffentlichen Quellcode-Repository empfiehlt es sich, den Commit des Releases zu taggen:

```shell
git tag -a "release/chart/<version>" -m "<version>

chart-version: <version>
app-version: <app-version>

this is a Dogu API v3 release"

git push origin "release/chart/<version>"
```

## Ergebnis prüfen

```shell
helm show chart \
  "oci://registry.cloudogu.com/<namespace>/dogu/v3/charts/<dogu>" \
  --version "<version>"
```

Die Ausgabe zeigt die `Chart.yaml` des veröffentlichten Charts mit der erwarteten `version`.

## Häufige Fehler

| Symptom | Ursache und Lösung |
| --- | --- |
| `401 Unauthorized` | Benutzername oder Passwort sind falsch. Erneut mit `docker login registry.cloudogu.com` anmelden. |
| `403 Forbidden` | Der Account darf nicht in den gewählten Namespace pushen. `namespace` in der `cldk.yaml` prüfen. |

Weitere Fehlerbilder beschreibt die [Fehlersuche von `cldk dogu release`](https://github.com/cloudogu/cldk/blob/develop/docs/operations/releasing_a_dogu_de.md#fehlersuche).

## Weiterführende Informationen

- [Dogu-V3-Artefakte verstehen](../concepts/artifacts_de.md)
- [README von `cldk`](https://github.com/cloudogu/cldk/blob/develop/README.md#dogu-release)
