# Interessantes für Entwickelnde

Dieses Kapitel beschreibt all das, das über die allgemeine oder spezielle Dogu-Entwicklung hinausgeht, aber neuen Entwicklern auf die (Dogu-)Sprünge hilft oder Ideen für Horizonterweiterung gibt.

## Containers

Der Bau von Container-Images ist zentraler Gegenstand der Dogu-Entwicklung.  
Die folgenden Abschnitte gehen auf übliche Best Practices ein.  
Diese Best Practices stammen sowohl aus der allgemeinen Container- als auch aus der Dogu-Entwicklung.

### 12-Factor-App

Der Begriff ["12-Factor App"][12-factor] bezieht sich auf eine Methodensammlung zum Erstellen von Software-as-a-Service-Anwendungen.
Diese Methoden zielen darauf ab, Anwendungen portabel und ausfallsicher zu gestalten.
Dabei sollen Unterschiede zwischen **Entwicklungsumgebung und Ausführungsumgebung** so weit minimiert werden, dass Produktivprobleme leichter nachvollzogen und bewältigt werden können.

Die 12 Faktoren sind Folgende:
1. **Codebase:** Eine im Versionsmanagementsystem verwaltete Codebase, viele Deployments
2. **Abhängigkeiten:** Abhängigkeiten explizit deklarieren und isolieren
3. **Konfiguration:** Die Konfiguration in Umgebungsvariablen ablegen
4. **Unterstützende Dienste:** Unterstützende Dienste als angehängte Ressourcen behandeln
5. **Build, release, run:** Build- und Run-Phase strikt trennen
6. **Prozesse:** Die App als einen oder mehrere Prozesse ausführen
7. **Bindung an Ports:** Dienste durch das Binden von Ports exportieren
8. **Nebenläufigkeit:** Mit dem Prozess-Modell skalieren
9. **Einweggebrauch:** Robuster mit schnellem Start und problemlosen Stopp
10. **Dev-Prod-Vergleichbarkeit:** Entwicklung, Staging und Produktion so ähnlich wie möglich halten
11. **Logs:** Logs als Strom von Ereignissen behandeln
12. **Admin-Prozesse:** Admin/Management-Aufgaben als einmalige Vorgänge behandeln

[12-factor]: https://12factor.net/de/

### Container-Images

Beim Image-Bau sollte auf die folgenden Aspekte betrachtet werden:
- möglichst root-less Container aufbauen
- wenn möglich aktuelle Version aller eingesetzten [Tools][container-tools] / [base-Images][base-images] oder Commits nutzen
  - Achtung: Neue alpine-Software-Version durch Erhöhung des base-Image möglich!
  - Aktualisierung der Tools aus dem base-Image
    - bspw. `apk update && apk upgrade`
- wenn sinnvoll auf eine möglichst geringe Imagegröße achten
- möglichst wenige Statements im Dockerfile
  - nur **ein** [COPY-Statement][copy-statement] für Dateisystemstruktur im Container
- LABELs für Metadaten verwenden
  - `LABEL maintainer="hello@cloudogu.com"` statt `MAINTAINER`-Statement
  - `NAME="namespace/dogu-name"` rein
  - `VERSION ="w.x.y-z"`, wird vom automatischen Release-Prozess übernommen
- Dockerfile enthält einen [Healthcheck][healthcheck]
  - z.B.: `HEALTHCHECK CMD doguctl healthy nexus || exit 1`
- Downloads (mit curl/wget o. Ä.) werden mit Check-Summen/Hashes geprüft

[container-tools]: https://github.com/cloudogu/base/blob/3466b5e95c25a6c5ac569069167e513c71815797/Dockerfile#L10
[base-images]: https://github.com/cloudogu/sonar/blob/8e389605d1f2fa7720d725a1cca6692f4c6b77e3/Dockerfile#L1
[healthcheck]: https://github.com/cloudogu/sonar/blob/8e389605d1f2fa7720d725a1cca6692f4c6b77e3/Dockerfile#L3
[copy-statement]: https://github.com/cloudogu/sonar/blob/8e389605d1f2fa7720d725a1cca6692f4c6b77e3/Dockerfile#L33

## Resources-Ordner & Dogu-Skripte

Für die Dogu-Entwicklung bei Cloudogu verwenden wir einen Resources-Ordner.
Wir empfehlen auch anderen Dogu-Entwicklungsteams, diese Struktur zu verwenden.
Diese Entscheidung ist aber dem Team selbst überlassen.

Dieser enthält alle wichtigen Ressourcen, die ein Dogu zur Laufzeit benötigt.
Der Ordner wird mit derselben Struktur wie das Container-Dateisystem aufgebaut.

Beispiel:
- `resources/`
  - `create-sa.sh` (optional)
  - `remove-sa.sh` (optional)
  - `post-upgrade.sh` (optional)
  - `pre-upgrade.sh` (optional)
  - `upgrade-notification.sh` (optional)
  - `startup.sh` (empfohlen)

Das Beispiel zeigt den Inhalt eines resource-Ordners eines Dogus.

Der komplette Ordner wird im Dockerfile in das Root-Verzeichnis des Dogu-Containers kopiert.  
Damit würde eine vorhandene Konfiguration der App im Pfad `/etc/your_app/config.json` überschrieben werden.  
Durch diesen Ansatz ist es möglich alle Dateien des Containers anzupassen.

### Bash-Skripte

Das Anlegen von Bash-Skripten in einem Dogu ist notwendig, um verschiedene Abläufe zu steuern.  
Darunter fällt das Starten und Upgraden des Dogus, sowie das Erstellen eines Service Accounts.

Generell sollten alle Skripte folgende Konventionen erfüllen:

#### Bash-Strict-Mode

Bei Bash-Skripten soll immer der [Bash-Strict-Mode][strict-mode] verwendet werden.  
Dieser verhindert bei einem Fehlverhalten die weitere Ausführung des Skripts.
Generell ist es sinnvoll, alle potenziellen Fehler im Startskript abzufangen und mit einer eigenen Fehlermethode zu berichten.  
Diese könnte folgendermaßen aussehen:

```shell
# This function prints an error to the console and waits 5 minutes before exiting the process.
function exitOnErrorWithMessage() {
  message=${1}
  echo -e "ERROR: ${message}. Exiting in 300 seconds."
  sleep 300
  exit 1
}
```

Solch eine Methode macht das komplette Skript lesbarer und bremst den Neustart-Loop des Dogus sowie die Masse der generierten Logs.

[strict-mode]: http://redsymbol.net/articles/unofficial-bash-strict-mode/

#### Line-Endings

Für alle Skripte sollten jederzeit Unix-basierte Line-Endings benutzt werden.

#### Doguctl verwenden

[Doguctl][doguctl] ist ein Kommandozeilentool, um die Konfiguration eines Dogus zu vereinfachen.  
Informationen über die Benutzung finden sich in ["Die Nutzung von doguctl"][doguctl-usage].

Wo anwendbar, soll immer Doguctl in Bash-Skripten wie der `startup.sh` verwendet werden.  
Für die sinnvolle Benutzung kann das [Startup-Skript vom Nexus][doguctl-example] als Beispiel genommen werden.

[doguctl]: https://github.com/cloudogu/doguctl
[doguctl-usage]: ../important/relevant_functionalities_de.md#die-nutzung-von-doguctl
[doguctl-example]: https://github.com/cloudogu/nexus/blob/4d2de3733eca684df1363179c527363a4d31526e/resources/startup.sh

### Dogu-Startskript: startup.sh

Das [Dogu-Start-Skript][startup-sh] ist der Einstiegspunkt eines Dogu-Containers und verwaltet alle Schritte, die für einen reibungslosen Start des Dogus notwendig sind.
Konventionell wird das Startup-Skript mit dem Namen `startup.sh` in dem `resources`-Ordner des Dogus angelegt.

[startup-sh]: ../important/relevant_functionalities_de.md#aufbau-und-best-practices-von-startupsh

## Schnelle Feedback-Zyklen durch Tool-Entwicklung ohne CES

In der Entwicklung ist eine rasche Erkenntnis wichtig, ob der richtige Entwicklungspfad beschritten wurde.  
Ein schnelles Feedback ist umso wichtiger, je komplexer die Umgebung wird.

In der Dogu-Entwicklung lässt sich schnelles Feedback auf unterschiedliche Weise erreichen:

### Multi-Stage-Build

Für eine effektive Entwicklung wird der Einsatz eins Multi-Stage-Builds für das Dogu-Image empfohlen.
Diese bieten Vorteile wie ein effektiveres Caching von Downloads, schnellere Bauzeiten, effektives Layering und kleine Image-Größen.
Mehr Information zu Multi-Stage-Build gibt es auf der [offiziellen Website von Docker][docker-multi-stage].

[docker-multi-stage]: https://docs.docker.com/develop/develop-images/multistage-build/

### CES-freie Entwicklungsumgebung

Das Bauen des Dogus und das Ausliefern ins lokale EcoSystem fordert nicht nur viel Zeit, sondern nimmt auch die Möglichkeit die Software effektiv zu debuggen.  
Daher ist es sinnvoll eine lokale Entwicklungsumgebung unabhängig vom CES aufzubauen, um die Dogu-Software effektiv auszuführen, zu testen und zu debuggen.

Die lokale Entwicklungsumgebung muss alle abhängigen CES-Dienste durch lokal gestartete Alternativen zu Verfügung stellen.
Wir empfehlen, dass eine lokale Entwicklungsumgebung in Form einer `docker-compose.yml`-Datei definiert ist und jederzeit einfach mit `docker-compose up` gestartet und mit `docker-compose down` heruntergefahren werden kann.  
Für ein Beispiel [siehe CAS][local-cas-example].

[local-cas-example]: https://github.com/cloudogu/cas/blob/develop/app/docker-compose.yml

## Qualitätssicherung für Dogus

### Container Validation (GOSS)

Die Container-Validation versichert die richtige Konfiguration des Containers, die für eine reibungslose Ausführung der Software notwendig ist.  

Dafür verwenden wir in unseren Dogus das Server-Validation Programm [goss][goss].  
Die eigentlichen goss-Tests werden in der folgenden Datenstruktur im Dogu angelegt:
- `<Root-Verzeichnis des Dogus>/`
  - `spec/`
    - `goss/`
      - `goss.yml` (enthält alle goss-Tests)

Die goss-Tests sollten am besten alle wichtigen Aspekte für den reibungslosen Start eines Dogus überprüfen.  
Darunter fallen Aspekte wie:
- Überprüfen der Berechtigung für Skripte (startup.sh), Volumes
- Überprüfen der UID & GID für Dateien (startup.sh, resources, Volumes, Schreib-abhängige Pfade)
- Erreichbarkeit der Software: TCP-Health-Checks

Für die Integration in eine Pipeline empfehlen wir den Einsatz der `ecoSystem.verify()` Methode aus der [dogu-build-lib][dogu-build-lib].  
Ein beispielhafter Aufruf sieht in einem Jenkinsfile wie folgt aus:

```groovy
stage('Verify') {
    ecoSystem.verify("/dogu")
}
```

Dieser ruft intern den Befehl `cesapp verify` auf, welcher die Goss-Tests ausführt.

[goss]: https://github.com/goss-org/goss
[dogu-build-lib]: https://github.com/cloudogu/dogu-build-lib

## Dokumentation

Jedes Tool sollte eine sinnvolle Dokumentation über verwendete Features besitzen, die während der Benutzung oder Support unterstützt.
Um eine einfache Integration in unsere vorhandene Dokumentations-Infrastruktur zu ermöglichen, sollte die Dokumentation in einem im Repository-Root befindlichen Order `/docs` abgelegt werden.

Standardmäßig umfasst die Doku folgende Dinge:
- Features von dem Dogu
- Installation des Dogus
- Konfiguration des Dogus
- Entwicklung am Dogu
- Readme
- Changelog

### Lizenz

Die Nutzungslizenz des Dogus sollte deutlich hervorgehen.
Diese kann zum Beispiel in der Readme angegeben werden.

### Readme

Eine Readme ist sowohl für eigene Entwicklungen als auch im Partnerkontext mit externen Entwicklern sinnvoll.  
Bei öffentlichen Repositories sollte jedoch ein höherer Maßstab an einer Readme angelegt werden.

Eine Readme-Datei liegt im Repository-Root und beschreibt knapp Hinweise zum generellen Zweck des Dogus.
Dies kann in Form eines Quickstart-Guides sein, indem zum Beispiel beschrieben ist wie man eine lokale Entwicklungsumgebung einrichtet.
Es sollte außerdem ein Verweis auf den Dokumentationsordner beinhalten und kann darüber hinaus auch auf spezielle Seiten der Dokumentation verweisen.
Zum Schluss ist es zu empfehlen benötigte Ressourcen für den Gebrauch und einen Verantwortlichen des Dogus anzugeben.

### Changelog

Changelogs werden für Menschen geschrieben, die Neuigkeiten in neueren Releases verstehen möchten.

Das Format des Changelogs sollte nach **[keep a changelog](https://keepachangelog.com)** angelegt sein.

Neue Änderungen werden stets in die oberste Sektion `## [Unreleased]` hinzugefügt.  
Bei einem neuen Release wandern diese Änderungen in eine neue Sektion, die die entsprechende Versionsnummer enthält.

Bei allen Änderungen sollten Verweise auf die Vorgänge des verwendeten Issue-Trackers hinzugefügt werden.  
Handelt es sich außerdem um komplexere Vorgänge, ist zusätzlich ein Verweis zu einer Dokumentation sinnvoll.
