# Dogu Artefakte

Ein Dogu ist eine betriebsbereite Anwendung innerhalb des Cloudogu Ecosystems (CES) – wie z. B. SCM-Manager, Jenkins oder SonarQube –, die nahtlos in die Plattform integriert ist. 

Mit dem Generationenwechsel ab Version 3 (V3) vollzieht das Ecosystem einen entscheidenden Schritt in Richtung cloud-nativer Standards: Das Helm-Chart wird zum zentralen Release-Artefakt und primären Format für die Paketierung, Versionierung und Bereitstellung eines Dogus. Durch die Ausbringung über Helm wird ein zusätzliches, plattformspezifisches Deskriptorformat (wie das bisherige `dogu.json`) überflüssig. Dies vereinfacht das Artefakt-Management und bettet die Anwendung nahtlos in das Kubernetes-Ökosystem ein. Container-Images bleiben separate Artefakte, auf die das Helm-Chart verweist.

## Das Chart als Dogu-Deskriptor

Im Helm-Chart dient die `Chart.yaml` als führendes Manifest zur Bereitstellung zentraler Metadaten. Die Datei regelt sowohl die Identifikation des Charts als auch dessen Paket-Abhängigkeiten. Darüber hinaus existieren weitere Manifeste, die für die Integration in das CES eine entscheidende Rolle spielen. Nachfolgend werden die wichtigsten Artefakte und deren Bedeutung für ein Dogu V3 detailliert beschrieben. Zu den Top-Level-Dateien im Helm-Chart gehören:

```
k8s/
└─ helm/
   ├─ templates/
   │  └─ ...
   ├─ Chart.yaml
   ├─ chart-patch-tpl.yaml
   ├─ dogu-upgrade.yaml
   ├─ dogu-values-metadata.yaml
   ├─ values.yaml
   └─ values.schema.json
```

### `Chart.yaml`

In der `Chart.yaml` befinden sich Metadaten, die ein Dogu beschreiben. Im Chart werden zwei Versionen unterschieden. 
Die Dogu-Version entspricht der Version des Charts (`version`), während die App-Version (`appVersion`) die Version der fachlichen Anwendung innerhalb des Dogus beschreibt. Beide Versionen sind voneinander unabhängig und dürfen sich unterscheiden. So könnte beispielsweise Redmine als Dogu in Version `45.7.0` veröffentlicht werden, während die enthaltene Redmine-Anwendung die App-Version `6.1.2` besitzt. 

Neben den [Helm-Standardfeldern](https://helm.sh/docs/topics/charts/#the-chartyaml-file) werden in der `Chart.yaml` plattformspezfische Annotations verwendet, die durch den Prefix `dogu.cloudogu.com/` gekennzeichnet sind. Dieser Prefix kennzeichnet Dogu-Metadaten im Helm-Chart und grenzt sie von allgemeinen Kubernetes- oder Helm-Annotations ab. Folgende plattformspezifische Annotations werden aktuell unterstützt:

- `api-version`: Die Dogu-API-Version, z.b. `v3`
- `display-name`: Der Anzeigename eines Dogus falls `name` Attribut der `Chart.yaml` nicht zur Anzeige verwendet werden soll
- `application.<name>`: Definiert weitere Anwendungen, die im Chart enthalten sind. `<name>` entspricht dem technischen Namen der enthaltenen Anwendung, wobei der Annotation-Wert eine nicht leere Versionsangabe als String enthält

Unbekannte Annotations mit dem Prefix `dogu.cloudogu.com/` sind zulässig, werden jedoch von plattformspezifischen Komponenten ignoriert, sofern sie nicht Teil der festgelegten Pflichtmetadaten sind. Als Pflichtfelder in der `Chart.yaml` gelten die Felder:

- `name`
- `version`
- `appVersion`
- `description`
- `annotations.dogu.cloudogu.com/api-version`

Ein Beispiel für eine gültige `Chart.yaml` für ein Dogu könnte wie folgt aussehen:

```yaml
apiVersion: v2
name: redmine
description: Redmine als Cloudogu EcoSystem Dogu
type: application
version: 45.7.0
appVersion: 6.1.2
home: https://cloudogu.com/ecosystem
sources:
  - https://github.com/cloudogu/redmine
icon: https://dogu.cloudogu.com/api/v3/dogus/official/redmine/icon.svg
keywords:
  - projectmanagement
  - issue
  - development
maintainers:
  - name: Cloudogu GmbH
    email: hello@cloudogu.com
annotations:
  dogu.cloudogu.com/api-version: v3
  dogu.cloudogu.com/display-name: Redmine
  dogu.cloudogu.com/application.redmine: "6.1.2"
  dogu.cloudogu.com/application.postgresql: "16.8"
```

### `values.yaml`

Die `values.yaml` dient als öffentliche Schnittstelle für die Konfiguration von Helm-Charts. Sie enthält Standardwerte, die für die Templates des Helm-Charts verwendet werden. Helm erlaubt es bei der Installation die Werte aus der `values.yaml` zu überschreiben, in dem es die übergebenen Werte mit den Standardwerten aus dem Helm-Chart zusammenführt. 

### `values.schema.json`

Für die `values.yaml` kann mithilfe der `values.schema.json` ein Schema in Form eines JSON-Schemas definiert werden, um die resultierenden Werte des Helm-Charts zu validieren. Da die Entwickler einer Anwendung am besten über den erforderlichen Input und dessen korrekte Struktur Bescheid wissen, sollte diese Schemadatei stets fester Bestandteil des Dogu-Helm-Charts sein. Der Dogu-Operator nutzt sie, um bei der Installation oder Aktualisierung eine automatisierte Validierung der Values durchzuführen und fehlerhafte Konfigurationen frühzeitig abzufangen.

### `dogu-values-metadata.yaml`

Für die Plattform können globale Konfigurationsparameter existieren, die nicht immer durch bestehende Werte aus der `values.yaml` eines Helm-Charts abgebildet werden können. So kann für die Plattform bspw. ein globales Log-Level gesetzt werden, welches für alle Dogus angewendet wird. Um diesem Umstand Rechnung zu tragen, wurde die `dogu-values-metadata.yaml` eingeführt. Durch sie können plattformspezifische Konfigurationen auf die Werte des Helm-Charts eines Dogus angewendet werden, indem in ihr das Mapping zwischen einzelnen Werten definiert wird. Plattformspezifische Konfigurationen werden in der Dogu-CR definiert und durch den Dogu-Operator angewendet. Folgend wird exemplarisch das Mapping des Log-Levels auf zwei Anwendungen eines Helm-Charts gezeigt:

```yaml
apiVersion: v1
metavalues:
  # Plattformspezifische Konfigurationwert
  mainLogLevel:
    keys:
      # ohne Mapping wird der Konfigurationswert ohne Änderung an die values.yaml durchgereicht 
      - path: controllerManager.env.loglevel
      # mit Mapping wird erst der Konfigurationswert abgebildet (z. B. panic -> error) und dann weitergereicht
      # Pfad zum Wert in der values.yaml für Hauptanwendung
      - path: app.env.loglevel
        mapping:
          DEBUG: debug
          INFO: info
          WARN: error
          ERROR: error
      # Pfad zum Wert in der values.yaml für zweite Anwendung
      - path: loglevel
        mapping:
          DEBUG: 1
          INFO: 2
          WARN: 3
          ERROR: 4      
```

**Dogu-CR, die den Konfigurationswert setzt:**

```yaml
apiVersion: k8s.cloudogu.com/v1
kind: Dogu
metadata:
  name: my-dogu
spec:
  name: official/my-dogu
  version: 1.2.3-4
  mappedValues:
    mainLogLevel: ERROR
```

Um plattformspezifische Konfigurationen treffen zu können, sollte ein Dogu-Helm-Chart die `dogu-values-metadata.yaml` stets mit ausliefern. Für konfigurationslose Dogus ist diese Datei optional. 

### `dogu-upgrade.yaml`

Ein wesentlicher Bestandteil des CES ist die Ausführung von Dogu-Upgrades, die unter normalen Umständen manuelle Aktionen des Administrators benötigen. Im CES sollen die Upgrade-Prozesse automatisiert durch den Dogu-Operator ausgeführt werden. Ein wichtiger Bestandteil hierfür sind Migrationspfade zwischen einzelnen Dogu-Versionen. Diese werden in der `dogu-upgrade.yaml` abgebildet und während eines Upgrades durch den Dogu-Operator validiert. In der `dogu-upgrade.yaml` werden grundsätzlich erlaubte Versionssprünge definiert:

```yaml
upgrades:
  # Upgrades innerhalb des Pfades sind ohne Migration erlaubt.
  - from: ">=1.0.0 <=1.7.0" 
    to: "1.8.0"
  # Upgrade bringt Migration mit.
  - from: ">=1.8.0 <2.0.0"
    to: "2.0.0"
    isMigration: true
    helmTimeout: 15m
    scaleSelectors: 
    - matchLabels:
        dogu.name: nexus
```

### `chart-patch-tpl.yaml`

Eine besondere Herausforderung für den Betrieb der Plattform sind sogenannte Air-Gapped-Umgebungen, die isoliert von anderen Umgebungen – insbesondere dem Internet – betrieben werden. Für solche Umgebungen werden sowohl die benötigten Helm-Charts als auch Container Images in eine interne OCI-Registry innerhalb der isolierten Umgebung gespiegelt, wodurch sich die Bezugsquellen der Images aus dem Helm-Chart ändern. Mit Hilfe der `chart-patch-tpl.yaml` können die Referenzen der Images durch ein Mirroring-Tool für die Zielumgebung aufgelöst und überschrieben werden.

**`chart-patch-tpl.yaml` am Beispiel des Nexus-Dogus**

```yaml
apiVersion: v1
values:
  images:
    nexus: registry.cloudogu.com/official/nexus:3.86.2-6
    nexusSaManager: registry.cloudogu.com/k8s/service-account-producer-sidecar:0.1.2
    postgresql: docker.io/library/postgres:14.18
patches:
  values.yaml:
    nexus:
      image:
        registry: "{{ registryFrom .images.nexus }}"
        repository: "{{ repositoryFrom .images.nexus }}"
        tag: "{{ tagFrom .images.nexus }}"
      saManager:
        image:
          registry: "{{ registryFrom .images.nexusSaManager }}"
          repository: "{{ repositoryFrom .images.nexusSaManager }}"
          tag: "{{ tagFrom .images.nexusSaManager }}"
    postgresql:
      image:
        registry: "{{ registryFrom .images.postgresql }}"
        repository: "{{ repositoryFrom .images.postgresql }}"
        tag: "{{ tagFrom .images.postgresql }}"
```

Die `chart-patch-tpl.yaml` sollte stets Bestandteil des Dogu-Helm-Charts sein, um die im Helm-Chart referenzierten Container Images in eine Air-Gapped-Umgebung spiegeln zu können.


## Templates und die Anbindung an die Plattform

In den Templates des Helm-Charts werden alle Kubernetes-Ressourcen zur Verfügung gestellt, um die Anwendung auf der Plattform ausführbar zu machen. Zu diesen Ressourcen zählen auch Custom Resources Definitions (CRDs), die von Cloudogu bereitgestellt werden und als API für die Plattform dienen. In diesem Abschnitt werden die einzelnen CRDs genauer beschrieben.

### AuthRegistration

Das CES bietet seinen Nutzer die Möglichkeit des Single Sign-On (SSO). Damit ein Dogu den SSO nutzen kann, muss es sich mithilfe der [`AuthRegistration-CR`](https://github.com/cloudogu/k8s-auth-registration-lib/blob/main/docs/operations/auth_registration_de.md) beim Identity-Provider (IdP) der Plattform registrieren. Aktuell werden die Authentifizierungsprotokolle `CAS`, `OAUTH` und `OIDC` unterstützt. Mit der Registration beim IdP werden zugleich Credentials in einem Secret bereitgestellt, die von dem Dogu für Anfragen gegen den IdP genutzt werden können. 

Eine `AuthRegistration` provisioniert nur die serverseitige Integration beim IdP. Die Anwendung muss das ausgewählte Authentifizierungsprotokoll weiterhin selbst implementieren.

### WarpMenuEntry

Für die zentrale Navigation auf der Plattform wird das Warp-Menü verwendet. Wenn ein Dogu im Warp-Menü angezeigt werden soll, muss für jeden internen, für Anwender:innen sichtbaren Einstiegspunkt eine [`WarpMenuEntry-CR`](https://github.com/cloudogu/k8s-warp-menu-entry-lib/blob/main/docs/operations/warp_menu_entry_de.md) bereitgestellt werden. Je nach Bedarf kann ein Dogu keinen, einen oder mehrere Einträge definieren.

Jeder Eintrag enthält einen deutschen und englischen Anzeigenamen, eine Kategorie sowie einen relativen Pfad zum Dogu. Das Warp-Menü macht die Anwendung nicht grundsätzlich erreichbar. Der relative Pfad muss zu einer unabhängig funktionierenden HTTP-Exposition passen.

### Exposition

Soll ein Dogu von außen erreichbar sein, muss hierfür eine oder mehrere [`Exposition-CR`](https://github.com/cloudogu/k8s-exposition-lib/blob/main/docs/operations/exposition_cr_de.md) deklariert werden. Die `Exposition` definiert, wie ein Service von außerhalb der Plattform erreichbar gemacht wird. Sie unterstützt HTTP-Routen (Layer 7) sowie rohe TCP- und UDP-Ports (Layer 4). Für HTTP-Routen können zusätzlich Path-Rewrites definiert werden. Auf Layer 4 Ebene kann es zu Kollisionen kommen, wenn zwei oder mehrere `Expositions` den gleichen Port für das gleiche Protokoll anfragen. In diesem Fall wird keine der betroffenen `Expositions` angewendet, solang der Konflikt besteht. Der Administrator wird hierüber im Status der CR informiert.  

### ServiceAccountRequest / ServiceAccountProducer

Im CES ist es möglich, dass verschiedene Dogus/Components miteinander interagieren. Hierfür muss für ein zugreifendes Dogu/Component (Consumer) ein entsprechender ServiceAccount beim Ziel-Dogu/-Component (Producer) erstellt werden. Diese Service-Account-Beziehung ist also frei wählbar und ist nicht nur auf Dogus unter sich fest gelegt. Dieser Mechanismus erlaubt auch Ringabhängigkeiten zwischen Producer und Consumer.

Die ServiceAccount-Erstellung lässt sich deklarativ über die CRDs [`ServiceAccountRequest`](https://github.com/cloudogu/k8s-serviceaccount-lib/blob/main/docs/operations/serviceaccountrequest_cr_de.md) und [`ServiceAccountProducer`](https://github.com/cloudogu/k8s-serviceaccount-lib/blob/main/docs/operations/serviceaccountproducer_cr_de.md) steuern.

**`ServiceAccountProducer`**

Bietet ein Producer eine Schnittstelle an, die von anderen Dogus genutzt werden kann, muss es eine `ServiceAccountProducer-CR` bereitstellen. Die `ServiceAccountProducer`-CR definiert, wie Service-Accounts für Consumer erstellt werden und welche Parameter unterstützt werden. Ferner beschreibt die CR die Struktur, wie die Werte nach dem Erstellen eines Service-Accounts vom Producer zurückgegeben werden. Jeder zurückgegebene Wert wird als Schlüssel in das vom anfragenden Consumer referenzierte Secret geschrieben.

**`ServiceAccountRequest`**

Benötigt ein Consumer einen Service-Account bei einem Producer, muss es diesen über die `ServiceAccountRequest-CR` anfordern, indem ein Producer benannt wird und optional Parameter übergeben werden. Die resultieren Credentials des Requests werden durch den Operator in ein referenziertes Kubernetes-Secret des konsumierenden Dogus geschrieben. Ist keine Referenz zu einem Secret im `ServiceAccountRequest` deklariert, wird ein Secret mit dem Namen der ServiceAccountRequest-Ressource erstellt. Die CR bildet das Gegenstück zum `ServiceAccountProducer`. 

## Abgrenzung zu Dogu V2

In Dogu V2 wird ein Dogu mithilfe des plattformspezifischen Deskriptorformats `dogu.json` beschrieben, in dem zugleich die Konfiguration sowie die Nutzung der Plattform-API beschrieben ist. In Dogu V3 findet ein Wechsel hin zum [HELM](https://helm.sh/de/), dem De-facto-Standard von Kubernetes zur Paketverwaltung, statt. Viele Inhalte der bisherigen `dogu.json` werden dadurch entweder als Kubernetes-Ressourcen, oder als gesonderte Dogu-spezifische Dateien in das Helm-Chart verschoben. Die folgende Tabelle zeigt, wie die einzelnen Felder aus der `dogu.json` im neuen Dogu-Helm-Chart abgebildet werden.

| Feld in `dogu.json`                                                | Behandlung in Dogu V3                                                                  |
|--------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| `Name`, `Version`, `DisplayName`, `Description`, `URL`, `Logo`     | Metadaten des `Chart.yaml`                                                             |
| `Image`                                                            | Container-Image-Referenzen in `chart-patch-tpl.yaml`                                   |
| `Dependencies`                                                     | Zusätzliche Workloads oder exteren Abhängigkeiten via ServiceAccount-CRs               |
| `ServiceAccounts`                                                  | ServiceAccount-CRs                                                                     |
| `Volumes`                                                          | Kubernetes-PVCs und Volume-Definitionen                                                |
| `ExposedCommands`                                                  | Entfällt als API                                                                       |
| `ExposedPorts`                                                     | Exposition-CRs                                                                         |
| `Tags`, `Category`                                                 | Katalog-Metadaten in `Chart.yaml`; Warp-Menü über WarpMenu-CR                          |
| `Configuration`                                                    | `values.yaml`, `dogu-values-metadata.yaml` und `values.schema.json`                    |
| `HealthChecks`                                                     | Kubernetes-Probes in Pod-Spezifikation                                                 |
| `Capabilities`, `EnvironmentVariables`, `Properties`, `Privileged` | entfallen oder werden als Kubernetes-/Helm-Konzepte abgebildet                         |
