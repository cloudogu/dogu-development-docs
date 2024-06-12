# Interne Aspekte

Dieser Abschnitt beschreibt Aspekte, die von internen Entwicklern der Cloudogu GmbH beachten werden müssen.
Sie enthalten Anforderungen zur Qualitätssicherung oder zum Beispiel grundlegende Definitionen zu einem fertigen,
finalen Dogu.
Somit ergibt sich, dass diese Informationen für externe Entwickler nicht zwingend relevant sind.
Allerdings können die Aspekte genutzt werden, um Qualitätsstandards zu definieren oder zu verbessern.

## Qualitätssicherung

Dogus können sowohl aus selbst entwickelter Software als auch aus gekapselter Fremdsoftware bestehen.
Im Falle einer Selbstentwicklung sollten übliche Maßstäbe für Qualitätssicherung gelten, die in den folgenden
Abschnitten genannt werden.
Wenn ein Dogu Software von Drittanbietern einfängt, ist es häufig nicht sinnvoll, sämtliche Features des eigentlichen
Tools zu testen.
Der Testfokus wird dann auf die hinzugefügten Dogu-Features abgegrenzt, die typischerweise den Mehrwert eines Dogus
darstellen.

### Qualitätssicherung des Tools

Die Qualitätssicherung liegt in der Verantwortung des Inhabers.
Für unsere Software setzen wir die folgenden Aspekte für die Qualitätssicherung um:

- Unit-Tests (80 % Code-Abdeckung)
- Integrations-Tests
- End-to-End-Tests
- Werkzeuge für statische Analyse
    - SonarQube (Code-Abdeckung, Smells, Bugs, Sicherheitslücken)
    - Reviewdog, Linter (Style, Smells)
- Reviews nach 4-Augen-Prinzip

### Qualitätssicherung des Dogus

Damit eine einwandfreie Integration der Dogus in das Cloudogu EcoSystem möglich ist, sollte die Qualität des Dogus durch
verschiedene Methoden gewährleistet werden.
Dafür werden die Dogus in zwei verschiedene Phasen getestet.

#### Automatisierung durch Makefiles

Mithilfe der Makefiles können Prozesse vereinheitlicht werden. Als Unterstützung bieten wir eine Sammlung von
[Makefiles][makefiles] an.

[makefiles]: https://github.com/cloudogu/makefiles

#### Automatisierung durch CI/CD

Ein Großteil der zu testenden Features sollte automatisch in einer Build-Pipeline getestet werden.
Das erfolgreiche Durchlaufen aller CI/CD-Test ist die Voraussetzung für ein Release.
Als Unterstützung bieten wir hier unsere [ces-build-lib][ces-build-lib] und [dogu-build-lib][dogu-build-lib] an.
Beide Bibliotheken werden effektiv in allen CI/CD Pipelines unserer Dogus eingesetzt.
Zu beachten ist, dass die `dogu-build-lib` von der Google-Cloud abhängig ist.
Sie ist in der Lage, Instanzen des Cloudogu EcoSystems in der Cloud zu provisionieren, damit diese für
Integrationstests genutzt werden können.

Im Folgenden werden die verschiedenen Methoden, die in einer Pipeline zum Testen verwendet werden können, beschrieben.
Dabei gehen wir speziell auf die Unterstützung unserer Bibliotheken ein.

[ces-build-lib]: https://github.com/cloudogu/ces-build-lib
[dogu-build-lib]: https://github.com/cloudogu/dogu-build-lib

##### Shell-Tests & Linting

Mit der Entwicklung von Dogus werden voraussichtlich mehrere Bash-Skripte erstellt.
Diese sollten getestet und auf Smells überprüft werden.
Für das Linting wird der Einsatz der `ShellCheck()` Methode aus der `dogu-build-lib` empfohlen.
Ein beispielhafter Aufruf sieht in einem Jenkins-File wie folgt aus:

```groovy
stage('Shellcheck') {
    shellCheck("resources/startup.sh")
}
```

Für das Testing kann [Bats][bats] aus der `ces-build-lib` verwendet werden:

```groovy
Docker docker = new Docker(this)

stage('Bats Tests') {
    Bats bats = new Bats(this, docker)
    bats.checkAndExecuteTests()
}
```

[bats]: https://github.com/bats-core/bats-core

##### Dockerfile-Linting

Das Dockerfile sollte nach Best Practices aufgebaut und natürlich auch syntaktisch korrekt sein.
Dafür wird der Einsatz der `lintDockerfile()` Methode aus der `ces-build-lib` empfohlen.
Ein beispielhafter Aufruf sieht in einem Jenkins-File wie folgt aus:

```groovy
stage('Lint') {
    lintDockerfile()
}
```

##### End-to-End-Tests (UI, API, CAS-Plugin)

Die Absicht der End-to-End Tests liegt darin, den Mehrwert des Dogus im Cloudogu EcoSystem zu testen.
Dabei bezieht man sich zum Beispiel auf die Integration des CAS-Dogus:

- Front-Channel-Login
- Back-Channel-Login
- Front-Channel-Logout
- Back-Channel-Logout
- Änderung der Admin-Gruppe

Die `dogu-build-lib` bietet hierbei verschiedene Methoden zur Ausführung.
Ein Aufruf im Jenkins-File sieht wie folgt aus:

```groovy
stage('Integration tests') {
    ecoSystem.runCypressIntegrationTests()
}
```

Die Tests selbst müssen sich unter dem Startverzeichnis des Dogus im Ordner `integrationTests` befinden.
Es existiert außerdem eine [Bibliothek][dogu-integration-lib], die zur
Unterstützung bei der Implementierung der Tests dient.

[dogu-integration-lib]: https://github.com/cloudogu/dogu-integration-test-library

##### Trivy

Es ist wichtig die Sicherheitslücken in einem Docker-Image, bzw. dem Zielsystem zu identifizieren und zu beheben.
Dafür bietet sich der Scanner Trivy an, welcher Sicherheitslücken im System identifiziert und priorisiert.
Die `dogu-build-lib` bietet dafür eine [Komponente][trivy] an, die wie folgt verwendet werden kann, um das Image eines
Dogus zu scannen.

```groovy
EcoSystem ecoSystem = new EcoSystem(this, "gcloud-credentials-id", "ssh-credentials-id")
Trivy trivy = new Trivy(this, ecoSystem)

stage('Trivy scan') {
    trivy.scanDogu("/dogu", TrivyScanFormat.HTML, TrivyScanLevel.CRITICAL, TrivyScanStrategy.FAIL)
}
```

[trivy]: https://github.com/cloudogu/dogu-build-lib/blob/develop/docs/development/trivy_de.md

#### Manuelle Tests auf einer Test-Stage

In der manuellen Testphase sollte das Dogu auf einer Test-Stage getestet werden.
Dabei sollte die Test-Stage eine möglichst reale Umgebung abbilden.
Das Dogu soll mit dem Befehl `cesapp install` installiert werden.
Zusätzlich sollte es systematisch im Browser manuell getestet und als funktionstauglich eingestuft werden.
Dabei sollte der Fokus auf den Features liegen, welche in Produktion die größte Relevanz haben.

## Definition of Done

Dieser Abschnitt beschreibt mögliche Voraussetzungen für einen finalen Zustand einer Entwicklung.
Er schafft ein gemeinsames Verständnis für Entwickler, wann ein Feature fertig ist und kann verschiedenste Anforderungen
beinhalten.

Folgende Punkte beschreiben grundlegende Anforderungen, die als Basis einer Definition of Done dienen können.

### Issue-Tracking

- Für jede Änderung existiert ein Issue in einem Ticketsystem.
- Es beschreibt das zu entwickelnde Produktinkrement.
- Tickets aus öffentlichen Ticketsystem werden in internen Systemen verlinkt.

### Dockerfile

Anforderung für das Dockerfile sind [hier](interesting_aspects_de.md#container-images) zu finden.

### Dogu.json

Die Dogu.json muss folgende konfigurierbare Schlüssel enthalten:
- `container_config/memory_limit`
- `container_config/swap_limit`

### Dogu-Scripting

Anforderungen für Shell-Scripting beschreibt das Dokument ["Interessantes für Entwickelnde"](interesting_aspects_de.md#bash-skripte).

### Dogu-Funktionalitäten

- Backup & Restore-Fähigkeit betrachten
- Upgrade-Fähigkeit betrachten
    - Upgrade-Skripte müssen so gestaltet sein, dass aus allen alten Versionen des Dogus ein Upgrade möglich ist
      (sofern das die im Dogu eingesetzte Software nicht verhindert)
    - Ein Upgrade des Dogus von früherer Version ist zu testen
- Admin-Gruppe
    - Admin-Gruppe muss änderbar sein
    - Admin-Gruppe aus dem `etcd` wird verwendet und bei einem Wechsel der Admin-Gruppe reagiert das Dogu entsprechend
      darauf, dann wird der alten Admin-Gruppe das Admin-Recht entzogen. Siehe z. B. SonarQube und Nexus Repository
      Manager.
- Logging
    - Log-Ausgabe des Dogus auf Fehler checken
- Übernahme einer Änderung der FQDN
    - Eine FQDN-Änderung muss in einem Dogu sinnvoll verarbeitet werden.
    - Das Dogu muss nach einem Neustart ordnungsgemäß starten.
- Single Sign-on & Single Sign-out betrachten

Das Dokument ["Relevante Funktionalitäten"](../important/relevant_functionalities_de.md) geht auf diese Funktionalitäten tiefer ein.

### Dokumentation

Dogu-Dokumentation aktualisieren und prüfen, ob diese dem [Dokumentationsregelwerk](#cloudogu-dokumentationsregelwerk) 
entspricht.
- Bei Abweichungen die Struktur entsprechend aufarbeiten. Dabei bestehende Strukturen bestehen lassen, aus den dort
abgelegten Dateien auf den neuen `docs`-Ordner verlinken.
- Die alten Strukturen werden nach einer angemessenen Zeit (1 Jahr) gelöscht.

## Cloudogu Dokumentationsregelwerk

Das Regelwerk soll die Dokumentation für Cloudogu vereinheitlichen und Entscheidungen festhalten, wie bestimmte Themen
zu behandeln sind. Die folgenden Punkte bilden eine Basis und können durch weitere Aspekte wie z. B.
Produktschreibweisen,
einem Dokumentationsplan oder einem Styleguide erweitert werden.

### Aufbau einer Dokumentation

Die technische Dokumentation ist im `docs`-Ordner im jeweiligen Repository verortet.
Neben dem `docs`-Ordner enthält jedes Repository ein `CHANGELOG` sowie eine `README`.
Übergreifende Anleitungen („Guides“) müssen nicht im `docs`-Ordner abgelegt werden.

#### README.md

Siehe [README](interesting_aspects_de.md#readme)

#### CHANGELOG

Siehe [CHANGELOG](interesting_aspects_de.md#changelog)

#### Der docs-Ordner

Der `docs`-Ordner ist nach diesem beispielhaften Schema aufgebaut:

```
/docs  
├── getting_started_de.md  
├── getting_started_en.md  
├── /gui  
│ ├── backup_button_de.md  
│ ├── backup_button_en.md  
│ ├── restore_button_de.md  
│ └── restore_button_en.md  
├── /operations  
│ ├── show_graph_feature_de.md  
│ ├── show_graph_feature_en.md  
│ ├── get_function_de.md  
│ ├── get_function_en.md  
│ ├── remove_function_de.md  
│ ├── remove_function_en.md  
│ ├── push_command_de.md  
│ ├── push_command_en.md  
│ ├── pull_command_de.md  
│ ├── pull_command_en.md  
│ ├── groups_endpoint_de.md  
│ ├── groups_endpoint_en.md  
│ ├── users_endpoint_de.md  
│ └── users_endpoint_en.md  
└── /development  
  ├── how_to_create_an_image.md  
  └── how_to_build_a_plugin.md
```

(Die Dateinamen sind beispielhaft gewählt und keine Vorgabe).
Das Namensschema ist Name des Features plus ein Suffix mit der Sprachversion (de, en).
Einzelne Worte im Namen werden mit Unterstrichen verbunden (z. B. name_des_features_de oder name_des_features_en).

#### operations-Ordner

Der Ordner „operations“ ist der Kern der Dokumentation. Jedes einzelne Feature, jede Funktion, jeder Befehl wird in drei
Abschnitten beschrieben:

- Was? Welches Feature beschreibt der Abschnitt? Was ist der generelle Zweck?
- Wie? Detaillierte Beschreibung aller Optionen, Flags oder Einstellungen.
- Warum? Hintergründe, Einordnung im Gesamtsystem, Einschränkungen der Umsetzung.

Alle Inhalte im Ordner `operations` liegen in Deutsch und Englisch vor. Die Übersetzung kann automatisiert erfolgen.

#### gui-Ordner

Der Ordner `gui` ist optional für Repositories, die ein grafisches Nutzerinterface mitbringen.
In diesem werden alle Elemente des grafischen Interface erklärt.
Der Aufbau der Beschreibungen ist äquivalent zu `operations`.

#### development-Ordner

Der Ordner `development` ist optional für Repositories, die weitere Hinweise für die Entwicklung, z. B. von Plugins,
benötigen.
Die Beschreibungen in diesem Ordner werden nur auf Englisch verfasst.

#### getting_started-Dateien

Im `docs`-Ordner beschreiben die Dateien `getting_started_de.md` und `getting_started_en.md` die notwendigen Schritte,
um ein Repository lokal bereit zum Entwickeln zu machen.

#### Sprachversionen

Zielzustand ist, alle Bestandteile der Dokumentation (von `CHANGELOG`, `README` und development abgesehen) auf Deutsch
und Englisch zur Verfügung zu stellen. `CHANGELOG`, `README` und development werden nur in Englisch verfasst.

#### Bilder

Wenn Bilder in der Dokumentation verwendet werden, so sind diese in einem neuen Unterordner `figures` abzulegen.
Für `operations`, `gui` und `development` sollte jeweils ein eigener `figures`-Ordner erstellt werden.
Um Namenskonflikte für Bilder zu minimieren, sollte im `figures`-Ordner ein weiterer Unterordner angelegt werden,
welcher denselben Namen hat, wie die Markdown-Datei, für welche die Bilder verwendet werden sollen.
Dort können dann die Bilder abgelegt werden.

### Produktschreibweisen Cloudogu

| Thema           | Schreibweise                      | Kommentar                            |
|-----------------|-----------------------------------|--------------------------------------|
| CES             | Cloudogu EcoSystem                | -                                    |
| Dōgu            | Dogu                              | Ohne übergesetztes Makron            |
| SCM-Manager     | SCM-Manager                       | Mit Bindestrich                      |
| User Management | User Management                   | Nicht zusammengeschrieben            |
| Warp Menü       | Warp Menü (de) und Warp Menu (en) | Das CES Ausklappmenü am rechten Rand |

