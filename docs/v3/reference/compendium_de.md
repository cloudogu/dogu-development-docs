# Kompendium der Dogu-V3-Artefakte

Dieses Kompendium gibt einen Überblick über die Artefakte und APIs von Dogu V3. Eine Einführung in ihr Zusammenspiel finden Sie unter [Dogu-V3-Artefakte verstehen](../concepts/artifacts_de.md).

## Chart-Artefakte und APIs

| Artefakt / API | Zweck | Wann benötigt? | Verantwortung | Referenzdokumentation |
| --- | --- | --- | --- | --- |
| Helm-Chart | Führendes Dogu-Paket; enthält Metadaten, Standardwerte, Templates und Begleitdateien | Immer | Dogu-Entwickler:in | [Helm-Chart-Format](https://helm.sh/docs/topics/charts/) |
| `Chart.yaml` | Dogu-Identität und Discovery-Metadaten; `version` ist die Dogu-Version, `appVersion` die Version der verpackten Anwendung | Immer | Dogu-Entwickler:in; konsumiert von Dogu-Tooling | [Helm-`Chart.yaml`](https://helm.sh/docs/topics/charts/#the-chartyaml-file) |
| `values.yaml` | Sichere Standardwerte und öffentliche Konfigurationsschnittstelle | Immer, gemäß Helm-Chart-Konvention | Dogu-Entwickler:in | [Helm-Values](https://helm.sh/docs/chart_template_guide/values_files/) |
| `values.schema.json` | Validiert die zusammengeführten Values | Wenn das Dogu konfigurierbare Werte anbietet | Dogu-Entwickler:in | [Helm-Schemadateien](https://helm.sh/docs/topics/charts/#schema-files) |
| `dogu-values-metadata.yaml` | Ordnet CES-Konfigurationsschlüssel wie einen globalen Log-Level Chart-Values zu | Für CES-weite Konfigurationszuordnungen; bei einem konfigurationslosen Dogu optional | Dogu-Entwickler:in; konsumiert vom Dogu Operator | — |
| `templates/` | Rendert Workloads, Services, PVCs, Probes und optionale CES-Integrationsressourcen | Immer | Dogu-Entwickler:in | [Helm-Templates](https://helm.sh/docs/chart_template_guide/) und Verträge der Kubernetes-/CRD-Verantwortlichen |
| `chart-patch-tpl.yaml` | Listet alle vom Chart referenzierten Container-Images für die Dogu Registry und Spiegelungswerkzeuge auf beziehungsweise löst sie auf | Wenn das Chart Container-Images referenziert, also normalerweise bei jedem Dogu | Dogu-Entwickler:in; konsumiert von der Dogu Registry und Spiegelungswerkzeugen | — |
| `dogu-upgrade.yaml` | Beschreibt gültige Übergänge zwischen Dogu-Versionen und optionale Parameter zur Upgrade-Koordination | Die akzeptierten ADRs legen noch nicht fest, wann die Datei vorhanden sein muss | Dogu-Entwickler:in; konsumiert vom Dogu Operator | — |
| Container-Images | Stellen die Anwendung sowie referenzierte Sidecar- und Init-Container bereit | Für jeden vom Chart gerenderten Container | Image-Produzent; Referenzen durch Dogu-Entwickler:in | [Kubernetes-Images](https://kubernetes.io/docs/concepts/containers/images/) |
| `AuthRegistration` | Deklariert bei Teilnahme an CES-Authentifizierung die Registrierung | Wenn das Dogu die CES-Authentifizierung nutzt | k8s-auth-registration-lib verantwortet die API; Deklaration durch Dogu-Entwickler:in | [AuthRegistration-v1-API](https://github.com/cloudogu/k8s-auth-registration-lib/tree/develop/api/v1) |
| `Exposition` | Verbindet bei Bedarf einen Chart-Service mit CES-bereitgestelltem externen Zugriff | Wenn das Dogu extern erreichbar sein soll | k8s-exposition-lib verantwortet die API; Deklaration durch Dogu-Entwickler:in | [Exposition-v1-API](https://github.com/cloudogu/k8s-exposition-lib/tree/develop/api/v1) |
| `ServiceAccountRequest` | Fordert bei Bedarf technische Credentials von einem Producer an | Wenn das Dogu technische Credentials eines anderen CES-Dienstes benötigt | k8s-serviceaccount-lib verantwortet die API; Deklaration durch Consumer | [Service-Account-v2-API](https://github.com/cloudogu/k8s-serviceaccount-lib/tree/develop/api/v2) |
| `ServiceAccountProducer` | Deklariert, dass das Dogu technische Accounts anbietet | Wenn das Dogu technische Accounts bereitstellt | k8s-serviceaccount-lib verantwortet die API; Deklaration durch Producer | [Service-Account-v2-API](https://github.com/cloudogu/k8s-serviceaccount-lib/tree/develop/api/v2) |
| `WarpMenuEntry` | Deklariert bei einem sichtbaren CES-Pfad einen Eintrag im gemeinsamen Menü | Wenn das Dogu einen sichtbaren CES-Pfad bereitstellt | k8s-warp-menu-entry-lib verantwortet die API; Deklaration durch Dogu-Entwickler:in | [WarpMenuEntry-v1-API](https://github.com/cloudogu/k8s-warp-menu-entry-lib/tree/develop/api/v1) |
| Dogu-Registry-Daten | Verbinden technische Chart-Metadaten mit außerhalb des Charts gepflegten Berechtigungs-, Marketing- oder Vertriebsdaten | Für Veröffentlichung, Discovery und Berechtigungsprüfung über die Dogu Registry | Cloudogu | — |

## Vollständige Zuordnung der V2-`dogu.json`

Die folgende Tabelle führt jedes Feld der V2-`dogu.json` und, soweit vorhanden, sein akzeptiertes V3-Ziel auf. Die Zuordnungen sind keine automatische Konvertierung und erhalten nicht zwangsläufig jedes V2-Verhalten. Wo die akzeptierte Architektur kein allgemeines V3-Ziel definiert, benennt die Tabelle diese Grenze ausdrücklich.

| V2-Feld | V3-Ziel | Zuordnung und Status |
| --- | --- | --- |
| `Name` | `Chart.yaml` + Registry-Namespace | Chart-`name` ist der einfache technische Name; der qualifizierte Namespace stammt aus dem Registry-Kontext. |
| `Version` | `Chart.yaml` | Wird zur Chart-`version`, der Dogu-Version. `appVersion` bezeichnet getrennt die Anwendung. |
| `PublishedAt` | Dogu-Registry-v3-API | Ist im akzeptierten API-Entwurf als Veröffentlichungsinformation enthalten. Quelle und Pflegeprozess sind noch nicht festgelegt. |
| `DisplayName` | `Chart.yaml` | Dogu-Annotation `dogu.cloudogu.com/display-name`; in den akzeptierten Zielmetadaten optional. |
| `Description` | `Chart.yaml` | Standardfeld `description`; in den akzeptierten Zielmetadaten verpflichtend. |
| `Category` | Kein konkretes V3-Feld festgelegt | Die akzeptierte Architektur behandelt die Kategorie als Katalogmetadatum, definiert aber weder ein `Chart.yaml`-Feld noch eine externe Quelle. Sie ist unabhängig von `WarpMenuEntry`. |
| `Tags` | `Chart.yaml` | Allgemeine Suchbegriffe werden als Helm-`keywords` gepflegt. Menüeinträge werden separat mit `WarpMenuEntry` definiert. |
| `Logo` | `Chart.yaml` | Die URL zum Dogu-Logo wird im Helm-Feld `icon` hinterlegt. |
| `URL` | `Chart.yaml` | Die Projekt- oder Herstellerwebsite wird im Helm-Feld `home` hinterlegt. |
| `Image` | Begleitdatei + Kubernetes-Workloads | Image-Referenzen werden in den Container-Specs definiert. `chart-patch-tpl.yaml` macht sie zusätzlich für Plattformwerkzeuge auffindbar. |
| `ExposedPorts` | Helm/Kubernetes + CES-CR | Kubernetes-Service plus `Exposition`, wenn externer Zugriff benötigt wird. |
| `ExposedCommands` | Entfällt / zweckgebundene V3-Mechanismen | Eine generische V3-`ExposedCommands`-API existiert nicht. Upgrade-Migrationen werden mit Init-Containern oder Helm-Hook-Jobs umgesetzt. Andere Lifecycle- oder benutzerdefinierte Befehle müssen separat betrachtet werden. |
| `Volumes` | Helm-/Kubernetes-Ressourcen | PVCs, Volumes und Volume-Mounts. Backup und Retention separat deklarieren/dokumentieren; keine automatische Feldkonvertierung. |
| `HealthCheck` | Helm-/Kubernetes-Ressourcen | Veralteter einzelner V2-Check. TCP- oder HTTP-Checks werden einer passenden Kubernetes-Probe zugeordnet, wenn das Signal gleichwertig ist; für den V2-Check vom Typ `state` definiert die akzeptierte Architektur kein direktes V3-Äquivalent. |
| `HealthChecks` | Helm-/Kubernetes-Ressourcen | Geeignete Checks werden entsprechend ihrem Zweck als Startup-, Readiness- oder Liveness-Probes modelliert. Dadurch bleiben nicht automatisch jeder V2-Check-Typ und jedes Consumer-Verhalten erhalten. |
| `ServiceAccounts` | CES-CRs | `ServiceAccountRequest` und bei angebotenen Accounts `ServiceAccountProducer`. |
| `Privileged` | Entfällt | Es gibt keinen direkten V3-Ersatz. Das V2-Feld band den Docker-Socket ein; der Kubernetes-Privileged-Modus oder ein Security Context ist kein gleichwertiger Ersatz. Im Chart werden nur die tatsächlich benötigten Pod- und Container-Berechtigungen definiert. |
| `Security` | Helm-/Kubernetes-Ressourcen | Unterstützte Einstellungen werden über Pod- oder Container-Sicherheitskontexte und Capabilities abgebildet. Die akzeptierte Architektur legt keine gemeinsame V3-Sicherheitsbaseline fest. |
| `Configuration` | Chart-Values, Schema und Begleitmetadaten | `values.yaml`, `values.schema.json` und für CES-Zuordnungen `dogu-values-metadata.yaml`. |
| `Properties` | Entfällt | Für das generische V2-Feld gibt es keinen allgemeinen V3-Ersatz. Ein konkretes Helm-, Kubernetes- oder CES-Konzept wird nur verwendet, wenn dessen API das benötigte Verhalten definiert. |
| `EnvironmentVariables` | Helm-/Kubernetes-Ressourcen | Explizite Container-Felder `env` und `envFrom`, je nach Inhalt aus Values, ConfigMaps oder Secrets gespeist. |
| `Dependencies` | Helm-/Kubernetes-Ressourcen oder CES-Service-Account-CRs | Service-Abhängigkeiten werden, soweit passend, mit `ServiceAccountRequest` abgebildet; andere Anforderungen werden explizit im Chart modelliert. Die akzeptierte Architektur definiert keinen generischen Ersatz für sämtliche V2-Client-, Paket- und Versionsprüfungen. |
| `OptionalDependencies` | Optionaler `ServiceAccountRequest` oder zweckgebundene Chart-Ressourcen | Für optionale Service-Account-Abhängigkeiten wird ein optionaler Request verwendet. Die akzeptierte Architektur definiert keinen generischen Vertrag, der sämtliche V2-Semantiken für optionale Abhängigkeiten und Versionsprüfungen erhält. |

## Zugehöriges Konzept

Zurück zu [Dogu-V3-Artefakte verstehen](../concepts/artifacts_de.md) oder weiter zur [Multinode-Laufzeitumgebung](../concepts/multinode-environment_de.md).
