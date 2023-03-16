# Kompendium

Dieses Dokument stellt das Kompendium der `dogu.json` dar. Es beschreibt alle Felder, einschließlich der abhängigen
Typen im Detail, die Syntax und Beispielinhalte. Für neue Entwickler ist der [Dogu](#type-dogu) Typ ein guter
Startpunkt.

# package core

## Index

- [type ConfigurationField](<#type-configurationfield>)
- [type Dependency](<#type-dependency>)
- [type Dogu](<#type-dogu>)
- [type DoguJsonV2FormatProvider](<#type-dogujsonv2formatprovider>)
- [type EnvironmentVariable](<#type-environmentvariable>)
- [type ExposedCommand](<#type-exposedcommand>)
- [type ExposedPort](<#type-exposedport>)
- [type HealthCheck](<#type-healthcheck>)
- [type ServiceAccount](<#type-serviceaccount>)
- [type ValidationDescriptor](<#type-validationdescriptor>)
- [type Volume](<#type-volume>)
- [type VolumeClient](<#type-volumeclient>)

## type [ConfigurationField](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L320-L395>)

ConfigurationField beschreibt ein einzelnes Dogu-Konfigurationsfeld, das in der Cloudogu EcoSystem-Registry gespeichert
ist.

```go
type ConfigurationField struct {
    Name string

    Description string

    Optional bool

    Encrypted bool

    Global bool

    Default string

    Validation ValidationDescriptor
}
```

### Name

Name enthält den Namen des Konfigurationsschlüssels. Das Feld ist obligatorisch. Es darf keine führenden oder
abschließenden Schrägstriche "/" enthalten, aber es kann Verzeichnisschlüssel enthalten, die mit Schrägstrichen "/"
innerhalb des Namens begrenzt sind.

Die Syntax des Namens sollte aus folgenden Zeichen bestehen:

- lateinische Kleinbuchstaben
- Sonderzeichen Unterstrich "_"
- Ziffern 0-9

Beispiel:

- feedback_url
- logging/root

### Description

Die Beschreibung bietet den Kontext und den Zweck des Konfigurationsfeldes in einem für Menschen lesbaren Format. Dieses
Feld ist optional, sollte aber auf jeden Fall gesetzt werden.

Beispiel:

- "Setzen Sie das Root-Log-Level auf einen der Werte ERROR, WARN, INFO, DEBUG oder TRACE. Standard ist INFO"
- "URL des Feedback-Dienstes"

### Optional

Optional bedeutet, dass dieses Konfigurationsfeld nicht gesetzt werden muss, andernfalls muss ein Wert angegeben werden.
Dieses Feld ist optional. Wenn es nicht gesetzt ist, wird ein Wert von `false` angenommen.

Beispiel:

- true

### Encrypted

Encrypted kennzeichnet dieses Konfigurationsfeld als einen sensiblen Wert, der mit dem privaten Schlüssel des Dogus
verschlüsselt wird. Dieses Feld ist optional. Wenn es nicht gesetzt ist, wird der Wert `false` angenommen.

Beispiel:

- true

### Global

Global bedeutet, dass dieses Konfigurationsfeld einen Wert enthält, der für alle Dogus verfügbar ist. Dieses Feld ist
optional. Wenn es nicht gesetzt ist, wird ein Wert von "false" angenommen.

Beispiel:

- true

### Default

Default definiert einen Standardwert, der ausgewertet werden kann, wenn kein Wert konfiguriert wurde oder der Wert leer
oder sogar ungültig ist. Dieses Feld ist optional.

Beispiel:

- "WARN"
- "true"
- "[https://scm-manager.org/plugins](https://scm-manager.org/plugins)"

### Validation

Validation konfiguriert einen Validator, der verwendet wird, um ungültige oder außerhalb des Bereichs liegende Werte für
dieses Konfigurationsfeld zu markieren. Dieses Feld ist optional.

Beispiel:

```
"Validation": {
   "Type": "ONE_OF", // only allows one of these two values
   "Values": [
     "value 1",
     "value 2",
   ]
 }

"Validation": {
   "Type": "FLOAT_PERCENTAGE_HUNDRED" // valid values range between 0.0 and 100.0
}

"Validation": {
  "Type": "BINARY_MEASUREMENT" // only allows suffixed integer values measured in byte, kibibyte, mebibyte, gibibyte
 }
```

## type [Dependency](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L454-L503>)

Dependency beschreibt die Abhängigkeit eines Dogus zu einer anderen Entität.

Beispiel:

```
{
  "type": "dogu",
  "name": "postgresql"
}

{
  "name": "postgresql"
}

{
  "type": "client",
  "name": "k8s-dogu-operator",
  "version": ">=0.16.0"
}

{
  "type": "package",
  "name": "cesappd",
  "version": ">=3.2.0"
}
```

```go
type Dependency struct {
    Type string `json:"type"`

    Name string `json:"name"`

    Version string `json:"version"`
}
```

### Type

Type identifiziert die Entität, von der das Dogu abhängt. Dieses Feld ist optional. Wenn es nicht gesetzt ist, wird der
Wert `dogu` angenommen.

Gültige Werte sind einer der folgenden: "dogu", "client", "package".

Der Typ "dogu" verweist auf einen anderes Dogu, das während der Abhängigkeitsüberprüfung vorhanden und aktiv sein muss.

Ein Typ "client" verweist auf den Client, der die "dogu.json" des Dogus verarbeitet. Mehrere Client-Abhängigkeiten eines
anderen Client-Typs können z. B. verwendet werden, um die Verarbeitung eines bestimmten Clients zu verbieten.

Ein Typ "package" verweist auf ein notwendiges Betriebssystem-Paket, das bei der Abhängigkeitsüberprüfung vorhanden sein
muss.

Beispiele:

- "dogu"
- "client"
- "package"

### Name

Name identifiziert die durch den Typ ausgewählte Entität. Dieses Feld ist obligatorisch. Wenn der Typ eine andere
Entität auswählt, muss Name den einfachen Entitätsnamen verwenden (z. B. "postgres"), nicht den voll qualifizierten
Entitätsnamen (nicht "official/postgres").

Beispiele:

- "postgresql"
- "k8s-dogu-operator"
- "cesappd"

### Version

Version wählt die Version der durch den Typ ausgewählten Entität aus. Dieses Feld ist optional. Wenn es nicht gesetzt
ist, wird jede Version der ausgewählten Entität bei der Abhängigkeitsüberprüfung akzeptiert.

Version akzeptiert verschiedene Versionsstile und Vergleichsoperatoren.

Beispiele:

- ">=4.1.1-2" - akzeptiert größer oder gleich der Version 4.1.1-2
- "<=1.0.1" - akzeptiert eine Entität, die kleiner oder gleich der Version 1.0.1 ist
- "1.2.3.4" - wählt genau die Version 1.2.3.4 aus

Mit einer nicht existierenden Version ist es möglich, eine Abhängigkeit zu negieren.

Beispiel:

- "<=0.0.0" - verbietet, dass die ausgewählte Entität vorhanden ist

## type [Dogu](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L510-L951>)

Dogu beschreibt die Eigenschaften einer containerisierten Anwendung für das Cloudogu EcoSystem. Neben den
Meta-Informationen und dem [OCI Container Image](https://opencontainers.org/) beschreibt das Dogu alle Notwendigkeiten
für die automatische Container-Instantiierung, z. B. Volumes, Abhängigkeiten zu anderen Dogus und vieles mehr.

```go
type Dogu struct {
    Name string
    
    Version string
    
    DisplayName string
    
    Description string
    
    Category string
    
    Tags []string
    
    Logo string
    
    URL string
    
    Image string
    
    ExposedPorts []ExposedPort
    
    ExposedCommands []ExposedCommand
    
    Volumes []Volume
    
    HealthCheck HealthCheck
    
    HealthChecks []HealthCheck
    
    ServiceAccounts []ServiceAccount
    
    Privileged bool
    
    Configuration []ConfigurationField
    
    Properties Properties
    
    EnvironmentVariables []EnvironmentVariable
    
    Dependencies []Dependency
    
    OptionalDependencies []Dependency
}
```

### Name

Name enthält den voll qualifizierten Namen des Dogus, der sich aus dem Dogu-Namensraum und dem einfachen Namen des Dogus
zusammensetzt, begrenzt durch einen einfachen Schrägstrich "/". Dieses Feld ist obligatorisch.

Der Dogu-Namespace ermöglicht es, den Zugriff auf Dogus in diesem Namespace zu regeln. Es gibt drei reservierte
Dogu-Namespaces: Die Namespaces `official` und `k8s` sind für alle Benutzer ohne weitere Kosten zugänglich. Im Gegensatz
dazu ist der Namespace `premium` nur für Abonnement-Nutzer zugänglich.

Die Namespace-Syntax sollte aus folgenden Zeichen bestehen:

- lateinische Kleinbuchstaben
- Sonderzeichen Unterstrich "_", Minus "-"
- Ziffern 0-9
- einer Gesamtlänge von weniger als 200 Zeichen

Der einfache Name von Dogu ermöglicht mehrere Adressierungsarten. Der einfache Name wird als Teil der URI des Cloudogu
EcoSystems verwendet, um einen URI-Teil zu adressieren (wenn das Dogu eine exponierte UI bietet). Außerdem wird der
einfache Name verwendet, um das Dogu nach dem Installationsprozess zu adressieren (z. B. um ein Dogu zu starten, zu
stoppen oder zu entfernen), oder um generierte Ressourcen zu adressieren, die zum Dogu gehören.

Die Syntax des einfachen Namens muss ein DNS-kompatibler Bezeichner sein und sollte aus folgenden Elementen bestehen:

- lateinischen Kleinbuchstaben
- Sonderzeichen Unterstrich "_", Minus "-"
- Ziffern 0-9
- eine Gesamtlänge von weniger als 20 Zeichen

Es wird empfohlen, denselben voll qualifizierten Namen innerhalb der Dockerfiles des Dogus als Umgebungsvariable `NAME`
zu verwenden.

Beispiele:

- official/redmine
- premium/confluence
- foo-1/bar-2

### Version

Version definiert die aktuelle Version des Dogus. Dieses Feld ist ein Pflichtfeld.

Die Version folgt dem Format der semantischen Versionierung und ist zusätzlich in zwei Teile aufgeteilt. Die
Anwendungsversion und die Dogu-Version.

Ein Beispiel wäre 1.7.8-1 oder 2.2.0-4. Der erste Teil der Version (z. B. 1.7.8) steht für die Version der Anwendung (z.
B. die nginx-Version in dem nginx-Dogu). Der zweite Teil steht für die Version des Dogus und sollte bei einer
Erstveröffentlichung mit 1 beginnen (z. B. 1.7.8-1).

Wenn sich die Anwendung nicht ändert, aber z.B. Änderungen im Startskript des Dogus vorgenommen werden, sollte die neue
Version 1.7.8-2 sein. Wenn sich die Anwendung selbst ändert (z. B. ein nginx-Upgrade), sollte die neue Version 1.7.9-1
sein. Beachten Sie, dass in diesem Fall die Version des Dogus wieder auf 1 gesetzt werden sollte.

Während die `dogu.json` der zentrale Ort für die Version ist und von der cesapp in verschiedenen Prozessen wie
Installation und Release verwendet wird, sollte die Version auch als Label in dem Dockerfile des Dogus platziert
werden.

Beispielversionen in der dogu.json:

- 1.7.8-1
- 2.2.0-4

Empfohlenes Beispiel im Dockerfile:

```
LABEL maintainer="hello@cloudogu.com" \
NAME="official/nginx" \
VERSION="1.23.2-1"
```

### DisplayName

DisplayName ist der Name des Dogus, der in UI-Frontends zur Darstellung des Dogus verwendet wird. Dieses Feld ist ein
Pflichtfeld.

Verwendungen: Bei der Einrichtung des Ökosystems wird der Anzeigename des Dogus verwendet, um ihn für die Installation
auszuwählen.

Der Anzeigename wird im Warp-Menü verwendet, damit der Benutzer zur Web-UI des Dogus navigieren kann.

Eine weitere Verwendung ist die textuelle Ausgabe von Tools wie `cesapp` oder `k8s-dogu-operator`, wo der Name in
Befehlen wie `list upgradeable dogus` verwendet wird.

Die Beschreibung kann bestehen aus:

- lateinischen Klein- und Großbuchstaben, wobei der Erste ein Großbuchstabe ist
- beliebige Sonderzeichen
- Ziffern 0-9

Der DisplayName sollte aus weniger als 30 Zeichen bestehen, da er im Warp-Menü des Cloudogu EcoSystems angezeigt wird.

Beispiele:

- Jenkins CI
- Backup & Restore
- SCM-Manager
- Smeagol

### Description

Die Beschreibung enthält eine kurze Erklärung, was das Dogu leistet. Dieses Feld ist ein Pflichtfeld.

Es wird bei der Einrichtung des Ökosystems in der Auswahl des Dogus verwendet. Daher sollte die Beschreibung einem
uninformierten Benutzer einen kurzen Hinweis darauf geben, was das Dogu ist und welche Funktionen es bietet.

Die Beschreibung kann bestehen aus:

- lateinischen Klein- und Großbuchstaben, wobei der erste Großbuchstabe ist
- beliebige Sonderzeichen
- Ziffern 0-9

Die Beschreibung sollte aus einem lesbaren Satz bestehen, der kurz das Hauptthema des Dogus erklärt.

Beispiele:

- Jenkins Continuous Integration Server
- MySQL - Relational database
- The Nexus Repository is like the local warehouse where all the parts and finished goods used in your software supply
  chain are stored and distributed.

### Category

Die Kategorie organisiert die Dogus in drei Kategorien. Dieses Feld ist obligatorisch.

Die Liste der Kategorien ist fix, es muss eines hiervon  gewählt werden:

- `Development Apps` - Für reguläre Dogus, die von einem regulären Benutzer des Ökosystems verwendet werden sollen,
- `Administration Apps` - Für Dogus, die von einem Benutzer mit Administrationsrechten verwendet werden sollen
- `Base` - Für Dogus, die für das Gesamtsystem wichtig sind.

Die Kategorien "Development Apps" und "Administration Apps" werden im Warp-Menü zur Anordnung der Dogus dargestellt.

Beispiel-Dogus für jede Kategorie:

- `Development Apps`: Redmine, SCM-Manager, Jenkins
- `Administration Apps`: Sicherung & Wiederherstellung, Benutzerverwaltung
- `Base`: Nginx, Registrator, OpenLDAP

### Tags

Tags enthält eine Liste von Ein-Wort-Tags, die im Zusammenhang mit dem Dogu stehen. Dieses Feld ist optional.

Wenn das Dogus im Warp-Menü angezeigt werden soll, ist das Tag "warp" notwendig. Andere Tags werden nicht automatisch
verarbeitet.

Beispiele für z. B. Jenkins:

```
{"warp", "build", "ci", "cd"}
```

### Logo

Logo ist ein URI zu einem Webbild, das das Dogu darstellt. Dieses Feld ist optional.

Veraltet: Das Cloudogu EcoSystem unterstützt den Logo URI nicht. Es ist ein Kandidat für die Entfernung. Andere Optionen
zur Darstellung eines Tools oder einer Anwendung können sein:

- Das Logo in das Git-Repository von Dogus einbetten (falls öffentlich)
- Bereitstellung des Logos in der Dogu-Benutzeroberfläche (wenn der Dogus eine solche bereitstellt)

### URL

Die URL verlinkt die Website mit dem ursprünglichen Anbieter des Tools. Dieses Feld ist optional.

Das Cloudogu EcoSystem unterstützt diese Information nicht. In einem öffentlichen Dogu-Repository, in dem ein
Dogu-Anbieter eine Anwendung eines Drittanbieters neu verpackt, kann die URL jedoch auf die Ressourcen des
ursprünglichen Tool-Anbieters verweisen.

Beispiel:

- [https://github.com/cloudogu/usermgt](https://github.com/cloudogu/usermgt)
- [https://www.atlassian.com/software/jira](https://www.atlassian.com/software/jira)

### Image

Image verweist auf das Image des [OCI-Containers](https://opencontainers.org/), in dem die Dogu-Anwendung verpackt ist.
Dieses Feld ist obligatorisch.

Das Image darf keine Image-Tags wie die Image-Version oder "latest" enthalten (verwenden Sie stattdessen das Feld
Version für diese Informationen). Der Image-Registry-Teil dieses Feldes muss auf "registry.cloudogu.com" verweisen.

Es ist eine gute Praxis, den gleichen Namen für das Build-Repository zu verwenden wie im Feld Name, um Zugriffsstrategien
zu ermöglichen und Speicherkonflikte zu vermeiden.

Beispiele für official/redmine:

- registry.cloudogu.com/official/redmine

### ExposedPorts

ExposedPorts enthält eine Liste von [ExposedPort](#type-exposedport). Dieses Feld ist optional.

ExposedPorts bieten eine Möglichkeit, Datenverkehr von außerhalb des Cloudogu EcoSystems an Dogus zu leiten. Dogus, die
eine grafische Benutzeroberfläche (GUI) anbieten, müssen auch einen Port bereitstellen, der Anfragen annimmt und
Antworten zurückgibt. Dogus, die keine grafische Benutzeroberfläche anbieten, können ebenfalls Ports freigeben, wenn das
Dogu Dienste für einen Verbraucher bereitstellt (z. B. wenn das Dogu eine API für ein CLI-Tool bereitstellen möchte).

Beispiele:

```
[ { "Typ": "tcp", "Container": "2222", "Host": "2222" } ]
```

### ExposedCommands

ExposedCommands definieren Aktionen des Typen [ExposedCommand](#type-exposedcommand), die in verschiedenen Phasen des Dogu-Lebenszyklus automatisch von einem
Dogus-Client wie der cesapp oder dem k8s-dogu-operator oder manuell von einem administrativen Benutzer ausgelöst werden
können. Dieses Feld ist optional.

Die folgenden Befehlsbezeichner beziehen sich auf Befehle, die von einem Dogu-Client automatisch im Rahmen von Upgrade- oder Service-Account-Prozessen ausgelöst werden. Diese Befehle sind: pre-upgrade, post-upgrade, upgrade-notification, service-account-create, service-account-remove

pre-upgrade: Dieser Befehl wird in einer frühen Phase eines Upgrade-Prozesses von einem Dogu ausgeführt. Ein
Dogu-Client mountet das Pre-Upgrade-Skript der neuen Dogu-Version in den Container des alten, noch laufenden Dogus und
führt es aus. Es wird hauptsächlich zur Vorbereitung von Datenmigrationen (z. B. Export einer Datenbank) verwendet. Kern
des Skriptes sollte der Vergleich zwischen den Versionen sein, um festzustellen, ob eine Migration notwendig ist. Dazu
wird der Dogu-Client das Skript mit der alten Version als erstem und der neuen Version als zweitem Parameter aufrufen.
Außerdem ist es empfehlenswert, im Skript Zustände wie "upgrade" oder "pre-upgrade done" zu setzen. Dies kann sehr
nützlich sein, da Sie es im Post-Upgrade oder beim regulären Start für eine Wartefunktion verwenden können.

post-upgrade: Dieser Befehl wird nach einem regulären Dogu-Upgrade ausgeführt. Wie beim Pre-Upgrade wird als erster
Parameter die alte Dogu-Version und als zweiter Parameter die neue Dogu-Version übergeben. Sie sollten verwendet
werden, um festzustellen, ob eine Aktion erforderlich ist. Denken Sie daran, dass in dieser Zeit auch das reguläre
Startskript Ihres neuen Containers ausgeführt wird. Verwenden Sie einen Zustand im Cloudogu EcoSystem-Registry, um eine
Wartefunktion beim
Start zu handhaben. Wenn das Post-Upgrade endet, setzen Sie diesen Status zurück und starten den regulären Container.

upgrade-notification: Wenn der Upgrade-Prozess z.B. mit sehr sensiblen Daten arbeitet, kann ein
Upgrade-Benachrichtigungsskript implementiert werden, um den Benutzer über das Risiko des Prozesses zu informieren
und z.B. eine Backup-Anweisung zu geben. Vor einem Upgrade führt der Dogus-Client das Benachrichtigungsskript aus, gibt
alle Upgrade-Schritte aus und fragt den Benutzer, ob er fortfahren möchte.

service-account-create: Der Befehl service-account-create wird im Installationsprozess eines Dogus verwendet. Ein
Service-Account im Cloudogu EcoSystem ist ein Berechtigungsnachweis zur Autorisierung eines anderen Dogus, z. B. einer
Datenbank oder eines Authentifizierungsservers. Der Befehl "service-account-create" muss in dem Dogu implementiert
werden, der den Service-Account herstellt und die Anmeldeinformationen erzeugt. Wenn ein
Service-Account-Verbraucher-Dogu
installiert wird und einen Service-Account benötigt (z. B. für postgresql), ruft der Dogu-Client das
service-account-create Skript in dem postgresql-Dogu mit dem Servicenamen als ersten Parameter und benutzerdefinierten
Parametern als zusätzliche Parameter auf. Siehe [core.ServiceAccount](#type-serviceaccount) für die Definition von
benutzerdefinierten Parametern. Mit diesen Informationen sollte das Skript ein Service-Account erstellen und speichern (
z. B. in der Tabelle USER in einer zugrunde liegenden Datenbank oder vielleicht verschlüsselt in der Cloudogu
EcoSystem-Registry). Danach
müssen
die Anmeldedaten auf der Konsole ausgegeben werden, damit der Dogu-Client die Anmeldedaten für das Dogu speichert, das
den Service-Account angefordert hat. Es ist auch wichtig, dass diese Ausgaben die einzigen des Skripts sind, da der
Dogu-Client sie sonst als Anmeldedaten verwendet.

Beispiel output:

```
echo "database: ${DATABASE}"
echo "username: ${USER}"
echo "password: ${PASSWORD}"
```

service-account-delete: Wenn ein Dogu gelöscht wird, sollte auch der verwendete Service-Account gelöscht werden. Aus
diesem Grund muss das Dogu, das Service-Accounts erzeugt (wie postgresql), einen service-account-delete Befehl
implementieren. Dieses Skript wird bei der Löschung eines Dogus ausgeführt. Der Dogu-Client übergibt den
Namen des Service-Accounts als einzigen Parameter. Im Gegensatz zum Befehl service-account-create kann das Skript
Logs ausgeben, da der Dogu-Client diese Daten nicht verwendet.

Beispiel ExposedCommands für Service-Accounts:

```
"ExposedCommands": [
   {
     "Name": "service-account-create",
     "Description": "Creates a new service account",
     "Command": "/create-sa.sh"
   },
   {
     "Name": "service-account-remove",
     "Description": "Removes a service account",
     "Command": "/remove-sa.sh"
   }
 ],
```

Benutzerdefinierte Befehle: Darüber hinaus kann ein Dogu Befehle angeben, die nicht automatisch im Lebenszyklus des
Dogus ausgeführt werden (z. B. Upgrade). Zum Beispiel kann ein Dogu wie Redmine einen Befehl zum Löschen oder
Installieren eines Plugins angeben. Zur Laufzeit kann ein Administrator diesen Befehl mit der cesapp wie folgt aufrufen:

```
cesapp command redmine plugin-install scrum-plugin
```

Beispiel:

```
"ExposedCommands": [
   {
     "Name": "plugin-install",
     "Description": "Installs a plugin",
     "Command": "/installPlugin.sh"
   }
 ],
```

### Volumes

Volumes enthält eine Liste von [Volume](#type-volume), die [OCI container volumes](https://opencontainers.org/)
definiert. Dieses Feld ist optional.

Volumes werden während der Erstellung oder des Upgrades eines Dogus erstellt. Volumes bieten eine Leistungssteigerung im
Vergleich zum Container-Speicher.

Beispiele:

```
[ { "Name": "temp", "Path":"/tmp", "Owner": "1000", "Group": "1000", "NeedsBackup": false} ]
```

### HealthCheck

HealthCheck definiert eine einzige Möglichkeit, den Zustand des Dogus auf Beobachtbarkeit zu prüfen. Dieses Feld ist
optional.

Veraltet: verwenden Sie stattdessen [HealthChecks](#healthchecks).

### HealthChecks

HealthChecks definieren eine Liste von [HealthCheck](#type-healthcheck). Sie bieten mehrere Möglichkeiten, den Zustand
des Dogus auf Beobachtbarkeit zu prüfen. Dieses Feld ist optional.

HealthChecks werden in verschiedenen Anwendungsfällen verwendet:

- um die `dogu is starting` Seite anzuzeigen, bis das Dogu beim Start healthy ist
- zur Überwachung über `cesapp healthy <dogu-name>`
- für die Überwachung über das Admin-Dogu
- um zu vermeiden, dass ein unhealthy Dogu gesichert wird

Es gibt verschiedene Arten von HealthChecks:

- state, über den `/state/<dogu>` Registry-Schlüssel, der von ces-setup gesetzt wird
- tcp, um auf einen offenen Port zu prüfen
- http, um einen Statuscode einer http-Antwort zu prüfen

Sie müssen ohne Authentifizierung ausführbar sein.

Beispiel:

```
"HealthChecks": [
  {"Type": "state"}
  {"Type": "tcp",  "Port": 8080},
  {"Type": "http", "Port": 8080, "Path": "/my/health/path"},
]
```

### ServiceAccounts

ServiceAccounts enthält eine Liste von [ServiceAccount](#type-serviceaccount). Dieses Feld ist optional.

Ein Service-Account schützt sensible Daten, die für ein anderes Dogus verwendet werden. Service-Account-Konsumenten
werden von Service-Account-Produzenten unterschieden. Um ein Servicekonto zu erzeugen, muss ein Dogu die Skripte
service-account-create und service-account-delete in [core.ExposedCommands](#exposedcommands) implementieren.
Um ein
Service-Account zu konsumieren, sollte ein Dogus eine Anfrage in ServiceAccounts definieren.

Bei der Installation erkennt ein Dogu-Client eine Service-Account-Anfrage und führt das Skript "service-account-create"
mit den Informationen des Service-Account-Produzenten aus. Die Anmeldedaten werden dann in der Cloudogu
EcoSystem-Registry gespeichert.

Beispielpfade:

- config/redmine/sa-postgresql/username
- config/redmine/sa-postgresql/password
- config/redmine/sa-postgresql/database

Das Dogu selbst kann dann den Service-Account verwenden, indem er die Anmeldeinformationen mit `doguctl` im Startskript
liest und entschlüsselt.

Beispiele:

```
"ServiceAccounts": [
  {
    "Type": "ldap",
    "Params": [
      "rw"
    ]
  },
  {
   "Type": "k8s-dogu-operator",
   "Kind": "k8s"
  }
],
```

### Privileged

Privileged gibt an, ob der Docker-Socket in das Container-Dateisystem eingehängt werden soll. Dieses Feld ist optional.
Der Standardwert ist `false`.

Aus Sicherheitsgründen ist es sehr empfehlenswert, Privileged auf false zu setzen, da fast kein Dogu rückwirkend
Einblick in den Container erhalten sollte.

Beispiel:

- false

### Configuration

Configuration enthält eine Liste von [ConfigurationField](#type-configurationfield). Dieses Feld ist optional.

Es beschreibt allgemeine Eigenschaften des Dogus in der Cloudogu EcoSystem-Registry.

Beispiele:

```
{
  "Name": "plugin_center_url",
  "Description": "URL of SCM-Manager Plugin Center",
  "Optional": true
}

{
  "Name": "logging/root",
  "Description": "Set the root log level to one of ERROR, WARN, INFO, DEBUG or TRACE. Default is INFO",
  "Optional": true,
  "Default": "INFO",
  "Validation": {
    "Type": "ONE_OF",
    "Values": [
      "WARN",
      "ERROR",
      "INFO",
      "DEBUG",
      "TRACE"
    ]
  }
}
```

### Properties

Properties is eine `map[string]string` von Properties. Dieses Feld ist optional. Es beschreibt generische
Eigenschaften des Dogus, die von einem Client wie cesapp oder k8s-dogu-operator ausgewertet werden.

Beispiel:

```
{
  "key1": "value1",
  "key2": "value2"
}
```

### EnvironmentVariables

EnvironmentVariables enthält eine Liste von [EnvironmentVariable](#type-environmentvariable), die im Container
gesetzt werden sollen. Dieses Feld ist optional.

Beispiel:

```
[
  {"Key": "my_key", "Value": "my_value"}
]
```

### Dependencies

Dependencies enthält eine Liste von [Dependency](#type-dependency). Dieses Feld ist optional.

In diesem Feld werden Abhängigkeiten definiert, die während der Abhängigkeitsprüfung erfüllt sein müssen. Wenn die
Abhängigkeit während der Prüfung nicht erfüllt werden kann, wird ein Fehler ausgelöst und die Verarbeitung wird
abgebrochen.

Beispiele:

```
[
  {
    "type": "dogu",
    "name": "postgresql"
  },
  {
    "type": "client",
    "name": "cesapp",
    "version": ">=6.0.0"
  },
  {
    "type": "package",
    "name": "cesappd",
    "version": ">=3.2.0"
  }
]
```

### OptionalDependencies

OptionalDependencies enthält eine Liste von [Dependency](#type-dependency). Dieses Feld ist optional.

Im Gegensatz zu Dependencies können mit OptionalDependencies Abhängigkeiten definiert werden, die
erfüllt
werden können, wenn sie vorhanden sind. Wenn eine optionale Abhängigkeit nicht vorhanden ist, hat dies keine negativen
Auswirkungen bei der Prüfung der Abhängigkeit. Ist die optionale Abhängigkeit jedoch vorhanden und kann sie während der
Abhängigkeitsprüfung nicht erfüllt werden, so wird ein Fehler ausgelöst und die Verarbeitung wird abgebrochen.

Beispiele:

```
[
  {
    "type": "dogu",
    "name": "ldap"
  }
]
```

## type [DoguJsonV2FormatProvider](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L1177>)

DoguJsonV2FormatProvider bietet Methoden, um Dogus kompatibel zur v2 API zu formatieren.

```go
type DoguJsonV2FormatProvider struct{}
```

## type [EnvironmentVariable](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L280-L283>)

EnvironmentVariable repräsentiert benutzerdefinierte Parameter, die das Verhalten eines Dogus Build-Prozesses
verändern können.

```go
type EnvironmentVariable struct {
    Key   string
    Value string
}
```

## type [ExposedCommand](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L243-L276>)

ExposedCommand repräsentiert einen Befehl, der innerhalb des Dogus ausgeführt werden kann.

```go
type ExposedCommand struct {
    Name string

    Description string

    Command string
}
```

### Name

Name identifiziert den exponierten Befehl. Dieses Feld ist obligatorisch. Er muss in allen Befehlen desselben Dogus
eindeutig sein.

Der Name muss bestehen aus:

- lateinischen Kleinbuchstaben
- Sonderzeichen Unterstrich "_", Minus "-"
- Ziffern 0-9

Beispiele:

- service-account-create
- plugin-delete

### Description

Die Description beschreibt den exponierten Befehl in wenigen Worten. Dieses Feld ist optional.

Beispiel:

- Creates a new service account
- Deletes a plugin

### Command

Command gibt das Skript an, das für diesen Befehl ausgeführt werden soll. Dieses Feld ist obligatorisch.

Die Syntax des Namens muss der Dateisystem-Syntax des jeweiligen Host-Betriebssystems entsprechen und sollte aus
folgenden Zeichen bestehen:

- Kleinbuchstaben, lateinische Zeichen
- Sonderzeichen Unterstrich "_", Minus "-"
- Ziffern 0-9

Beispiele:

- /resources/create-sa.sh
- /resources/deletePlugin.sh

## type [ExposedPort](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L204-L231>)

ExposedPort wird verwendet, um Ports zu definieren, die an den Host exportiert werden.

Beispiel:

```
{ "Type": "tcp", "Container": "2222", "Host":"2222" }
{ "Container": "2222", "Host":"2222" }
```

```go
type ExposedPort struct {
    Type string

    Container int

    Host int
}
```

### Type

Type enthält den Protokolltyp, über den der Container kommuniziert (z. B. 'tcp'). Dieses Feld ist optional (der Wert
von `tcp` wird dann angenommen).

Beispiel:

- tcp
- udp
- sctp

### Container

Container enthält den gemappten Port auf der Seite des Containers. Dieses Feld ist obligatorisch. Es gelten die üblichen
Einschränkungen des Portbereichs.

Beispiele:

- 80
- 8080
- 65535

### Host

Host enthält den gemappten Port auf der Seite des Hosts. Dieses Feld ist obligatorisch. Es gelten die üblichen
Einschränkungen des Portbereichs.

Beispiele:

- 80
- 8080
- 65535

## type [HealthCheck](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L171-L196>)

HealthCheck bietet Verfügbarkeitsprüfungen für den Dogus-Container.

```go
type HealthCheck struct {
    Type string

    State string

    Port int

    Path string

    Parameters map[string]string
}
```

### Type

Type gibt die Art der Prüfung an. Dieses Feld ist obligatorisch. Es kann entweder tcp, http oder state sein.

Bei Typ "tcp" muss der angegebene Port offen sein, um als healthy zu gelten.

Bei Typ "http" muss der Dienst einen Statuscode zwischen >= 200 und < 300 zurückgeben, um als healthy zu gelten.
Port und Pfad werden verwendet, um den Dienst zu erreichen.

Für den Typ "state" wird der Schlüssel /state/<dogu> in der Cloudogu EcoSystem-Registry überprüft. Dieser Schlüssel
wird durch den Installationsprozess geschrieben. Ein 'ready'-Wert in der Registry bedeutet, dass das Dogu healthy ist.

### State

State enthält den erwarteten Status der Prüfung von Type state. Dieses Feld ist optional, auch für
Prüfungen vom Typ "state". Der Standardwert ist "ready".

### Port

Port ist der tcp-Port für Prüfungen vom Typ tcp und http. Dieses Feld ist obligatorisch für die Prüfungen vom
Typ "tcp" und "http".

### Parameters

Parameter können Schlüssel-Werte-Paare für prüfungsspezifische Parameter enthalten.

Deprecated: wird nicht verwendet.

## type [ServiceAccount](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L299-L317>)

ServiceAccount kann verwendet werden, um Zugang zu einem anderen Dogu zu erhalten.

Example:

```
{
 "Type": "k8s-dogu-operator",
 "Kind": "k8s"
}
```

```go
type ServiceAccount struct {
    Type string

    Params []string

    Kind string `json:"Kind,omitempty"`
}
```

### Type

Type enthält den Namen des Dienstes, für den das Konto erstellt werden soll. Dieses Feld ist obligatorisch.

Beispiel:

- postgresql
- dein-dogu

### Params

Params enthält zusätzliche Argumente, die für die Erstellung des Service-Accounts erforderlich sind. Die Optionalität
dieses Feldes hängt vom gewünschten Service-Account-Produzenten-Dogu ab. Bitte konsultieren Sie das jeweiligen Dogu.

### Kind

Kind definiert die Art des Dienstes, auf dem das Konto erstellt werden soll, z.B. `dogu` oder `k8s`. Dieses Feld ist
optional. Wenn es leer ist, sollte der Standardwert `dogu` angenommen werden.

Das Auslesen dieser Eigenschaft und die Erstellung eines entsprechenden Service-Accounts obliegt dem Client.

## type [ValidationDescriptor](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L398-L411>)

ValidationDescriptor beschreibt, wie man feststellt, ob ein Konfigurationswert gültig ist.

```go
type ValidationDescriptor struct {
    Type string

    Values []string
}
```

### Type

Type enthält den Namen des Konfigurationswert-Validators. Dieses Feld ist obligatorisch. Gültige Typen sind:

- ONE_OF
- BINARY_MEASUREMENT
- FLOAT_PERCENTAGE_HUNDRED

### Values

Values können Werte enthalten, die dem ausgewählten Validator helfen. Je nach dem verwendeten Type
können die Werte optional sein oder nicht. Es hängt vom gewählten Validator ab, ob dieses Feld obligatorisch, optional
oder unbenutzt ist.

## type [Volume](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L74-L136>)

Volume definiert Container-Volumes, die während der Erstellung oder des Upgrades eines Dogus erstellt werden.

Examples:

- { "Name": "data", "Path":"/usr/share/yourtool/data", "Owner":"1000", "Group":"1000", "NeedsBackup": true}
- { "Name": "temp", "Path":"/tmp", "Owner":"1000", "Group":"1000", "NeedsBackup": false}

```go
type Volume struct {
    Name string

    Path string

    Owner string

    Group string

    NeedsBackup bool

    Clients []VolumeClient `json:"Clients,omitempty"`
}
```

### Name

Der Name identifiziert das Volume. Dieses Feld ist obligatorisch. Er muss in allen Volumes desselben Dogus eindeutig
sein.

Die Syntax des Namens muss der Dateisystem-Syntax des jeweiligen Host-Betriebssystems entsprechen und sollte aus
folgenden Zeichen bestehen:

- lateinischen Kleinbuchstaben
- Sonderzeichen Unterstrich "_", Minus "-"
- Ziffern 0-9

Der Name darf nicht "_private" lauten, um Konflikte mit dem privaten Schlüssel des Dogus zu vermeiden.

Beispiele:

- tooldata
- werkzeug-daten-0

### Path

Pfad zu dem Verzeichnis oder der Datei, in dem/der das Volume innerhalb des Dogus gemountet werden soll. Dieses Feld
ist obligatorisch.

Der Pfad kann aus mehreren Verzeichnisebenen bestehen, die durch einen Schrägstrich "/" getrennt sind. Der Pfad muss mit
der Dateisystem-Syntax des Container-Betriebssystems übereinstimmen.

Der Pfad darf nicht mit `/private` übereinstimmen, um Konflikte mit dem privaten Schlüssel des Dogus zu vermeiden.

Beispiele:

- /usr/share/yourtool
- /tmp
- /usr/share/license.txt

### owner

Owner enthält die numerische Unix-UID des Benutzers, der Eigentümer dieses Volumes ist. Dieses Feld ist optional.

Aus Sicherheitsgründen wird dringend empfohlen, den Besitzer des Volumes auf einen unprivilegierten Benutzer zu setzen.
Bitte beachten Sie, dass das Container-Image dann so erstellt werden muss, dass der Container-Prozess den Pfad entweder
durch Benutzer- oder Gruppenbesitz besitzen kann.

Die Eigentümer-Syntax darf nur aus Ziffern (0-9) bestehen.

Beispiele:

- "1000" - ein unprivilegierter Benutzer
- "0" - der Root-Benutzer

### Group

Group enthält die numerische Unix-GID der Gruppe, der dieses Volume gehört. Dieses Feld ist optional.

Aus Sicherheitsgründen wird dringend empfohlen, die Gruppe des Volumes auf eine unprivilegierte Gruppe zu setzen. Bitte
beachten Sie, dass das Container-Image dann so erstellt werden muss, dass der Container-Prozess den Pfad entweder durch
Benutzer- oder Gruppenbesitz besitzen kann.

Die Syntax der Gruppe darf nur aus Ziffern (0-9) bestehen.

Beispiele:

- "1000" - eine unprivilegierte Gruppe
- "0" - die Root-Gruppe

### NeedsBackup

NeedsBackup steuert, ob das Cloudogu EcoSystem-Backup das gesamte Volume sichert oder nicht. Dieses Feld ist
optional. Wenn es nicht gesetzt ist, wird ein Wert von `false` angenommen.

### Clients

Clients enthält eine Liste von Client-spezifischen (d. h., der Client, der die dogu.json interpretiert) Konfigurationen
für das Volume. Dieses Feld ist optional.

## type [VolumeClient](<https://github.com/cloudogu/cesapp-lib/blob/main/core/dogu_v2.go#L58-L67>)

VolumeClient fügt zusätzliche Informationen für Clients zur Erstellung von Volumes hinzu.

Beispiel:

```
{
  "Name": "k8s-dogu-operator",
  "Params": {
    "Type": "configmap",
    "Content": {
      "Name": "k8s-ces-menu-json"
    }
  }
}
```

```go
type VolumeClient struct {
    Name string

    Params interface{}
}

```

### Name

Name identifiziert den Client, der für die Verarbeitung dieser Volume-Definition zuständig ist. Dieses Feld ist
obligatorisch.

Beispiele:

- cesapp
- k8s-dogu-operator

### Params

Params enthält generische Daten, die nur vom Client interpretiert werden können. Dieses Feld ist obligatorisch.
