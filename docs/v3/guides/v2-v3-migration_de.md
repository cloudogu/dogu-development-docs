# Ein V2-Dogu als migrierbares V3-Chart bereitstellen

Ein V3-Dogu wird als Helm-Chart ausgeliefert. Das Chart beschreibt die Ressourcen der Anwendung und enthält die Jobs, die bestehende V2-Daten übernehmen.
Der Dogu-Operator steuert die Migration und stellt den Jobs die bisherigen Volumes und Quell-Credentials bereit.

Für Dogu-Entwickler:innen ändert sich vor allem:

- **Ressourcen:** Die Inhalte der `dogu.json` werden in Chart-Dateien und Kubernetes-Ressourcen überführt.
- **Konfiguration:** Helm-Values werden zur neuen Source of Truth. Die bisherige V2-Dogu-Konfiguration wird nicht automatisch übernommen.
- **Daten:** Chart-eigene Migrations-Jobs übertragen persistente Dateien und bei Bedarf Daten aus bisher geteilten Services.

Jeder Abschnitt beschreibt zunächst die Anforderungen an das eigene Dogu und zeigt anschließend ein Beispiel aus dem [Nexus-Chart][nexus-chart].
Die Durchführung einer Migration durch Betreiber beschreibt die [Betriebsanleitung des Dogu-Operators][operations].

## Inhalt

- [Voraussetzungen und Einschränkungen](#voraussetzungen-und-einschränkungen): Ausgangspunkt, Downtime, Speicherbedarf und Rollback
- [V3-Chart vorbereiten](#v3-chart-vorbereiten): Artefakte, Workloads und CES-Integrationen
- [Konfiguration bereitstellen](#konfiguration-bereitstellen): Helm-Values und Abgrenzung zu persistenten Daten
- [Migrations-Jobs erstellen](#migrations-jobs-erstellen): Schalter, Labels, Quellressourcen und Nexus-Beispiele
- [Chart prüfen](#chart-prüfen): Templates für Neuinstallation und Migration rendern
- [Fehleranalyse](#fehleranalyse): Conditions, Events und typische Chart-Fehler

## Voraussetzungen und Einschränkungen

### Voraussetzungen

- Das bestehende V2-Dogu und seine `dogu.json` sind verfügbar.
- Aufbau und Templates eines Helm-Charts sind bekannt, siehe [Vom Helm-Chart zum Dogu V3](../getting-started/quickstart_de.md).
- Die persistenten Daten des Dogus sind bekannt: Dateien im eigenen Volume sowie Daten in geteilten Services, etwa PostgreSQL.

Die spätere Migration setzt ein vom Dogu-Operator verwaltetes Dogu voraus. Eine direkte Helm-Installation führt sie nicht aus.

### Einschränkungen

- **Downtime:** Die Anwendung bleibt während der Datenübernahme gestoppt.
- **Speicherbedarf:** Quell- und Zielvolumes benötigen während der Migration gleichzeitig Speicher.
- **Kein automatisierter Rollback auf V2:** Bei fehlgeschlagener Migration bleibt die Anwendung gestoppt. Eine erforderliche Wiederherstellung erfolgt über ein Backup.
- **Einmaliger Übergang:** V3→V3-Upgrades sind ein eigener Prozess. Die Migrationsunterstützung bleibt im Chart erhalten, solange noch V2-Installationen migriert werden müssen.

**Beispiel: Nexus**

Für Nexus werden sowohl die Dateien im eigenen Volume als auch die Daten in der bisher geteilten PostgreSQL-Instanz benötigt.
Beide Datenbestände müssen bei der Vorbereitung des Charts berücksichtigt werden.

## V3-Chart vorbereiten

Die bisherige `dogu.json` dient als Ausgangspunkt für das Chart.
Die vollständige [Zuordnung der dogu.json-Felder zu V3-Artefakten](../concepts/artifacts_de.md#abgrenzung-zu-dogu-v2) beschreibt, wohin die einzelnen Inhalte übertragen werden.

### Chart-Dateien und Ressourcen

| Datei oder Ressource | Aufgabe |
| --- | --- |
| `Chart.yaml` | Name, Beschreibung und Versionen des Dogus. Die Annotation `dogu.cloudogu.com/api-version: v3` kennzeichnet die V3-Dogu-API. |
| Workload-Templates | Deployment oder StatefulSet mit Containern, Probes und Ressourcenbedarf. |
| Service-Templates | Netzwerkzugänge zu den Workloads innerhalb des Clusters. |
| PVCs und Volume-Definitionen | Persistenter Speicher und seine Einbindung in die Container. |
| `mirror-patch-tpl.yaml` | Image-Referenzen und ihre Zielschlüssel in den Values für das Spiegeln in andere Registries. |

In `Chart.yaml` bezeichnet `version` die Chart-Version und `appVersion` die Anwendungsversion.
Die dortige `apiVersion: v2` bezeichnet das Helm-Chart-Format; die Dogu-API-Version steht in der Annotation.
Die Dateien für die Konfiguration sind im [folgenden Abschnitt](#konfiguration-bereitstellen) beschrieben.
Für das eigene Dogu ist festzulegen, welche bisher geteilten Services künftig zum Chart gehören und welche externen Integrationen bestehen bleiben.

**Beispiel: Nexus**

Nexus verwendet einen StatefulSet und bringt eine eigene PostgreSQL-Instanz mit.
Die [Nexus-Chart.yaml][nexus-chart] enthält die Metadaten und die V3-Kennzeichnung; die Templates definieren die Workloads und ihre PVCs.

### CES-Integrationen

Die benötigten Integrationen werden als Ressourcen in den Chart-Templates deklariert:

| Ressource | Aufgabe |
| --- | --- |
| `AuthRegistration` | Registriert die Anwendung beim Identity-Provider. |
| `Exposition` | Macht einen Service über die Plattform von außen erreichbar. |
| `WarpMenuEntry` | Fügt einen Einstiegspunkt der Anwendung zum Warp-Menü hinzu. |
| `ServiceAccountRequest` | Fordert einen technischen Account bei einem externen Producer an. |
| `ServiceAccountProducer` | Beschreibt, wie das Dogu selbst Accounts für andere Anwendungen bereitstellt. |

Die Dokumentation der jeweiligen APIs steht unter [Templates und Plattformanbindung](../concepts/artifacts_de.md#templates-und-die-anbindung-an-die-plattform).

**Beispiel: Nexus**

Nexus registriert seine CAS-Anbindung über eine `AuthRegistration`.
Ein `ServiceAccountProducer` beschreibt die Accounts, die Nexus anderen Anwendungen bereitstellt.

### Weiterhin benötigte Service-Accounts

Benötigt das V3-Dogu dauerhaft einen Account bei einem externen Producer, gehört ein `ServiceAccountRequest` in das Chart.
Er wird auch außerhalb einer Migration bereitgestellt. Vorhandene Accounts werden durch die Operatoren übernommen.

**Beispiel**

```yaml
apiVersion: k8s.cloudogu.com/v2
kind: ServiceAccountRequest
metadata:
  name: example-external-account
spec:
  consumer: example
  consumerType: Dogu
  producer: example-producer
  secretRef:
    name: example-external-credentials
```

Der Producer muss existieren. Welche zusätzlichen Parameter erforderlich sind, legt der Producer fest.
Das Dogu verwendet die Credentials aus `example-external-credentials`; Details beschreibt die [ServiceAccountRequest-Referenz][sar].

**Beispiel: Nexus**

Das Nexus-Chart enthält keinen `ServiceAccountRequest`; für seine chart-eigene Datenbank benötigt es keinen externen Account.
Der vorübergehende Zugriff auf die alte PostgreSQL-Instanz erfolgt über das [Migration-Secret](#quellressourcen).
Nexus stellt selbst Accounts über einen `ServiceAccountProducer` bereit; das ist die Gegenrichtung eines Requests.

## Konfiguration bereitstellen

Die `values.yaml` ist die neue Source of Truth für die Konfiguration des Dogus.
Sie enthält die Defaults, die bei der Installation durch bereitgestellte Helm-Values überschrieben werden können.
Aus den zusammengeführten Values erzeugen die Templates die Konfiguration für die Anwendung.

| Datei | Aufgabe |
| --- | --- |
| `values.yaml` | Konfigurierbare Werte und ihre Defaults. |
| `values.schema.json` | Typen, Pflichtwerte und zulässige Werte für die Validierung. |
| `dogu-values-metadata.yaml` | Zuordnung plattformspezifischer Konfigurationswerte zu Chart-Values. |

Die bisherige V2-Dogu-Konfiguration wird nicht automatisch in diese Values übertragen.
Die gewünschten Einstellungen müssen über die neue Konfigurationsschnittstelle bereitgestellt werden.
Values, Schema und Verwendung im Workload müssen zusammenpassen.

**Beispiel: Nexus**

Nexus bildet beispielsweise `logging/root` in seiner [values.yaml][nexus-values] so ab:

```yaml
nexus:
  configuration:
    normal:
      logging:
        root: "WARN"
```

Das [ConfigMap-Template][nexus-config] rendert `nexus.configuration.normal` als Konfiguration für die Anwendung.

### Konfiguration und persistente Daten

Die in der `dogu.json` beschriebene Dogu-Konfiguration wird nicht automatisch migriert. Die gewünschten Einstellungen müssen über Helm-Values bereitgestellt werden.
Persistente Konfiguration im Dogu-PVC wird dagegen zusammen mit den übrigen persistenten Daten übernommen.

| Bereich | Behandlung |
| --- | --- |
| Dogu-Konfiguration aus der `dogu.json` | Wird nicht migriert; gewünschte Einstellungen über Helm-Values bereitstellen. |
| Persistente Konfiguration und Daten im Dogu-PVC | Über den Migration-PVC in die Ziel-PVCs übernehmen. |
| Daten in einem geteilten Service | Mit einem eigenen Job in die Zielinstanz übertragen. |

Für das eigene Dogu müssen die benötigten Dateien und ihre Zielverzeichnisse festgelegt werden.
Daten in geteilten Services sind zusätzlich zu berücksichtigen; sie liegen außerhalb des eigenen PVs.

**Beispiel: Nexus**

Nexus speichert in `localConfig` persistente lokale Konfiguration, unter anderem, ob die Erstkonfiguration bereits erfolgt ist.
Der PVC-Migrationsjob übernimmt dieses Verzeichnis zusammen mit `data`.
Die in der `dogu.json` beschriebene Dogu-Konfiguration wird dabei nicht in Helm-Values übersetzt.
Der Log-Level `logging/root` wird über die Values bereitgestellt. Die Datenbank im bisherigen PostgreSQL-Dogu wird durch einen separaten Job übertragen.

## Migrations-Jobs erstellen

Das Chart definiert die benötigten Jobs als reguläre Ressourcen unter `templates/`.
Der Operator aktiviert sie während der Migration und wartet auf ihren Abschluss.
Die Jobs werden nicht als Helm-Hooks angelegt.
Anzahl und Aufgaben der Jobs richten sich nach den zu übertragenden Daten.
Auch ein Dogu ohne Datenbank kann einen Job für die Übernahme von Dateien benötigen.

**Beispiel: Nexus**

Nexus benötigt zwei Jobs:

| Job | Aufgabe |
| --- | --- |
| [PVC-Migrationsjob][nexus-pvc] | Kopiert `data` und `localConfig` aus dem bisherigen Volume in den Ziel-PVC. |
| [PostgreSQL-Migrationsjob][nexus-db-job] | Überträgt die Datenbank aus der geteilten PostgreSQL-Instanz in die chart-eigene Instanz. |

Die vollständigen Templates sind in der Tabelle verlinkt.

### Aktivierung

In der `values.yaml` muss der folgende Schalter vorhanden sein:

```yaml
migrationJob:
  enabled: false
```

Bei einer Neuinstallation bleibt er `false`.
Für die Migration setzt ihn der Operator auf `true`; nach erfolgreichem Abschluss setzt er ihn wieder auf `false`.
Die Job-Templates werden nur bei aktiviertem Schalter gerendert.
Images, Commands, Mounts und Ressourcenbedarf legt das Chart selbst fest. Dafür ist kein zusätzliches generisches `migration`-Schema erforderlich.

**Beispiel: Nexus**

Die Nexus-Values enthalten `migrationJob.enabled: false`.
Sowohl der PVC- als auch der Datenbankjob sind im jeweiligen Template von `{{- if .Values.migrationJob.enabled }}` umschlossen.

### Labels

Das Chart kennzeichnet die Jobs für die Migrationserkennung und ihre Pods für den Zugriff auf geteilte Quellservices:

| Label | Platzierung | Zweck |
| --- | --- | --- |
| `k8s.cloudogu.com/migration-job: "true"` | `metadata.labels` des Jobs | Kennzeichnet einen Migrations-Job. Nur auf dem Pod gesetzt reicht es für die Job-Erkennung nicht aus. |
| `k8s.cloudogu.com/dogu.name: <dogu>` | Vom Operator an den Chart-Ressourcen ergänzt; vom Chart zusätzlich im Pod-Template gesetzt | Ordnet Job und Pods dem Dogu zu. |
| `k8s.cloudogu.com/migration-<name>: "true"` | `spec.template.metadata.labels` des Jobs | Erlaubt zusammen mit dem Dogu-Label den Zugriff über die temporäre NetworkPolicy auf den geteilten Quellservice. |

`<dogu>` bezeichnet den Namen des Dogus ohne Dogu-Namespace.
`<name>` in `k8s.cloudogu.com/migration-<name>` bezeichnet den **Quellservice**.
Dieses Label gehört nur auf die Pods, die den jeweiligen Service erreichen müssen.

Der Operator erkennt Jobs anhand von Migrationslabel und Dogu-Label gemeinsam.
Abgeschlossene Jobs müssen bis zum Ende der Migration beobachtbar bleiben. Eine TTL oder andere vorzeitige Bereinigung darf sie nicht vorher entfernen.

**Beispiel: Nexus**

Der [Nexus-Datenbankjob][nexus-db-job] benötigt Zugriff auf den geteilten PostgreSQL-Service.
Seine Pods tragen deshalb `migration-postgresql` und das Dogu-Label mit dem Wert `nexus`:

```yaml
{{- if .Values.migrationJob.enabled }}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "nexus.fullname" . }}-postgres-migration
  labels:
    k8s.cloudogu.com/migration-job: "true"
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        k8s.cloudogu.com/migration-postgresql: "true"
        k8s.cloudogu.com/dogu.name: nexus
    # Weitere Pod-Einstellungen siehe vollständiges Template.
{{- end }}
```

Der Nexus-PVC-Kopierjob benötigt kein PostgreSQL-Zugriffslabel.

### Quellressourcen

Der Operator stellt die bisherigen Daten und Credentials im Namespace des Dogus bereit:

| Ressource | Verwendung |
| --- | --- |
| PVC `<dogu>-migration` | Quelle für die Übernahme persistenter Dateien. |
| Secret `<dogu>-migration` | Quell-Credentials für bisher geteilte Services. |

Das Chart referenziert diese Ressourcen und legt seine eigenen Ziel-PVCs und Ziel-Credentials an.
Die Quellressourcen werden vom Operator angelegt und nach erfolgreicher Migration entfernt.
Secret und PVC können denselben Namen haben, da es unterschiedliche Ressourcenarten sind.

**Beispiel: Nexus**

Der PVC `nexus-migration` enthält die bisherigen Dateien des Dogus.
Das gleichnamige Secret enthält unter anderem die Zugangsdaten zur bisherigen PostgreSQL-Datenbank.

### Dateien übernehmen

Der Kopierjob bindet den Migration-PVC als Quelle und einen oder mehrere chart-eigene PVCs als Ziel ein.
Ziel-PVCs, Mounts und Verzeichnisse richten sich nach dem Datenlayout des Dogus.
Die Anwendung muss später genau die PVCs verwenden, in die der Job geschrieben hat.
Diese müssen auch nach dem Deaktivieren der Migration erhalten bleiben.

Der Name `<dogu>-migration` ist die feste Schnittstelle zur Quelle und wird nicht aus einem beliebig abweichenden Helm-Fullname abgeleitet.

**Beispiel: Nexus**

Der Nexus-Kopierjob bindet den Quell-PVC unter `/source` und den Ziel-PVC unter `/target` ein.
Er kopiert die Verzeichnisse `data` und `localConfig` in das Zielvolume.

Der Zielname wird im [Template][nexus-pvc] passend zum späteren StatefulSet-PVC festgelegt:

```gotemplate
{{- $pvcName := printf "%s-%s-0" "nexus-data" (include "nexus.fullname" .) }}
```

Die Volumes des Job-Pods referenzieren Quelle und Ziel:

```yaml
volumes:
  - name: source-data
    persistentVolumeClaim:
      claimName: nexus-migration
  - name: target-data
    persistentVolumeClaim:
      claimName: {{ $pvcName }}
```

Der Ziel-PVC heißt damit `nexus-data-<StatefulSet-Name>-0`.
Das Template legt ihn vorab mit `helm.sh/resource-policy: keep` an, damit er beim Deaktivieren des Migration-Templates erhalten bleibt.
Der Nexus-StatefulSet verwendet anschließend diesen PVC für seine Anwendungsdaten.

### Daten aus geteilten Services übernehmen

Der Migrationsjob greift mit den Credentials aus `<dogu>-migration` auf den bisherigen Service zu.
Quelladresse und Secret-Keys müssen den Vorgaben des jeweiligen Services entsprechen.
Die Zielinstanz und ihre Credentials definiert das Chart selbst.
Das Dogu implementiert die Übertragung der benötigten Daten in diese Zielinstanz.

**Beispiel: Nexus**

Der Nexus-Datenbankjob liest seine Quell-Credentials aus `nexus-migration`.
Die Keys für PostgreSQL sind `postgresql_database`, `postgresql_username` und `postgresql_password`:

```yaml
env:
  - name: SOURCE_POSTGRESQL_DATABASE
    valueFrom:
      secretKeyRef:
        name: nexus-migration
        key: postgresql_database
  - name: SOURCE_POSTGRESQL_USERNAME
    valueFrom:
      secretKeyRef:
        name: nexus-migration
        key: postgresql_username
  - name: SOURCE_POSTGRESQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: nexus-migration
        key: postgresql_password
```

Die Quelle ist `postgresql.<Namespace>.svc.cluster.local`; das Ziel ist die chart-eigene PostgreSQL-Instanz.
Das [vollständige Job-Template][nexus-db-job] und das [Nexus-Migrationsskript][nexus-db-script] zeigen die vorhandene Umsetzung.

### Anwendung während der Migration anhalten

Die Anwendung darf erst nach Übernahme ihrer Daten starten.
Ihr Workload muss deshalb bei aktiviertem `migrationJob.enabled` auf null Replikate gesetzt werden.
Die für die Übernahme benötigten Zielservices müssen währenddessen verfügbar bleiben.

**Beispiel: Nexus**

Der [Nexus-StatefulSet][nexus-statefulset] setzt dafür während der Migration die Replikate auf null:

```yaml
replicas: {{ if .Values.migrationJob.enabled }} 0 {{ else }} {{ .Values.nexus.replicas | default 1 }} {{ end }}
```

Die chart-eigene PostgreSQL-Instanz bleibt verfügbar, damit der Job Daten importieren kann.

### Anforderungen an die Jobs

| Anforderung | Bedeutung |
| --- | --- |
| Idempotenz | Ein erneuter Versuch nach einem Abbruch darf keine inkonsistenten oder doppelt angelegten Daten erzeugen. Ein Retry-Limit allein stellt dies nicht sicher. |
| Versionskompatibilität | Quellversion, Migrationswerkzeuge und Zielversion müssen zueinander passen. |
| Vollständigkeit | Alle benötigten Dateien und Daten aus geteilten Services müssen am später verwendeten Ziel ankommen. |
| Erkennbarer Abschluss | Ein erfolgreicher Job endet erfolgreich; Fehler werden als Fehlschlag gemeldet. |
| Stabile Ressourcennamen | Jobnamen, Ziel-PVCs und Referenzen bleiben bei erneutem Rendern gleich. Zufallswerte oder Versionsnummern dürfen sie nicht verändern. |

Weitere Hinweise zu Namen und Referenzen stehen unter [Vorsicht beim Umbenennen von Kubernetes-Ressourcen](../best-practices/helm-chart_de.md#vorsicht-beim-umbenennen-von-kubernetes-ressourcen).

**Beispiel: Nexus**

Nexus verwendet die stabilen Jobnamen `<fullname>-pvc-migration` und `<fullname>-postgres-migration`.
Bei seiner Datenbankmigration müssen Quell-PostgreSQL, die verwendeten Export-/Importwerkzeuge und die chart-eigene Zielversion kompatibel sein.

## Chart prüfen

Das Chart wird mit deaktivierter und mit aktivierter Migration gerendert.
Die Ergebnisse werden auf folgende Eigenschaften geprüft:

| Prüfung | Erwartetes Ergebnis |
| --- | --- |
| `migrationJob.enabled=false` | Keine Migrations-Jobs; der Anwendungsworkload verwendet seine normale Replikazahl. |
| `migrationJob.enabled=true` | Alle benötigten Jobs werden gerendert; die Anwendung bleibt bei null Replikaten, benötigte Zielservices sind verfügbar. |
| Labels | Das Migrationslabel steht am Job, service-spezifische Zugriffslabels und das Dogu-Label stehen am Pod-Template. |
| Quellreferenzen | PVC- und Secret-Namen folgen `<dogu>-migration`; die Secret-Keys passen zum Quellservice. |
| Zielreferenzen | Die Jobs schreiben in dieselben PVCs und Verzeichnisse, die der spätere Anwendungsworkload verwendet. |

Das lokale Rendern führt keine Migration aus und ergänzt keine vom Operator gesetzten Labels.
An den verwalteten Job-Ressourcen müssen später sowohl das Migrationslabel als auch das Dogu-Label vorhanden sein.

**Beispiel: Nexus**

Im Nexus-Repository liegt das Chart unter `k8s/helm`:

```shell
helm template nexus ./k8s/helm --set migrationJob.enabled=false
helm template nexus ./k8s/helm --set migrationJob.enabled=true
```

Mit aktiviertem Schalter werden PVC- und PostgreSQL-Migrationsjob gerendert.
Der Nexus-StatefulSet hat dann null Replikate, während die Ziel-PostgreSQL verfügbar bleibt.

## Fehleranalyse

Der Zustand der Migration steht in der `Ready`-Condition der V3-Dogu-Ressource.
Der Reason nennt den Zustand, die Message beschreibt den aktuellen Schritt oder Fehler. Events zeigen den Verlauf.

| Ready | Reason | Bedeutung |
| --- | --- | --- |
| `False` | `MigrationPending` | Die Migration läuft oder wartet. Die Message nennt den aktuellen Fortschritt. |
| `False` | `MigrationFailed` | Die Migration ist fehlgeschlagen. Message und Warning-Events nennen die Ursache. |
| `True` | Regulärer Erfolgs-Reason | Die Migration ist abgeschlossen. Das Dogu wird als normales V3-Dogu weiterbetrieben. |

`MigrationPending` und `MigrationFailed` sind Reasons von `Ready`, keine eigenen Condition-Typen.

**Beispiel: Nexus**

Die folgenden Befehle lesen den Zustand der Nexus-Migration im Namespace `ecosystem`:

```shell
# Ready-Condition
kubectl -n ecosystem get dogus.v3beta1.k8s.cloudogu.com nexus \
  -o jsonpath='{range .status.conditions[?(@.type=="Ready")]}{.status}: {.reason}: {.message}{"\n"}{end}'

# Events
kubectl -n ecosystem get events \
  --field-selector involvedObject.kind=Dogu,involvedObject.name=nexus
```

### Häufige Fehler

| Symptom | Prüfung im Chart |
| --- | --- |
| Kein Job wird innerhalb des Timeouts erkannt; `MigrationFailed`. | Wird der Job bei aktiviertem Schalter gerendert? Steht das Migrationslabel am Job und ist das Dogu-Label an der verwalteten Ressource vorhanden? |
| Job-Pod startet wegen fehlendem Secret oder Secret-Key nicht. | `<dogu>-migration` und die erwarteten Keys prüfen. Fehlt die Quellressource selbst, die zugehörigen Events an der Dogu-Ressource beachten. |
| Datenbankjob erreicht den geteilten Quellservice nicht. | Quelladresse, `migration-<name>` und Dogu-Label auf dem Pod prüfen; Pod-Events und Logs ergänzend heranziehen. |
| Kopierjob wartet auf einen PVC oder die Anwendung findet später keine übernommenen Dateien. | Claim-Namen, Mounts und Verzeichnisse mit dem späteren Anwendungsworkload vergleichen. |
| Job schlägt endgültig fehl; `MigrationFailed`. | Job-Logs auf Kopier- oder Importfehler, Rechte, Speicherbedarf und Versionskompatibilität prüfen. |

Wiederanlauf und weitere Schritte nach einer fehlgeschlagenen Migration beschreibt die [Betriebsanleitung][operations].

**Beispiel: Nexus**

Kann der Datenbankjob die bisherige PostgreSQL-Instanz nicht erreichen, werden zunächst Quelladresse und Pod-Labels geprüft.
Am Pod müssen `k8s.cloudogu.com/migration-postgresql: "true"` und `k8s.cloudogu.com/dogu.name: nexus` gesetzt sein.
Die Message der Dogu-Ressource und die Job-Logs helfen, den Fehler einzugrenzen.

## Referenzen 

[nexus-chart]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/Chart.yaml
[nexus-values]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/values.yaml
[nexus-config]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/config.yaml
[nexus-pvc]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/pvc-migration.yaml
[nexus-db-job]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/postgres-migration-job.yaml
[nexus-db-script]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/files/postgres-migration.sh
[nexus-statefulset]: https://github.com/cloudogu/nexus/blob/ad48f2a6b7688067c15b07b2ac3704b08a4d6c2e/k8s/helm/templates/statefulset.yaml
[operations]: https://github.com/cloudogu/k8s-dogu-operator/blob/a38702ddd26d4c19681915e2ff8858330a852fa6/docs/operations/v3/migrating_from_v2_to_v3_de.md
[sar]: https://github.com/cloudogu/k8s-serviceaccount-lib/blob/develop/docs/operations/serviceaccountrequest_cr_de.md
