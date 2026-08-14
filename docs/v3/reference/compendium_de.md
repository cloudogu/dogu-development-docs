# Kompendium der Dogu-V3-Artefakte

Dieses Kompendium gibt einen Überblick über die Artefakte und APIs - den CES-Integrationsressourcen - von Dogu V3. Eine Einführung in ihr Zusammenspiel finden Sie unter [Dogu-Artefakte](../concepts/artifacts_de.md).

## Chart-Artefakte und APIs

| Artefakt / API | Zweck | Referenzdokumentation |
| --- | --- | --- |
| Helm-Chart | Führendes Dogu-Paket; enthält Metadaten, Standardwerte, Templates und Begleitdateien | [Helm-Chart-Format](https://helm.sh/docs/topics/charts/) |
| `Chart.yaml` | Dogu-Identität und Discovery-Metadaten; `version` ist die Dogu-Version, `appVersion` die Version der verpackten Anwendung | [Helm-`Chart.yaml`](https://helm.sh/docs/topics/charts/#the-chartyaml-file) |
| `values.yaml` | Standardwerte und öffentliche Konfigurationsschnittstelle | [Helm-Values](https://helm.sh/docs/chart_template_guide/values_files/) |
| `values.schema.json` | Validiert die zusammengeführten Values | [Helm-Schemadateien](https://helm.sh/docs/topics/charts/#schema-files) |
| `dogu-values-metadata.yaml` | Ordnet CES-Konfigurationsschlüssel wie einen globalen Log-Level Chart-Values zu | [dogu-values-metadata.yaml](../concepts/artifacts_de.md#dogu-values-metadatayaml) |
| `templates/` | Rendert Workloads, Services, PVCs, Probes und optionale CES-Integrationsressourcen | [Helm-Templates](https://helm.sh/docs/chart_template_guide/) und Verträge der Kubernetes-/CRD-Verantwortlichen |
| `chart-patch-tpl.yaml` | Listet alle vom Chart referenzierten Container-Images auf und löst deren Referenzen über ein Spiegelungswerkzeug für Air-Gapped-Umgebungen auf | [chart-patch-tpl.yaml](../concepts/artifacts_de.md#chart-patch-tplyaml) |
| `dogu-upgrade.yaml` | Beschreibt gültige Übergänge zwischen Dogu-Versionen und optionale Parameter zur Upgrade-Koordination | [dogu-upgrade.yaml](../concepts/artifacts_de.md#dogu-upgradeyaml) |
| Container-Images | Stellen die Anwendung sowie referenzierte Sidecar- und Init-Container bereit | [Kubernetes-Images](https://kubernetes.io/docs/concepts/containers/images/) |
| `AuthRegistration` | Deklariert bei Teilnahme an CES-Authentifizierung die Registrierung | [AuthRegistration](../concepts/artifacts_de.md#authregistration) |
| `Exposition` | Verbindet bei Bedarf einen Chart-Service mit CES-bereitgestelltem externen Zugriff | [Exposition](../concepts/artifacts_de.md#exposition) |
| `ServiceAccountRequest` | Fordert bei Bedarf technische Credentials von einem Producer an | [ServiceAccountRequest](../concepts/artifacts_de.md#serviceaccountrequest--serviceaccountproducer) |
| `ServiceAccountProducer` | Deklariert, dass das Dogu technische Accounts anbietet | [ServiceAccountProducer](../concepts/artifacts_de.md#serviceaccountrequest--serviceaccountproducer) |
| `WarpMenuEntry` | Deklariert bei einem sichtbaren CES-Pfad einen Eintrag im Warp-Menü | [WarpMenuEntry](../concepts/artifacts_de.md#warpmenuentry) |
| Dogu-Registry-Daten | Verbinden technische Chart-Metadaten mit außerhalb des Charts gepflegten Berechtigungs-, Marketing- oder Vertriebsdaten | — |
